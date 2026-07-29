---
name: sunny-datetime-i18n
description: Guides sunny internationalized datetime handling — UTC Unix timestamp in, Asia/Shanghai storage, UTC ISO 8601 out via Resource. Use when implementing time filters, created_at/updated_at responses, TimeHelper, start_time/end_time, or when the user mentions 国际化时间字段.
disable-model-invocation: true
---

# sunny 国际化时间字段

完整原文：[`merchant-settle-apply-api.md`](../../../merchant-settle-apply-api.md)。参考实现：user 服务权限组列表筛选、通讯录时间筛选及 Resource 出参。

## 强制约定

1. **入参**：前端按客户端时区换算为 **UTC Unix 时间戳**（秒或毫秒）提交；后端不接受无时区本地字符串作权威时间。
2. **落库**：统一按 **北京时间（`Asia/Shanghai`）** 写入（`Y-m-d H:i:s` / `datetime(6)`），便于运维 SQL / 对账。
3. **出参**：统一 **UTC ISO 8601**（如 `2025-01-01T04:00:00.000000Z`）；**禁止** Resource 再格式化成业务本地 `Y-m-d H:i:s`。
4. **前端展示**：解析 ISO 8601 → 客户端时区 → `Y-m-d H:i:s`。

原则：**传输用 UTC（时间戳进、ISO 8601 出）；库内北京时间存查；本地展示交给前端。**

```text
客户端本地时间
  → UTC 时间戳（秒/毫秒）
  → 后端换算北京时间后落库 / where
  → 响应 UTC ISO 8601（…Z）
  → 前端转本地 Y-m-d H:i:s
```

## 请求

| 项 | 说明 |
|---|---|
| 字段例 | `start_time`、`end_time`、`invite_start_time`、`apply_start_time` |
| 类型 | `integer` / `numeric`（约 10 位秒 / ≥13 位毫秒） |
| 语义 | UTC 绝对时刻 |
| 校验 | 不要用本地时间字符串强校验 |

同一接口内秒/毫秒约定一致；兼容两者时用位数或 `>= 1e12` 判毫秒。

解析优先：`Tgkwsunny\Helper\TimeHelper::parseDateOrTimestamp()` / `toCarbon()`，再 `setTimezone('Asia/Shanghai')`。

## 响应

| 项 | 说明 |
|---|---|
| 格式 | UTC ISO 8601，带 `Z` |
| Resource | 透出模型时间，或 `formatDate()`（约定为透传，**不再**服务端转本地字符串） |

`'created_at' => $this->created_at` 与 `'created_at' => $this->formatDate($this->created_at)` 均可，最终 JSON 须为 UTC ISO 8601。

## 接入清单

- [ ] Request：时间筛选用 `sometimes|integer` 或 `numeric`
- [ ] Service：UTC 时间戳 → `Asia/Shanghai` 后再 `where` / 写入
- [ ] 库内不混存 UTC 墙钟字符串与北京时间
- [ ] Resource 出参为 UTC ISO 8601，不按客户端/北京时间格式化 `Y-m-d H:i:s`
- [ ] 同一接口时间类型参数统一为时间戳

## 样例

```php
// Request
'start_time' => 'sometimes|integer',
'end_time' => 'sometimes|integer',

// Service：筛选 / 落库
use Tgkwsunny\Helper\TimeHelper;

$start = TimeHelper::parseDateOrTimestamp($filters['start_time'])
    ->setTimezone('Asia/Shanghai')
    ->format('Y-m-d H:i:s');
$query->where('created_at', '>=', $start);

// Resource
return [
    'created_at' => $this->created_at, // → 2025-01-01T04:00:00.000000Z
    'updated_at' => $this->updated_at,
];
```

## 禁止清单

| ❌ | ✅ |
|---|---|
| 前端提交无时区 `Y-m-d H:i:s` | UTC 时间戳 |
| 库内 UTC 墙钟或与北京时间混存 | 统一北京时间落库 |
| Resource 返回本地/北京 `Y-m-d H:i:s` | UTC ISO 8601 |
| 出参按客户端时区格式化 | UTC 出参，展示交给前端 |
| 同接口混用时间戳与本地字符串 | 统一时间戳 |
| 忽略秒/毫秒差 1000 倍 | 约定或按量级兼容 |

存量仍返回 `Y-m-d H:i:s` 的接口按排期切换；改造中避免同一资源混用 ISO 与本地字符串且无文档说明。
