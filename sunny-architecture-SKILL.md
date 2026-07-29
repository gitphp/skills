---
name: adc-architecture
description: Documents the ADC full-stack microservice architecture (Vue 3 MicroApp, Hyperf 3.1, Nacos, Redis, RabbitMQ, Xxl-Job, MySQL, Nginx/k8s). Use when the user explicitly invokes this skill or asks about ADC project architecture, tech stack, middleware, or deployment.
disable-model-invocation: true
---

# ADC 项目架构

ADC 采用前后端分离的微服务架构，功能均通过接口交互。部分规划中的技术可能尚未投入使用。

详细原文见仓库根目录 `merchant-settle-apply-api.md`。架构图见同文档引用的 `images/construct.png`、`images/node_simple.png`、`images/nodes.png`。

## 架构原则

- 前后端分离，接口交互
- 微服务，各服务可独立技术栈与独立数据库（天然分库）
- 容器化部署，支持水平扩展与 k8s 高可用

## 前端

| 层 | 选型 | 说明 |
|---|---|---|
| 微前端 | MicroApp | 京东团队方案；JS/样式/元素/路由隔离；技术栈无关。官网：https://jd-opensource.github.io/micro-app/ |
| 框架 | Vue 3 | https://v3.cn.vuejs.org/ |
| UI | Element Plus | https://element-plus.org/zh-CN/ |
| 移动端（规划） | uniApp | 一套代码多端（App/Web/小程序等）。https://uniapp.dcloud.net.cn/ |

## 后端

| 层 | 选型 | 说明 |
|---|---|---|
| 语言 | PHP 8.3.0 | 主语言；特定服务后续可用 Java/Go 等 |
| 框架 | Hyperf 3.1.0 | 协程、PSR、DI。https://hyperf.io/ · 文档：https://hyperf.wiki/3.1/#/ |
| 数据库 | MySQL 8.0.3 | 可替代：PolarDB / TDSQL / OceanBase（兼容 MySQL 8.0 协议） |
| 服务/配置中心 | Nacos 2.5.1 | https://nacos.io/zh-cn/index.html |

### 中间件

| 组件 | 版本 | 用途 | 约束 |
|---|---|---|---|
| Redis | ~7.0 | 缓存 | 共用实例（量大用集群）；各微服务用 **key 前缀** 隔离。https://redis.io/ |
| RabbitMQ | 3.13.7 | 消息队列 | https://www.rabbitmq.com/ |
| Xxl-Job | 3.2.0 | 分布式任务调度 | https://github.com/xuxueli/xxl-job |

## 部署

### 环境约束

- **OS**：生产仅 Linux（不支持 Windows/macOS）
- **Web**：仅 Nginx（不支持 Apache）——按 URL 规则转发到不同 Docker 端口，并托管前端静态页

### 容量参考

| 场景 | 配置 |
|---|---|
| 最低 | 4 核 / 16G / 带宽 5M |
| k8s 建议 | 单 pod 2 核 2G |
| 扩容 | 按微服务访问量分机部署 |

### 拓扑

- 单节点：见 `images/node_simple.png`
- 多节点：见 `images/nodes.png`

## Agent 使用约定

回答架构相关问题时：

1. 以本 skill 的选型与版本为准；不确定是否已落地时，标明「规划中可能未投入使用」。
2. 为本仓库（`adc-merchant`）写代码时，后端默认 **PHP 8.3 + Hyperf 3.1**；Redis key / 队列 / 任务 / 日志使用服务前缀隔离（见 `CLAUDE.md` 中 `APP_NAME=merchant`）。
3. 不要假设 Apache、非 Linux 生产环境，或跨服务共享无前缀的 Redis key。
4. 需要更完整原文或配图说明时，读取 `merchant-settle-apply-api.md`。
