# Spec Delta: browser-leases

## Purpose

定义浏览器独占控制租约：唯一性、状态机、TTL/续租/宽限、`lease_epoch` fencing、释放与撤销流程，以及租约与浏览器、数据通道生命周期的解耦，保证任何时刻一个浏览器实例至多一个有效控制方。

## ADDED Requirements

### Requirement: 单实例单有效租约
同一 Browser Instance 在任意时刻 MUST 至多存在一个处于 `ACTIVE`、`GRACE`、`RELEASING` 或 `REVOKING` 状态的租约，该不变量 SHALL 由数据库部分唯一约束保证，并在所有故障恢复路径中保持。

#### Scenario: 并发争抢
- **WHEN** 两个请求同时竞争同一空闲实例
- **THEN** 仅一个获得 `ACTIVE` 租约，另一个得到 409 `LEASE_CONFLICT` 或被调度到其他资源

### Requirement: 租约绑定与代际
租约 SHALL 绑定 `principal_id`、`profile_id`、`instance_id`、`instance_generation`、`expires_at` 与 `lease_epoch`。`lease_epoch` SHALL 按实例单调递增，每次新租约授予时加一。只有租约所属主体（或管理员）可查询、续租、释放该租约。

#### Scenario: 他人续租
- **WHEN** 用户 B 调用 A 的租约 `renew`
- **THEN** 响应为 404，租约不变

#### Scenario: epoch 递增
- **WHEN** 实例 I 先后被授予租约 L1 与 L2
- **THEN** `L2.lease_epoch > L1.lease_epoch`

### Requirement: 租约状态机
租约状态 SHALL 为：`REQUESTED`、`WAITING`、`ACTIVE`、`GRACE`、`RELEASING`、`REVOKING`、`RELEASED`、`CANCELLED`、`FAILED`。允许转移：REQUESTED→WAITING|ACTIVE|FAILED；WAITING→ACTIVE|CANCELLED|FAILED；ACTIVE→GRACE|RELEASING|REVOKING|FAILED；GRACE→ACTIVE|RELEASING|REVOKING；RELEASING→RELEASED；REVOKING→RELEASED；其余 MUST 以 `INVALID_STATE_TRANSITION` 拒绝。`RELEASED`、`CANCELLED`、`FAILED` 为终态。

#### Scenario: 终态不可续租
- **WHEN** 对 `RELEASED` 租约调用 renew
- **THEN** 响应为 409 `LEASE_EXPIRED`

#### Scenario: 实例故障导致租约失败
- **WHEN** ACTIVE 租约对应的 Chrome 崩溃
- **THEN** 租约转为 `FAILED`，客户端查询得到 `state=FAILED` 与原因 `instance_failed`

### Requirement: TTL 与续租
租约 TTL（默认 120 秒）与建议续租间隔（默认 30 秒，经 `renew_after_seconds` 返回）SHALL 可配置。`POST /leases/{id}/renew` 成功 SHALL 将 `expires_at` 延长为当前时间 + TTL 并返回新值；续租请求 MAY 携带 `lease_epoch`，不一致 MUST 返回 409 `STALE_EPOCH`。

#### Scenario: 正常续租不回收
- **WHEN** 客户端每 30 秒续租，持续 1 小时
- **THEN** 租约始终为 `ACTIVE`，未被回收

### Requirement: 续租超时进入宽限
租约到达 `expires_at` 未续租时 SHALL 转为 `GRACE`，`grace_until = expires_at + reconnect_grace_seconds`（默认 60 秒）。GRACE 期间实例 MUST NOT 被分配给其他主体；原主体续租成功 SHALL 回到 `ACTIVE`；超过 `grace_until` SHALL 转为 `REVOKING`。

#### Scenario: 宽限内恢复
- **WHEN** 客户端停止续租 130 秒后（GRACE 中）重新续租
- **THEN** 租约回到 `ACTIVE`，`lease_epoch` 不变

#### Scenario: 宽限期内他人不可获得
- **WHEN** 租约处于 GRACE 时其他用户申请同一 Profile
- **THEN** 该申请得到 `PROFILE_IN_USE` 或排队，不获得该实例

### Requirement: 数据通道断开不改变租约
CDP 数据通道的断开 MUST NOT 改变租约状态或重置实例；租约状态与隧道状态 SHALL 分开存储。ACTIVE 或 GRACE 租约的所属主体 SHALL 能重新领取 connect ticket 重连。

#### Scenario: CDP 断线重连
- **WHEN** 客户端 CDP WSS 意外断开，租约仍在 TTL 内
- **THEN** 租约仍为 `ACTIVE`，客户端领取新票据后重新连接成功

### Requirement: 释放流程
`DELETE /api/v1/leases/{id}` SHALL 幂等：将租约转为 `RELEASING`，关闭该租约所有 CDP 隧道并要求节点确认 fencing（拒绝该 epoch 的任何隧道），确认后转为 `RELEASED`；持久化实例回到 `READY`（空闲计时开始），ephemeral 实例被停止。对已释放租约重复调用 SHALL 返回成功。

#### Scenario: 释放后实例保留
- **WHEN** Agent 完成任务并释放 persistent Profile 的租约
- **THEN** 租约 `RELEASED`，实例为 `READY` 且 Chrome 未退出

#### Scenario: 重复释放
- **WHEN** 对同一租约连续调用两次 DELETE
- **THEN** 两次均返回成功，第二次无额外副作用

### Requirement: 先 fencing 后重新分配
租约从 `RELEASING`/`REVOKING` 转为 `RELEASED` 之前，系统 MUST 获得节点对 `CLOSE_TUNNEL{fence_epoch}` 的确认（节点已关闭该实例所有 `lease_epoch <= fence_epoch` 的隧道并拒绝其新建）。在确认之前实例 MUST NOT 被授予新租约。节点不可达时租约 SHALL 停留在 `REVOKING`，实例不可再分配，直至节点重连对账确认。

#### Scenario: 旧 Agent 不退出
- **WHEN** 租约 L1 被撤销，但持有 L1 的旧 Agent 仍持续发送 CDP 命令
- **THEN** 旧 WSS 被关闭，旧 Agent 的重连被拒绝；新租约 L2 仅在 fencing 确认后授予，不存在双控

#### Scenario: 节点失联时撤销
- **WHEN** 撤销租约时节点控制通道断开
- **THEN** 租约保持 `REVOKING`，实例不被分配，节点重连并确认后租约才变为 `RELEASED`

### Requirement: 陈旧代际拒绝
携带过期 `lease_epoch` 或 `instance_generation` 的 `OPEN_TUNNEL`、renew、STOP 等操作 MUST 被以 `STALE_EPOCH` 拒绝。

#### Scenario: 迟到的 OPEN_TUNNEL
- **WHEN** 节点在已确认 fence_epoch=8 之后收到 `lease_epoch=8` 的 `OPEN_TUNNEL`
- **THEN** 节点拒绝并回复 `STALE_EPOCH`

### Requirement: 管理员强制撤销
具有 `admin:leases` 的管理员 SHALL 能通过 `POST /api/v1/leases/{id}/revoke`（必填 `reason`）强制撤销租约，走与过期撤销相同的 fencing 流程，并写入含操作者与原因的审计。

#### Scenario: 管理撤销
- **WHEN** 管理员撤销 ACTIVE 租约
- **THEN** 客户端 CDP WSS 被关闭，后续 renew 返回 409 `LEASE_EXPIRED`，审计可查

### Requirement: 浏览器生命周期解耦
租约结束 MUST NOT 自动结束 persistent 实例；Agent 进程退出不应导致 Chrome 退出。实例的停止仅由空闲回收、驱逐、管理员操作或故障触发。

#### Scenario: Agent 退出后复用
- **WHEN** Agent 进程退出（租约随后被释放或过期撤销），之后另一授权用户申请同一 Profile
- **THEN** 复用同一 READY 实例，返回 200

### Requirement: 租约查询
`GET /api/v1/leases/{id}` SHALL 返回租约状态、`lease_epoch`、`expires_at`、`grace_until`、实例与节点 ID、当前隧道状态摘要；`GET /api/v1/leases`（管理）SHALL 支持按主体、实例、节点、状态过滤。

#### Scenario: 查询 GRACE 租约
- **WHEN** 查询处于宽限期的租约
- **THEN** 返回 `state=GRACE` 与 `grace_until`
