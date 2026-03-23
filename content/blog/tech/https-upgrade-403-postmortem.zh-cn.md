---
title: "记一次 HTTPS 升级引发的 403 排查：从安全中间件到容器隔离机制的深度复盘"
date: 2026-03-17T12:00:00+08:00
draft: false
tags: ["HTTPS", "Nginx", "Go", "Gin", "CORS", "CSRF", "Podman", "Rootless", "Postmortem"]
categories: ["Tech", "Engineering", "Security"]
summary: "一次 HTTPS 协议升级后稳定复现 403 的排障复盘，最终定位为 CSRF 白名单遗漏与 Podman Rootless 镜像隔离叠加问题。"
url: "/zh-cn/blog/tech/https-upgrade-403-postmortem/"
---

近期在推进核心业务线基础架构安全演进时，我们将全局 HTTP 协议升级为 HTTPS。Nginx 网关层完成 SSL 配置并开启 443 强转后，回归测试稳定复现登录接口 `403 Forbidden`。

返回报文如下：

```json
{"code":403,"msg":"forbidden: invalid origin"}
```

系统架构：

- 前端：Vue
- 后端：Go + Gin
- 网关：Nginx 反向代理
- 运行时：Podman（Rootless 模式）

---

## 排查策略：自顶向下，逐层剥离

本次排查遵循“边界清晰、逐层验证”的原则，依次穿透网关层、应用层和容器运行时层。

## 第一层：网关层（Nginx）

第一反应是 Nginx 未正确透传请求头。检查配置后确认 `Origin` 等关键头部已正常透传。

调试过程中发现，只要在网关中强行注入：

```nginx
proxy_set_header Origin "http://业务域名";
```

接口就能“恢复正常”。

但这个方案本质是在伪造请求来源，用 HTTP 值去冒充 HTTPS 请求，直接绕过后端 Origin 校验，等价于削弱 CSRF 防线。该 workaround 被明确废弃，继续向应用层深挖。

## 第二层：应用层（CORS 与 CSRF 的错位）

抓包确认响应头中已经返回：

```http
Access-Control-Allow-Origin: https://业务域名
```

说明 CORS 中间件对 HTTPS 来源已经放行。问题在于：

- CORS 负责“浏览器能否读取跨域响应”
- CSRF 负责“后端是否信任请求来源”

两者并行但独立。

最终在代码中定位到核心遗漏：协议升级时只更新了 CORS 白名单，CSRF 中间件仍保留旧的 `http://` 信任列表。请求通过了 CORS，却被 CSRF 拦截并返回 `invalid origin`。

修复动作：同步补齐 CSRF 中间件中的 `https://` 域名白名单。

## 第三层：运行时层（Podman Rootless 隔离）

代码修复并触发构建后，线上现象仍未消失。进一步通过 `podman inspect` 对比镜像哈希，发现了一个工程化陷阱：

- 生产服务运行在 `deploy` 账号的 Rootless 命名空间
- 部分镜像曾在 Root 命名空间构建
- Rootless 隔离下，`deploy` 无法感知 Root 空间里的新镜像

结果是部署脚本反复重启旧镜像，形成“代码已修复、环境未生效”的假象。

为从机制上杜绝该类问题，我们在生产机落地了强约束：通过 `/usr/local/bin/podman` 与 `/usr/local/bin/podman-compose` 包装脚本，检测到 `uid=0` 直接拒绝执行并退出；`which podman` 指向包装器后，即使 root 误操作也会立即失败，避免再次写入错误命名空间镜像。

---

## 根因总结

这次故障不是单点 Bug，而是两类系统性遗漏叠加：

1. 架构规范缺口：CORS 与 CSRF 白名单分散配置，协议升级时缺乏统一治理入口。
2. 工程闭环缺失：生产镜像构建权限未完全收敛到标准化 CI/CD，导致跨命名空间脏状态。

---

## 过程改进与行动项

### 1）完善安全发版 SOP 与 CheckList

将“CORS 与 CSRF 策略一致性校验”纳入强制发布检查项，新项目上线和协议升级必须过卡点。

### 2）收敛生产环境运维准入

明确应用容器仅允许通过 CI/CD 管道构建与发布，禁止人工跨权限命名空间干预线上服务。

### 3）推进可观测性规范

将中间件加载与路由注册顺序纳入架构规范，确保日志层前置于安全拦截层，避免“被拦截但无日志”的幽灵请求。

---

## 结语

这次 403 排查的价值不在于“把接口调通”，而在于暴露了安全配置治理和交付链路上的真实薄弱点。技术修复解决的是当下，流程与制度收敛解决的是下一次不再重演。
