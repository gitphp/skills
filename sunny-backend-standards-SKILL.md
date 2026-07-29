---
name: sunny-backend-dev
description: Applies sunny Hyperf backend development standards for REST APIs, Request/Service/Resource layers, DB migrations, MQ naming, git commits, and error codes. Use when implementing or reviewing APIs, controllers, requests, services, resources, migrations, or MQ in sunny-merchant / sunny microservices, or when the user mentions sunny 后端开发规范.
disable-model-invocation: true
---

# sunny 后端开发规范

完整原文：仓库根目录 [`merchant-settle-apply-api.md`](../../../merchant-settle-apply-api.md)。示例代码可参考 `sunny-user` 仓库。

本仓库身份：`APP_NAME=merchant`，HTTP `9622`，JSON-RPC `9722`，库 `sunny_merchant`。

## 何时使用

写/改 Controller、Request、Service、Resource、Migration、MQ、错误码，或做代码审查时，按本 skill 执行。所有 **强制规范** 必须遵守。

## 技术栈

Hyperf 微服务 · JSON-RPC · MySQL · RabbitMQ · Nacos · ELK · 公共包 `vendor/tgkw-sunny/helper`

- 业务日志统一用 `LogHelper`（`vendor/tgkw-sunny/helper/src/Helper/Log/LogHelper.php`）
- 先查 helper 包，避免重复造轮子
- 更新：`composer update tgkw-sunny/helper --ignore-platform-reqs`

## 强制规范（必须遵守）

1. **禁止编辑器保存时自动格式化**；提交前统一格式化：
   - 有 PHP：`composer cs-fix` 或 `vendor/bin/php-cs-fixer fix`
   - 容器内：`sh cs-fix.sh`
2. **禁止用数据库工具直接改表**；一律用 Migration
3. **路由必须用注解**，不用 `config/routes.php`
4. **必须用 Request 验证**（继承 `Tgkwsunny\Request\BaseRequest` + `@Scene`）
5. **对外 API 必须经 Resource 包装**，用 `ApiResponseHelper` 返回
6. **错误码枚举化**，禁止魔法数字
7. **URL 版本化**：`/v1/`、`/v2/`

## 分层职责

| 层 | 职责 | 禁止 |
|---|---|---|
| Controller | 触发 Request 场景校验、权限注解、调 Service、返回 Resource | 写业务逻辑、管事务 |
| Request | 字段级基础校验（格式/必填/类型/长度/枚举） | 业务规则、关联数据合法性 |
| Service | 业务校验、事务、日志；方法小驼峰 | 直接处理 HTTP |
| Resource | 数据转换；继承 `BaseResource` / `BaseCollection` | 业务逻辑 |

权限：`#[OrgPermission]` + `OrgMiddleware`。

异常示例：
```php
throw new BusinessException(RoleCode::ROLE_NOT_EXIST);
```

## API / REST

- 资源为中心；`application/json`
- 路径：小写复数 + kebab-case（如 `user-groups`）
- 字段：Snake Case（如 `role_name`）
- 方法语义：`GET` 查 · `POST` 建/增量关联 · `PUT` 全量 · `PATCH` 部分 · `DELETE` 删
- 批量删除：`DELETE /v1/{resources}/batch`（body 传 ID 数组），不用 Query `ids=`

反模式：
| ❌ | ✅ |
|---|---|
| `POST /v1/roles/update/{id}` | `PUT /v1/roles/{id}` |
| `GET /v1/roles/delete?id=1` | `DELETE /v1/roles/1` |
| `POST /v1/roleGroups/...` | `POST /v1/role-groups/...` |

典型 CRUD 形态：`/columns`、列表、`/{id}`、POST、PUT/PATCH、关联子资源、DELETE、`/batch`。

## Request 约定

- 一 Controller 一 Request 类
- 不同方法不同 `@Scene`，规则所见即所得
- 复杂/动态规则进 Service

## Resource 目录

```
app/Resource/
├── Application/V1/...   # 应用层
└── Domain/...           # 领域层
```

## 数据库

| 项 | 规范 |
|---|---|
| 模型/类文件 | 单数（`Photo` / `Photo.php`） |
| 表名 | 复数 snake（`photos`） |
| 迁移名 | 单数 + 时间前缀（`2025_10_10_154417_create_photo_table.php`） |
| 字段 | snake；主键 `id`；外键 `*_id` |
| 时间 | `datetime(6)`；`created_at` / `updated_at` / `deleted_at` |
| 操作人 | `bigint`：`created_by` / `updated_by` / `deleted_by` |

索引：不建无用/重复索引；单表建议 ≤ 5 个。

## 接口约定

| 项 | 规范 |
|---|---|
| Content-Type | `application/json; charset=utf-8` |
| 时间入参 | UTC Unix 时间戳（秒或毫秒） |
| 时间落库 | 北京时间 `Asia/Shanghai` |
| 时间出参 | UTC ISO 8601（如 `2025-01-01T04:00:00.000000Z`） |
| 金额 | 最小货币单位存储，Resource 层格式化 |
| 分页入参 | `current_page`（默认 1）、`per_page`（默认 10，最大 1000）、`sort`、`order` |
| 分页出参 | `items` / `total` / `current_page` / `per_page` / `last_page` |
| 追踪 | 支持 `X-Trace-Id`（中间件已处理） |
| 审计 | 登录、权限变更、资金/关键业务必须记审计日志 |

统一错误响应：
```json
{
  "code": 1001001,
  "message": "用户状态异常",
  "request_id": "a1b2c3d4",
  "data": []
}
```

错误码分段示例：成功 `0` · 通用 `1001xx` · 用户 `2001xx` · 订单 `3001xx`。

## MQ 命名

格式：`[服务名].[功能模块].[操作/事件]`（服务名 = `APP_NAME` 小写）

- Consumer Exchange：同上；Queue：末尾加 `.queue`
- Producer：发往**目标服务**的 Exchange
- 仅字母数字、`.`、`-`；系统级以 `system.` 开头

本服务示例前缀：`merchant.{module}.{action}` / `merchant.{module}.{action}.queue`

## Git

分支：`main` / `dev` / `test` / `feature/{scope}-{short}` / `hotfix/{issue}-{short}`

提交：`type(scope): summary`  
类型：`feat` · `fix` · `docs` · `refactor` · `test` · `chore` · `perf` · `style`

## Agent 执行清单

实现新接口时按序检查：

- [ ] 注解路由 + `/v1/` + kebab-case 复数资源路径
- [ ] Request（BaseRequest + Scene）只做字段校验
- [ ] Service 承载业务规则与事务；日志用 LogHelper
- [ ] 返回经 Resource + ApiResponseHelper
- [ ] 权限用 OrgPermission / OrgMiddleware（如需）
- [ ] 错误码走枚举 + i18n，无魔法数字
- [ ] 表变更有 Migration，命名与字段规范符合上表
- [ ] 涉及 MQ 时命名符合 `[APP_NAME].[module].[action]`
- [ ] 改完跑 `composer cs-fix`（或 `sh cs-fix.sh`），勿依赖保存时格式化

需要端口表、完整 FAQ 或更长示例时，读取 [`merchant-settle-apply-api.md`](../../../merchant-settle-apply-api.md)。
