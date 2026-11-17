# Spec Delta: api-conventions

## Purpose

定义 RBP 中央服务所有对外 REST API 共同遵守的约定：基础路径、统一错误体与错误码、请求追踪、幂等键与分页，使客户端 SDK、管理控制台与第三方调用方获得一致、可处理的行为。

## ADDED Requirements

### Requirement: 统一 API 基础路径与内容类型
中央服务 SHALL 在 `/api/v1` 下提供所有客户端与管理 REST API，请求与响应体 SHALL 使用 `application/json`（截图二进制下载除外）。生产环境 SHALL 只通过 HTTPS 提供服务，明文 HTTP 请求 MUST 被拒绝或重定向，且不得处理业务。

#### Scenario: JSON 响应
- **WHEN** 客户端以有效凭据调用 `GET /api/v1/profiles`
- **THEN** 响应 `Content-Type` 为 `application/json`，主体为 JSON 文档

#### Scenario: 生产环境拒绝明文
- **WHEN** 生产配置下客户端以 `http://` 访问任一 `/api/v1` 端点
- **THEN** 请求不被业务处理，返回重定向到 HTTPS 或连接被拒绝

### Requirement: 统一错误响应体
所有非 2xx 响应 SHALL 返回统一错误体 `{"error": {"code", "message", "retryable", "details"}, "request_id"}`。`code` MUST 取自错误码目录中的稳定枚举，`retryable` MUST 准确指示调用方是否可重试；服务端 MUST NOT 以未分类的 500 表达可预期的业务失败。

#### Scenario: 业务失败使用稳定错误码
- **WHEN** 申请的 Profile 已被其他运行实例占用
- **THEN** 响应为 HTTP 409，`error.code` 为 `PROFILE_IN_USE`，`retryable` 为 `true`，并含 `request_id`

#### Scenario: 未预期异常
- **WHEN** 服务端发生未分类内部异常
- **THEN** 响应为 HTTP 500，`error.code` 为 `INTERNAL`，`message` 不包含堆栈、SQL 或机密信息

### Requirement: 错误码目录与 HTTP 映射
系统 SHALL 至少支持下列错误码及推荐状态码：`UNAUTHENTICATED`(401)、`FORBIDDEN`(403)、`PROFILE_NOT_FOUND`(404)、`NOT_FOUND`(404)、`VALIDATION_FAILED`(422)、`PROFILE_IN_USE`(409)、`LEASE_CONFLICT`(409)、`LEASE_EXPIRED`(409)、`NODE_CAPACITY_REACHED`(409)、`INVALID_STATE_TRANSITION`(409)、`STALE_EPOCH`(409)、`IDEMPOTENCY_KEY_REUSED`(422)、`QUOTA_EXCEEDED`(429)、`RATE_LIMITED`(429)、`RESOURCE_EXHAUSTED`(503)、`NODE_OFFLINE`(503)、`CDP_UNHEALTHY`(502)、`CHROME_START_TIMEOUT`(504)、`TUNNEL_TIMEOUT`(504)、`INTERNAL`(500)。

#### Scenario: 全局容量不足
- **WHEN** 所有匹配节点均无可用容量且请求不允许排队
- **THEN** 响应为 HTTP 503，`error.code` 为 `RESOURCE_EXHAUSTED`，并携带 `Retry-After` 头

#### Scenario: 错误码与 OpenAPI 一致
- **WHEN** 生成 OpenAPI 文档
- **THEN** 每个端点声明的错误响应码与错误码目录一致，且错误体 Schema 被引用

### Requirement: 无权资源的隐藏策略
对调用方无权知晓其存在的私有资源（Profile、租约、实例、截图），系统 SHALL 返回 404 而非 403，避免资源枚举；对调用方可见但无操作权限的资源 SHALL 返回 403 `FORBIDDEN`。

#### Scenario: 访问他人私有 Profile
- **WHEN** 用户 B 请求 `GET /api/v1/profiles/{A 的私有 profile_id}`
- **THEN** 响应为 404 `PROFILE_NOT_FOUND`，响应体不泄露该 Profile 的任何属性

### Requirement: 请求追踪标识
系统 SHALL 接受客户端提供的 `X-Request-Id`（缺省时自动生成 UUID），并在响应头和错误体中回传同一值；该值 SHALL 贯穿中央日志、下发节点的控制消息 `request_id` 与审计记录。

#### Scenario: 透传请求 ID
- **WHEN** 客户端以 `X-Request-Id: abc-123` 调用 acquire
- **THEN** 响应头 `X-Request-Id` 为 `abc-123`，对应 `CREATE_BROWSER` 控制消息与相关日志均可按该 ID 检索

### Requirement: 写操作幂等键
所有产生资源或副作用的写操作（至少 acquire、renew、release、connect-ticket、创建 Profile、截图、管理撤销）SHALL 支持 `Idempotency-Key` 头。相同主体、相同键、相同请求体的重复请求 MUST 返回首次请求的结果且不产生额外副作用；相同键但请求体不同 MUST 返回 422 `IDEMPOTENCY_KEY_REUSED`。幂等记录 SHALL 至少保留 24 小时。

#### Scenario: 重复 acquire 不重复建实例
- **WHEN** 客户端使用同一 `Idempotency-Key` 与相同请求体连续两次调用 `POST /leases/acquire`
- **THEN** 两次返回相同的 `lease_id` 与 `instance_id`，系统只创建了至多一个 Chrome 实例

#### Scenario: 幂等键被复用于不同请求
- **WHEN** 客户端用同一 `Idempotency-Key` 提交不同 `profile_id` 的 acquire
- **THEN** 响应为 422 `IDEMPOTENCY_KEY_REUSED`，不执行第二次请求

#### Scenario: 不同主体使用相同键
- **WHEN** 用户 A 与用户 B 恰好使用相同 `Idempotency-Key`
- **THEN** 两个请求相互独立处理，互不返回对方结果

### Requirement: 列表分页与过滤
所有列表端点 SHALL 支持基于游标的分页（`limit` 默认 50、上限 200，`cursor` 不透明），并返回 `next_cursor`；列表结果 MUST 只包含调用方有权查看的资源。

#### Scenario: 超出上限
- **WHEN** 调用方传入 `limit=1000`
- **THEN** 系统按 200 截断或返回 422 `VALIDATION_FAILED`，行为在 OpenAPI 中明确

#### Scenario: 列表权限过滤
- **WHEN** Operator 调用 `GET /api/v1/profiles`
- **THEN** 结果只包含其拥有或被 ACL 授权的 Profile

### Requirement: 请求频率与连接数限制
系统 SHALL 按主体对 REST 请求与 WSS 建连实施可配置的频率与并发上限，超限 MUST 返回 429 `RATE_LIMITED` 并携带 `Retry-After`。

#### Scenario: 超出频率限制
- **WHEN** 单一 API Key 在一秒内超过配置的请求数
- **THEN** 超出的请求得到 429 `RATE_LIMITED`，其他主体的请求不受影响
