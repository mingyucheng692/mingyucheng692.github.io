---
title: "Rootless Podman + Systemd 托管失效复盘：一次恢复链路排查与修复记录"
date: 2026-04-03T20:00:00+08:00
draft: false
tags: ["Podman", "Systemd", "Rootless", "Container", "Nginx", "Redis", "Go", "Postmortem", "SRE"]
categories: ["Tech", "Engineering", "Infrastructure"]
summary: "一次 Rootless Podman + Systemd 托管失效的复盘：从主 Unit 损坏、启动状态判定偏差到部署脚本显式移交 Systemd 接管，梳理已确认的问题、修复动作、验证方式与未覆盖风险。"
url: "/zh-cn/blog/tech/rootless-podman-systemd-watchdog-postmortem/"
---

这篇复盘记录一次容器恢复链路失效后的排查与修复过程。在一套由 Rootless Podman 托管的业务服务中，数据库容器异常退出后没有按预期被自动拉起，随后又陆续暴露出启动超时、容器在线但服务 `inactive`、以及频率限制锁死等问题。麻烦之处不在单次退出，而在于同类故障在不同时间可能表现为自动恢复、反复重启后锁死，或容器仍在运行但托管状态已经脱节。

> 说明：本文中的项目名、服务名、路径、域名、用户名、主机标识、时间点、PID、命令输出与日志片段均已做脱敏、抽象或重写，仅保留与排障链路相关的技术信息，不对应任何真实生产标识。

- 运行时：Rootless Podman 4.9.x
- 进程托管：Systemd User Units
- 业务组件：TimescaleDB、Redis、Go Backend、Nginx Frontend
- 目标能力：容器异常退出后由 Systemd 自动恢复

## 事故摘要

- 触发背景：数据库容器异常退出后，现场最先表现为数据库对应 Unit `inactive`，其余服务对应 Unit 仍为 `active`
- 过程表现：进入修复后，数据库服务先恢复为 `active`，但其余服务一度未被 Systemd 正确接管，出现启动超时、容器状态与 Unit 状态不一致，以及部分服务因频率限制被锁死
- 已确认的致因因素：主 Unit 曾被脚本通过 `sed -i` 直接改写，并在 PowerShell 转义截断后被写坏；Compose 与 Systemd 并存导致控制权分离；自动生成的 Unit 仍需后处理；恢复脚本对依赖清理和失败恢复覆盖不足
- 修复方向：停止用 `sed -i` 直接改主 Unit，改为重新生成 `.service` 后由 Python 脚本修补，并显式把运行期控制权移交给 Systemd，再补齐状态轮询与最终健康检查
- 当前结果：在当前环境中，恢复路径已能按固定顺序执行，失败时也有明确的诊断入口；但 readiness 判定和监控采集仍待完善

## 现象与影响

最初现场并不是“所有服务一起掉线”，而是数据库对应 Unit 先变成 `inactive`，其余服务仍保持 `active`。真正把问题放大的，是进入修复和接管阶段后又暴露出另一组现象：
- 有些服务在 `systemctl --user status` 中显示超时重启
- 有些容器已经 `Up`，但对应的 Systemd 服务却仍是 `inactive`
- 有些服务因为频繁失败被 Systemd 直接锁死，不再继续重试

这说明问题不是某一个容器单点异常，而是“容器实际状态、Systemd 托管状态、部署脚本默认假设”三者之间已经出现偏差。本文聚焦“恢复链路为什么失效、脚本后来怎样调整”，不展开数据库掉线本身的业务触发原因。

---

## 排查思路：先确认谁真正拥有进程控制权

这次排查没有从单个报错入手，而是先回答一个基础问题：容器退出后，究竟是谁负责发现它、判断它失败并把它重新拉起来。沿着这条线索，最终定位到四类相互叠加的问题：

## 问题一：主 Unit 被直接修改，最终损坏


最先暴露的问题，是数据库对应的主 Unit 被破坏成了空文件：

```text
systemd[<pid>]: /home/<deploy-user>/.config/systemd/user/container-<db-service>.service:1: Missing '='.
```

进一步检查文件本体时，可以直接看到它已经被截断为 `0` 字节：

```bash
ls -l /home/<deploy-user>/.config/systemd/user/container-<db-service>.service
# -rw-rw-r-- 1 <deploy-user> <deploy-user> 0 <timestamp> container-<db-service>.service

file /home/<deploy-user>/.config/systemd/user/container-<db-service>.service
# container-<db-service>.service: empty
```

这次对触发链已经能说得更具体：旧版脚本确实会直接用 `sed -i` 修改主 Unit；而在通过 PowerShell 下发这类命令时，转义处理存在偏差，导致目标文件没有被正确写回，最终出现主 Unit 被截断为 `0` 字节、Systemd 无法继续解析的结果。这里的问题不是单纯的“脚本里用了 `sed -i`”，而是“把主 Unit 当作可原地改写对象，再叠加跨 Shell 转义差异”，共同放大了损坏风险。

旧版脚本中的关键路径大致如下：

```bash
# 旧版：直接修改主 Unit
sed -i 's/Restart=always/Restart=no/g' "$service"
```

问题已经足够明确：主服务文件同时承担“运行定义”和“运维开关”两种角色，本身就是高风险设计。一旦主 Unit 损坏，解析、启动和重启策略会一起失效。

后来的调整比较直接：

- 主服务文件只保留生成产物属性，不再由 `sed -i` 直接修改
- 重启策略等开关下沉到 `.service.d/watchdog.conf`
- 生成后的 `.service` 兼容性修补交给 Python 处理，Drop-in 则由脚本覆盖写入

当前脚本里，Drop-in 的写入方式类似下面这样：

```bash
cat > "$DROP_IN_DIR/watchdog.conf" << EOF
[Service]
Restart=always
RestartSec=10s
ExecStartPre=-/usr/bin/podman rm -f <db-service>
EOF
```

这样做至少把边界拆清楚了：主 Unit 视为可重建产物，运行期开关统一由 Drop-in 管理。就当前脚本和后续现场结果看，也没有再复现“因原地修改主 Unit 而导致服务文件损坏”的问题。

## 问题二：Systemd 认定失败，但容器实际已经启动

第二类问题更隐蔽。故障阶段，Systemd 会把某些服务标记为启动超时：

```text
Active: activating (auto-restart) (Result: timeout)
Main PID: <pid> (code=exited, status=0/SUCCESS)
```

但同时 `podman ps` 又能看到容器确实已经起来了，本质上是进程契约不一致。

`podman generate systemd` 生成的 Unit 默认使用 `Type=notify`。这意味着 Systemd 不会仅凭进程存在就判定服务启动成功，而是要求运行中的容器向宿主发送 `sd_notify` 就绪信号。

故障阶段未完成修补时，生成产物中的关键字段如下：

```ini
Type=notify
NotifyAccess=all
```

恢复后重新采集当前环境时，重新生成的产物仍然保持这个默认值；而成功上线后的生效 Unit 已经被修补为 `Type=simple`：

```ini
# 故障期对应的默认生成结果
Type=notify

# 修复上线后的生效 Unit
Type=simple
```

这次现场能确认的现象是：在服务尚未完成稳定托管、生成产物仍保持 `Type=notify` 的阶段，容器已经 `Up`，但对应 Unit 仍会在超时后被 Systemd 判为失败并重启。后来的修复做法，是把生成产物中的 `Type=notify` 改成 `Type=simple`，把状态认定切回到“由 Systemd 直接追踪前台进程”。

这里需要说明边界：这并不等于已经证明 Rootless 场景下 `notify` 一定不可用，而是当前这套环境里，`notify` 没有表现出稳定、可依赖的就绪语义。对这类长驻前台进程来说，`Type=simple` 是更保守、也更容易排障的选择。

## 问题三：一部分容器并不在 Systemd 的控制内

更准确地说，这个危险状态出现在修复过程里，而不是最初故障现场：数据库对应 Unit 已恢复为 `active`，但其余服务对应 Unit 仍显示 `inactive (dead)`。

当时现场的状态大致如下：

```bash
podman ps
# ... 部分业务容器可能已经重新起来

systemctl --user list-units 'container-*.service'
# ... container-<db-service>.service 为 active
# ... 其余服务对应 Unit 为 inactive (dead)
```

原因是这些容器并不是由 Systemd 拉起，而是早先通过 `podman-compose up -d` 直接启动的。对于这类非托管容器，即使容器进程已经存在，Systemd 也没有进程控制权，自然谈不上稳定接管、失败感知和自动恢复。

这件事暴露出一个容易被忽略的前提：托管能力不仅取决于 Unit 文件是否存在，更取决于进程是不是由 Systemd 持有。如果运行期仍然由 Compose 持有进程控制权，那么配置虽然在，恢复能力并没有真正建立起来。

这里也需要说明实现细节：当前部署脚本并不是完全不用 Compose。它仍会先用 `podman-compose` 完成构建和冷启动，再调用 `watchdog-enable.sh` 生成并修补 Unit，随后显式停止这些 compose 容器，最后由 `systemctl --user start` 按顺序接管。问题不在于“脚本里出现过 `podman-compose`”，而在于运行期是否仍由 Compose 持有进程控制权。后来的调整，就是把这一步接管固定下来：

- 停止现有非托管容器，重新由 `systemctl --user start` 接管
- 在接管前后做状态检查，避免再次留下“容器在跑但 Systemd 不知情”的状态

## 问题四：自动生成与恢复脚本还需要补齐细节

### 1）自动生成的 Unit 仍然需要后处理

重新使用 `podman generate systemd --new` 后，至少暴露出三类问题：

- 通过 `podman-compose` 等工具初始化的参数也会带来干扰。例如，`compose` 生成的容器可能带有 `-d` 参数，`podman generate systemd --new` 会将其原样提取。这会导致 Systemd 执行 `podman run -d` 时进程立即退出，无法追踪容器主进程
- 另一个典型案例是 Redis 的配置。若原本通过空参数禁用危险命令（如 `rename-command ""`），自动生成工具会静默剥离空字符串，导致最终启动命令变为 `redis-server --rename-command FLUSHALL`，引发启动参数报错
- `Type=notify` 默认值仍会再次引入 Rootless 场景的超时问题

Redis 的问题在原始容器参数和生成结果之间可以直接对照出来：

```yaml
# 原始容器参数
command:
  - --rename-command
  - "FLUSHALL"
  - ""
```

```text
# 未修补的 generate 结果
redis-server --rename-command FLUSHALL
```

恢复后重新执行 `podman generate systemd --new --name <redis-service>`，仍能看到生成结果里空字符串被吞掉；而当前生效 Unit 中已经补回：

```ini
# 恢复后重新生成的结果
--rename-command FLUSHALL

# 修复上线后的生效 Unit
--rename-command FLUSHALL ""
```

`-d` 也是同类问题。恢复后对照生成产物与生效 Unit，可以直接看到默认生成结果仍包含 `-d`，而接管用的 Unit 已经将其移除：

```ini
# 恢复后重新生成的结果
ExecStart=/usr/bin/podman run \
  ...
  -d \
  --sdnotify=conmon \

# 修复上线后的生效 Unit
ExecStart=/usr/bin/podman run \
  ...
  --sdnotify=conmon \
```

`watchdog-enable.sh` 当前的做法，是先重新执行 `podman generate systemd --new --name ...`，再用 Python 脚本修补生成产物。实际落地的处理包括三件事：

- 把 `Type=notify` 改成 `Type=simple`
- 移除 `ExecStart` 中可能残留的 `-d`
- 对 Redis 的 `rename-command ""` 做回填，避免空参数在生成阶段丢失

这里使用 Python 的原因也很具体：它更适合处理由 `\` 续行的多行命令，能让生成产物的修补行为更可控。

### 2）依赖关系会影响失败容器的清理

数据库服务恢复前，需要先执行 `podman rm -f` 清理旧容器；但由于上游服务仍持有依赖引用，删除动作返回 `exit 125`。这说明单容器恢复不是孤立操作，依赖图会反向影响清理过程。

直接报错类似这样：

```text
Process: ExecStartPre=/usr/bin/podman rm -f <db-service> (code=exited, status=125)
```

当前脚本的处理方式，是把依赖清理逻辑写进 Drop-in：

- 数据库服务在 `ExecStartPre` 中会先停掉依赖它的上层服务，再执行 `podman rm -f <db-service>`
- 其他几个服务只清理各自容器

这不是一套通用模板，只是针对这次这组服务依赖关系做的脚本化处理。

### 3）恢复链路不能只依赖固定 `sleep`

早期脚本里确实存在靠固定 `sleep` 等待依赖的做法。

当前部署脚本在 Compose 冷启动阶段仍保留了少量固定等待；但在控制权移交给 Systemd 之后，已经改成按依赖顺序逐个 `systemctl --user start`，并配合 `wait_for_service()` 轮询 `is-active` / `is-failed`，在失败或超时时输出对应 `journalctl`。这比盲等更容易定位失败点。

因此最终把启动顺序改成显式编排：

- `<db-service>` -> `<redis-service>` -> `<backend-service>` -> `<frontend-service>`

另外，脚本在全部服务接管完成后还会追加一次应用级健康检查，用来确认“服务 active”已经进一步接近“对外可用”。这一步是对 `Type=simple` 取舍的补充，不是每个服务启动时都做的深度 readiness 检查。

交接给 Systemd 后，等待逻辑大致如下：

```bash
systemctl --user start container-<db-service>.service
for i in $(seq 1 60); do
  systemctl --user is-active --quiet container-<db-service>.service && break
  systemctl --user is-failed --quiet container-<db-service>.service && exit 1
  sleep 1
done
```

### 4）Systemd 的频率限制会把服务锁死

在依赖尚未就绪时（例如 Nginx 在启动时会强行解析 upstream 的 `<backend-service>` 域名），Frontend 会反复失败重启，随后触发 `StartLimitBurst` 限制（例如在 300 秒内重启超过 10 次）。此时即使后续依赖恢复正常，Systemd 也不会再自动拉起该服务，必须显式执行：

```text
container-<frontend-service>.service: Start request repeated too quickly.
container-<frontend-service>.service: Failed with result 'exit-code'.
```

这类失败在 Nginx 侧通常还能看到更直接的应用级证据：

```text
[emerg] 1#1: host not found in upstream "<backend-service>" in default.conf:31
```

```bash
systemctl --user reset-failed container-<frontend-service>.service
```

`deploy-all.sh` 当前也已经在拉起 Frontend 之前补了一次 `reset-failed`，用来清掉此前可能残留的锁死状态。

---

## 本次故障中已确认的致因因素

这次故障表面上看是数据库异常退出后，现场状态与接管状态不断分裂；回头看，本质上是几类问题叠加：

1. 主 Unit 直接被脚本改写，服务定义本身有被破坏的风险。
2. `podman-compose` 直接启动和 Systemd 托管并存，容器状态与托管状态会分离。
3. `podman generate systemd` 的产物不能直接拿来用，还需要补齐 `Type`、`-d` 和参数修补。
4. 恢复脚本对依赖清理、频控恢复和最终健康检查最初覆盖得不够完整。

这不是某一个开关写错导致的单点问题，而是恢复链路从定义、接管到执行都存在断点。对应的修复思路也不是只处理某一条报错，而是把运行期控制权、生成后的修补动作和失败后的诊断入口一起补齐。

---

## 已落地的修复动作

结合 `watchdog-enable.sh` 和 `deploy-all.sh`，这次实际落地的动作大致如下：

- 部署脚本先校验执行用户、`HOME`、`XDG_RUNTIME_DIR` 和 `DBUS_SESSION_BUS_ADDRESS`，减少 Rootless User Units 因运行环境不一致而失效的概率
- 部署阶段仍使用 `podman-compose` 完成构建和冷启动，但在交接阶段会显式停掉 compose 容器，再由 Systemd 顺序接管
- `watchdog-enable.sh` 会先备份旧 Unit，再重新生成 `.service`
- 生成后的 `.service` 改由 Python 脚本做三项修补：改 `Type=simple`、删 `-d`、补 Redis 空参数，同时避开继续用 `sed -i` 直接改主 Unit 时的 PowerShell 转义截断问题
- Drop-in 中统一写入 `Restart=always`、`RestartSec`、`StartLimit*` 和各服务对应的 `ExecStartPre`
- 交接给 Systemd 后，脚本按 `<db-service> -> <redis-service> -> <backend-service> -> <frontend-service>` 顺序启动，并通过 `wait_for_service()` 检查 `is-active` / `is-failed`；若失败或超时，直接输出对应 `journalctl`
- 在拉起 Frontend 前补一次 `reset-failed`，避免残留的频率限制状态阻断恢复
- 全部服务接管完成后，再追加一次应用级健康检查，确认系统已经从“服务 active”进一步接近“对外可用”

这套做法仍然是单机、Rootless、User Units 这组约束下的工程化实现，解决的是“谁接管进程、怎样恢复、失败后怎样排查”这些具体问题，不等于更广义的高可用方案。

---

## 修复后如何验证

- 接管前后对照 `podman ps` 与 `systemctl --user list-units`，确认不再出现“容器在跑但对应 Unit 为 `inactive`”的状态
- 通过 `wait_for_service()` 轮询 `is-active` / `is-failed`，让服务在启动阶段就暴露失败点，而不是交给固定 `sleep`
- 若服务失败或超时，立即输出对应 `journalctl`，把排查入口固定到具体 Unit
- 全部服务接管完成后追加一次应用级 `/health` 检查，用来验证外部可用性不再只依赖 `Type=simple` 的进程在线语义

恢复后再次核对时，四个容器与四个 User Unit 已经重新对齐：

```text
podman ps
<db-service> / <redis-service> / <backend-service> / <frontend-service> 均为 Up

systemctl --user list-units 'container-*.service'
对应四个 Unit 均为 active (running)
```

这些验证主要覆盖“是否由 Systemd 持有进程”“失败时是否能及时暴露”“接管完成后是否对外可用”，还没有扩展到更细的 readiness 信号和监控采集。

---

## 这次排障后明确的约束

- 主 Unit 只作为生成产物保留，运行期开关统一放到 Drop-in。这样做不是为了形式上的规范，而是为了减少主文件损坏时同时打断解析、启动和重启策略的风险。
- `podman generate systemd` 的产物不能直接视为可运行结果，生成、修补、重载应被看作同一个步骤。就这次脚本而言，`Type=notify`、`-d` 和空参数问题都属于生成后必须立即处理的兼容性项。
- 部署期和运行期的控制面需要明确分开。部署阶段仍可使用 Compose 做构建和冷启动，但进入运行期后，进程控制权必须显式交回 Systemd，否则“已经写了重启配置”和“恢复能力真的生效”会继续混在一起。
- 恢复链路不能只写 `Restart=always` 就结束，还需要覆盖依赖清理、频控恢复、状态轮询和最终健康检查。这样做的目的不是堆功能，而是让故障后的排查和恢复路径可重复。

---

## 后续演进方向

这次修复解决了恢复链路中的几个确定问题，但仍有几类风险没有被完全覆盖，长期维护上还需要继续推进：

- `Type=simple` 配合 `is-active` / `is-failed` 和最终 `/health` 检查，解决了“进程在线”和“最终可用”的判断问题，但还没有覆盖更细粒度的 readiness 信号；如果服务启动后短时间内可见为 `active`、随后才暴露内部初始化失败，当前脚本仍可能滞后发现。
- 当前方案仍依赖 `podman generate systemd` 加脚本修补；只要生成产物的格式、参数展开方式或 Podman 版本行为发生变化，现有补丁逻辑就可能失效，因此是否迁移到 Quadlet 或其他声明式方式，仍需要单独评估。
- 目前状态验证仍偏脚本内诊断；失败次数、退出码、容器状态、频率限制命中情况还没有接入采集、上报和告警。这意味着脚本外仍缺少持续观测能力，类似问题更可能在故障发生后才被动暴露。

## 这次排障带来的直接改进

- 进程由谁启动、谁负责重启，边界比之前清楚了
- 自动生成的 Unit 是否可直接使用，有了明确判断和后处理步骤
- 接管失败时该看哪一层日志、在哪一步退出，脚本里有了稳定入口
- 服务 active 与对外可用之间的差异，被补上了一层应用级验证

这些改动并不意味着这套方案已经成为更广义的高可用设计，但至少把“重启配置存在却无法稳定恢复”的状态改成了一条更容易验证、也更容易重复执行的恢复链路。

## 结语

这次排查最后留下的结论很直接：配置上写了 `Restart=always`，并不代表恢复链路就已经完整。主 Unit 是否稳定、容器是不是由 Systemd 启动、生成产物有没有修补、依赖和频率限制有没有被脚本覆盖，这些细节都会影响最终结果。

至少在这次这套单机 Rootless Podman 场景里，把这些断点补进脚本之后，进程控制权、排查入口和故障处置路径都比之前清楚了很多。这未必意味着方案已经“完美”，但已经把一次原本依赖经验判断的故障恢复，整理成了一条更可解释、也更容易验证的工程链路。

---

## 附录：关于 Linger 机制的补充说明

Rootless Podman 如果配合 `systemd --user` 托管服务，通常需要先为目标用户开启 `linger`。否则一旦该用户退出登录，用户级 Systemd 实例可能被回收，相关服务也无法在“无人登录”的情况下持续运行。本次事故发生时环境已经满足 `Linger=yes`，因此 `linger` 不是本次故障根因，这里仅作为前置检查项补充。

```bash
# 检查状态
loginctl show-user <deploy-user> --property=Linger

# 若未启用，由 sudo 用户执行：
sudo loginctl enable-linger <deploy-user>
```
