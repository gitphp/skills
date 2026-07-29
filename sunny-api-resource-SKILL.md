---
name: sunny-api-resource
description: Guides sunny Hyperf API Resource usage (BaseResource/BaseCollection, Application vs Domain layers, ApiResponseHelper). Use when writing or reviewing Resource/Collection classes, formatting API responses, or when the user mentions 资源类, BaseResource, or 后端 API 资源类使用指南.
disable-model-invocation: true
---

# sunny API 资源类

完整原文：[`merchant-settle-apply-api.md`](../../../merchant-settle-apply-api.md)。

## 强制规范

除数据流外，**所有对外 API** 必须经 Resource 包装后由 `Tgkwsunny\Helper\ApiResponseHelper` 返回；**禁止**直接返回模型或数组。

本仓库惯用：`Resource::make($data)` / `XxxCollection::make($data)`，再 `ApiResponseHelper::success(...)`。

## 职责边界

Resource **只做数据转换与格式化**，不做业务逻辑、DB 查询、发邮件等。

| 层 | 放哪 | 面向 | 版本 |
|---|---|---|---|
| Application | `app/Resource/Application/V{n}/模块/` | 对外 HTTP API | 必须分 V1/V2… |
| Domain | `app/Resource/Domain/模块/` | 内部 RPC / 跨服务 | 不分版本 |

- HTTP 出站 → Application Resource
- RPC / 跨服务契约 → Domain Resource
- Application 可组合、重命名、格式化多个 Domain 字段

## 目录与命名

```
app/Resource/
├── Application/V1/{Module}/
│   ├── {Name}Resource.php
│   └── {Name}Collection.php
└── Domain/{Module}/
    ├── {Name}Resource.php
    └── {Name}Collection.php
```

| 类型 | 命名 |
|---|---|
| 单资源 | `{Name}Resource` extends `BaseResource` |
| 集合 | `{Name}Collection` extends `BaseCollection` |

列表字段精简、详情字段完整时可拆 `XxxListResource` / `XxxDetailResource`。

## 输入类型

凡可通过 `$this->field` 或数组访问取值的均可：模型、数组、Collection、带 `__get` 的自定义对象。

## 控制器写法（本仓库）

```php
// 单个
return ApiResponseHelper::success(UserResource::make($row));

// 集合 / 分页（BaseCollection 处理分页结构）
return ApiResponseHelper::success(UserCollection::make($paginator));

// 非分页列表
return ApiResponseHelper::success(UserCollection::make($rows)->withPagination(false));
```

勿在 Controller 里拼响应字段数组。

## BaseResource 常用能力

| 方法 | 用途 |
|---|---|
| `when($cond, $value)` | 条件字段（权限/场景） |
| `whenLoaded($relation)` | 仅关联已预加载时返回，防 N+1 |
| `whenPivotLoaded($relation)` | 中间表 |
| `merge($data)` | 合并额外字段 |
| `formatDate($date)` | 日期 |
| `formatMoney($amount)` | 金额 |
| `getEnumText($enum)` | 枚举文案 |

## BaseCollection 常用能力

`withData` · `withMeta` · `withStats` · `filter` · `sort` · 本仓库另见 `withPagination(false)`。

## 实现清单

写/改 Resource 时：

- [ ] 继承 `BaseResource` / `BaseCollection`，目录落在 Application 或 Domain 正确层
- [ ] `toArray()` 只映射/格式化字段；缺省用 `??` 兜底
- [ ] 关联用 `whenLoaded` + 嵌套 Resource/Collection；控制器侧 `with` / `withCount` 预加载
- [ ] 复杂计算、查库、副作用放在 Service，不进 Resource
- [ ] 对外返回一律 `ApiResponseHelper::success(Resource…)`
- [ ] 应用层变更走版本目录，不破坏旧 API 契约时再改 V1

## 反模式

| ❌ | ✅ |
|---|---|
| `return $model` / `return $array` | `ApiResponseHelper::success(XxxResource::make(...))` |
| Resource 里 `->posts()->count()` | 控制器 `withCount`，Resource 读属性 |
| Resource 里发邮件/改库 | Service 处理 |
| 未加载关联直接 `$this->posts` | `whenLoaded('posts')` |
| Domain 与 Application 混放、Application 无版本目录 | 按上表分层 |

## 嵌套示例

```php
public function toArray(): array
{
    return [
        'id' => $this->id,
        'status' => $this->getEnumText($this->status),
        'created_at' => $this->formatDate($this->created_at),
        'role' => new RoleResource($this->whenLoaded('role')),
        'posts' => PostResource::collection($this->whenLoaded('posts')),
        'email' => $this->when($isAdmin, $this->email),
    ];
}
```

更长案例、分页 JSON 形态、聚合 Dashboard 等见原文 [`merchant-settle-apply-api.md`](../../../merchant-settle-apply-api.md)。
