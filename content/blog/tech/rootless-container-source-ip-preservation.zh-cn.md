---
title: "Rootless 容器入向源 IP 保留：架构权衡与 Hybrid Host-Bridge 拓扑落地记录"
date: 2026-07-11T20:00:00+08:00
draft: false
tags: ["Podman", "Rootless", "Nginx", "EMQX", "Networking", "NAT", "Source-IP", "Container", "Postmortem"]
categories: ["backend-infra"]
summary: "记录一次 Rootless Podman 下入向源 IP 被 NAT 改写问题的架构分析与方案落地。通过 Hybrid Host-Bridge 拓扑在边缘服务保留客户端真实 IP，同时维持内部服务的网络隔离，并给出方案选型权衡、实现细节与验证方法论。"
url: "/zh-cn/blog/tech/rootless-container-source-ip-preservation/"
---

入向流量源 IP 被 Rootless 端口转发器的 NAT 改写为容器网桥网关 IP,导致审计、限流与诊断链路失效。这是 Rootless 用户命名空间设计的结构性约束,无法通过配置规避。最终采用 Hybrid Host-Bridge 拓扑:边缘服务(Nginx、EMQX)切换至 Host Network 保留客户端真实 IP,内部服务(后端、DB、Redis)保留 Bridge Network 维持隔离,loopback 作为跨命名空间通信通道。该方案在单机部署、边缘服务数量有限、无 NetworkPolicy 需求的约束下成立。

> 说明:本文中的项目名、服务名、域名、IP、账号、路径、日志片段与命令输出均已脱敏、抽象或重写,仅保留与架构决策相关的技术信息,不对应任何真实生产标识。

## 根因:Rootless NAT 机制

系统由四类服务构成:前端反向代理(Nginx)、MQTT Broker(EMQX)、业务后端(Go)、数据层(TimescaleDB + Redis)。容器运行时为 Rootless Podman,部署在云主机上,外部客户端包含浏览器与 IoT 设备两类。

问题表现为:应用访问日志与 Broker 日志中客户端源 IP 统一被记录为容器网桥地址(如 `10.89.0.7`),基于源 IP 的速率限制因单一网桥 IP 命中限流而导致全部客户端被阻断,审计与诊断链路缺乏可信的客户端地理来源信息。

根因在于 Rootless 容器的网络隔离依赖用户命名空间(User Namespace),非特权用户无法修改宿主机路由表,也无法绑定特权端口(< 1024)。为绕开这一约束,Podman 通过用户态端口转发器(`rootlessport` 或 `slirp4netns`)将宿主机端口映射进容器网桥命名空间,流程如下:

1. 端口转发器在宿主机侧监听目标端口
2. 外部连接到达后,转发器在容器网桥命名空间内发起新连接
3. 新连接的源地址被改写为网桥网关 IP(如 `10.89.0.7`)
4. 容器内的 `$remote_addr` 与后续转发的 `X-Forwarded-For` / `X-Real-IP` 均基于改写后的地址

```
[External Client] ---> (Host TCP Port 443/1883)
                             |
                   [Rootless Port Forwarder] (NAT Layer)
                             |
                             v  (Source IP rewritten to Gateway IP)
        [Edge Container (Bridge Namespace)]
                             |
                             v  (Propagates rewritten IP via HTTP Headers)
             [Backend Container (Bridge Network)]
```

NAT 发生在容器命名空间边界之前,边缘容器无法从连接本身获取原始客户端 IP。只要流量经端口转发器进入容器网桥,源 IP 改写就不可避免,这是方案选型的前置约束。

---

## 方案选型与权衡

在确认根因后,候选方案有三类。下表从源 IP 保留、网络隔离、实现复杂度、维护成本四个维度对比:

| 方案 | 源 IP 保留 | 网络隔离 | 实现复杂度 | 维护成本 |
|---|---|---|---|---|
| A. Proxy Protocol | 需转发器与上游均支持 | 保留 | 中 | 协议兼容性依赖强 |
| B. 全量 Host Network | 保留 | 丧失 | 低 | 端口冲突风险随服务数增长 |
| C. Hybrid Host-Bridge | 保留(边缘) | 部分(内部) | 中 | 边界清晰,耦合度低 |

### 方案 A:Proxy Protocol

通过在转发器与上游之间引入 Proxy Protocol 传递原始 IP。该方案的前提是 Rootless 端口转发器需支持输出 PROXY protocol 头,且所有上游(Nginx、EMQX)需开启对应接收配置。当前 `rootlessport` 对 PROXY protocol 输出的支持有限,且 EMQX 的 MQTT 监听器与 HTTP 监听器的 Proxy Protocol 配置路径不同,跨协议维护成本较高。

### 方案 B:全量 Host Network

所有容器切换至 `network_mode: "host"`。源 IP 保留问题消除,但数据层(TimescaleDB、Redis)端口直接暴露在宿主机接口上,丧失网络级隔离。后续若扩展多实例或引入网络策略,端口冲突与隔离缺失会增加维护成本。

### 方案 C:Hybrid Host-Bridge(最终采用)

仅将需要接收外部流量的边缘服务(Nginx、EMQX)切换至 Host Network,内部服务(后端、数据库、缓存)保留在 Bridge Network。边缘服务通过 loopback 与后端通信,后端仍维持网络隔离。

该方案以边缘服务的网络隔离换取源 IP 保留能力,同时通过 loopback 边界限制内部服务的暴露面。在当前单机部署、边缘服务数量有限的约束下,该权衡可接受。

---

## 落地实现:Hybrid Host-Bridge 拓扑

### 网络拓扑

```
                              HOST NETWORK NAMESPACE
+---------------------------------------------------------------------------------+
|                                                                                 |
|  [External Client] ---> (Port 443) ---> [Nginx (Host Mode)]                     |
|                                                |                                |
|  [IoT Gateway]     ---> (Port 1883) ---> [EMQX (Host Mode)]                     |
|                                                |                                |
|                                                v (Localhost Tunnel)             |
|                                          [127.0.0.1:<backend-port>]             |
|                                                |                                |
+------------------------------------------------|--------------------------------+
                                                 | (Port Publish, loopback only)
                                                 v
                               +----------------------------------+
                               |     CONTAINER BRIDGE NETWORK     |
                               |           (<bridge-net>)         |
                               |                                  |
                               |        [<backend-service>]       |
                               |          /           \           |
                               |         /             \          |
                               |        v               v         |
                               |   [<db-service>] [<redis-service>]|
                               |   (Port 5432)     (Port 6379)    |
                               +----------------------------------+
```

### 关键实现点

**1. Host Kernel 端口绑定阈值检查**

Nginx 需在 Host Network 下监听 80/443,但 Rootless 用户默认无法绑定 < 1024 端口。该阈值是全局内核参数,非容器级配置。若前期 HTTPS 配置阶段已下放该阈值,此处仅作确认检查;若未下放,需在单机部署场景下评估全局下放的可接受性后执行。

```bash
# 检查当前阈值（HTTPS 配置阶段通常已下放至 80）
sysctl net.ipv4.ip_unprivileged_port_start

# 若输出非 80,则下放（Nginx 需要 80 做 HTTP→HTTPS 重定向）
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee -a /etc/sysctl.conf
```

多租户环境下需评估是否引入更细粒度的端口权限控制(见后续演进)。

**2. Compose 配置调整**

边缘服务移除 `ports` 与 `networks` 字段后,容器直接共享宿主机网络命名空间,绕开端口转发器;后端服务仅发布到 loopback,避免在 Host Network 切换后暴露至宿主机接口。

```yaml
services:
  <frontend-service>:
    network_mode: "host"
    # ports 与 networks 字段移除

  <emqx-service>:
    network_mode: "host"
    environment:
      - EMQX_AUTHORIZATION__SOURCES__1__URL=http://127.0.0.1:<backend-port>/api/v1/iot/auth

  <backend-service>:
    ports:
      - "127.0.0.1:<backend-port>:<backend-port>"  # 仅 loopback
    environment:
      - <EMQX_BASE_URL>=http://host.containers.internal:<dashboard-port>
```

`host.containers.internal` 是 Podman 提供的特殊 DNS 名称,容器内可通过该名称访问宿主机命名空间的服务,替代原 Bridge Network 内的容器间直连。该 DNS 仅用于容器→宿主机方向;反向(宿主机→容器)仍需通过端口发布。

**3. 反向代理与 Broker 配置收敛**

边缘服务上游地址统一指向 loopback,确保跨命名空间通信仅经 loopback 边界。`$remote_addr` 在 Host Network 模式下即为客户端真实 IP,无需依赖上游转发头。

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:<backend-port>;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

```hocon
authentication = [
  { backend = "http", url = "http://127.0.0.1:<backend-port>/api/v1/iot/auth" }
]
bridges.webhook.<ingest-bridge> {
  url = "http://127.0.0.1:<backend-port>/api/v1/iot/ingest"
}
```

**4. 重新部署与 Supervisor 层状态同步**

配置变更后不能直接 `podman-compose up` 拉起容器。原部署通过 systemd unit 管理容器生命周期,而 unit 文件是基于原 Bridge Network 模式生成的。网络模式从 Bridge 切换至 Host 后,systemd unit 需由部署脚本重新生成以捕获新的网络属性,随后将控制权移交 systemd。若跳过此环节,旧 unit 会以原 Bridge 模式拉起容器,导致配置与运行时不一致,且开机自启可能失效。

```bash
# 通过原部署脚本执行,内部重新生成 systemd unit 并移交管理
./<deploy-script>  # CI: 部署流水线应校验旧 unit 已停止后再生成新 unit,避免新旧 unit 并存导致双实例
```

---

## 验证策略

验证需覆盖全链路,分三层完成。三层验证的逻辑关系:Nginx 日志确认边缘层已获得真实 IP;后端日志确认 IP 通过 HTTP 头正确传播;EMQX 日志确认 Broker 层 AuthN/Webhook 链路已收敛至 loopback。三者缺一不可——仅验证 Nginx 无法确认后端是否正确解析头,仅验证后端无法确认 Broker 链路是否已调整。

### 1. Nginx 日志验证

```bash
podman logs <frontend-service>
```

预期:日志条目起始 IP 为客户端公网 IP,而非网桥网关地址。

```
<client-public-ip> - - [<timestamp>] "GET /api/v1/projects HTTP/1.1" 200 ...
```

### 2. 后端结构化日志验证

```bash
podman logs <backend-service>
```

预期:JSON 日志中 `"ip"` 字段与 Nginx 日志中的客户端 IP 一致。

```json
{"level":"INFO","ts":"<timestamp>","caller":"middleware/logger.go:<line>","msg":"Inbound Request","ip":"<client-public-ip>","status":200,"method":"GET","path":"/api/v1/<resource>"}
```

### 3. EMQX 集成验证

```bash
podman logs <emqx-service>
```

预期:监听器绑定至 `0.0.0.0`,AuthN 与 Webhook 请求路由至 `127.0.0.1`。

```
EMQX_AUTHORIZATION__SOURCES__1__URL [authorization.sources.1.url]: http://127.0.0.1:<backend-port>/api/v1/iot/auth
Listener tcp:default on 0.0.0.0:1883 started.
Listener http:dashboard on :<dashboard-port> started.
```

---

## 工程约束与后续演进

### 当前方案的有效边界

Hybrid Host-Bridge 拓扑在以下约束下成立:

- **单机部署**:Host Network 的端口冲突风险在多机/集群场景下放大
- **边缘服务数量有限**:每增加一个 Host Network 服务,宿主机端口管理复杂度线性增长
- **无网络策略需求**:Rootless Podman 的 Host Network 不支持 NetworkPolicy,隔离依赖 loopback 边界

### 已识别的后续演进方向

**1. 从 Host Network 迁移至 Pasta**

pasta 模式在 Podman 4.x 引入作为 slirp4netns 的替代,在 Podman 5.0 中已成为 rootless 默认网络栈(配合 netavark + nftables)。相对 slirp4netns 的关键改进是不再对入向连接做 SNAT,源 IP 保留因此在 Bridge 模式下也具备可行性。当运行时版本升级时,可评估是否回退边缘服务至 Bridge Network,恢复网络隔离。

**2. 端口绑定权限的细粒度控制**

当前通过全局 `sysctl` 下放端口阈值。在引入更多服务时,可评估基于 systemd 的 `AmbientCapabilities=CAP_NET_BIND_SERVICE` 或 RootlessKit 的端口白名单机制,避免全局阈值下放。

**3. 监控与告警覆盖**

当前验证依赖手动日志检查。源 IP 保留的失效在运行期不会产生明显报错,需通过监控项(如日志中网桥 IP 出现频率、速率限制命中率异常)建立被动发现能力。

**4. 多实例与横向扩展**

当前单机拓扑下,Host Network 的端口绑定是独占的。若后续边缘服务需要多实例或灰度发布,需引入上游负载均衡层,届时源 IP 保留问题会转移至 LB 层,需在 LB 层重新评估 Proxy Protocol 或保留客户端 IP 的方案。

---

## 沉淀的工程约束

- **网络隔离与源 IP 保留存在固有张力**:Host Network 保留 IP 但丧失隔离,Bridge Network 保留隔离但改写 IP。方案选型的核心工作是在两者之间划定边界。
- **Hybrid 拓扑的边界应沿数据流向划分**:接收外部流量的边缘服务用 Host Network,服务间通信的内部服务用 Bridge Network,loopback 作为跨命名空间通信通道。
- **验证需覆盖全链路**:单点日志不足以确认端到端 IP 传播正确性,需从边缘到后端逐层核对。
- **网络模式变更需同步 supervisor 层状态**:容器网络模式切换后,supervisor(如 systemd)的 unit 文件需重新生成以捕获新属性,否则会出现配置与运行时不一致、开机自启失效等问题。

本文沉淀的约束可作为后续运行时升级或拓扑变更的参考基线。
