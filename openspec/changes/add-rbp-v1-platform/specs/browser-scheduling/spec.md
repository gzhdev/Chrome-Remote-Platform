# Spec Delta: browser-scheduling

## Purpose

定义浏览器申请（acquire）的调度行为：授权、节点筛选、空闲实例复用、新建与双层容量准入、空闲实例驱逐、排队/取消/超时、配额与可处理的失败原因，确保在并发与状态延迟下不超配。

## ADDED Requirements

### Requirement: Acquire 请求
`POST /api/v1/leases/acquire`（需 `browser:acquire`）SHALL 接受：`profile_id`（可选）、`reuse`（默认 true）、`wait_timeout_seconds`（默认 0，上限为配置的 `max_queue_wait_seconds`，默认 300）、`constraints.node_tags`（全部匹配）、`constraints.browser`（V1 仅 `chrome`）。成功时 SHALL 返回 `lease_id`、`lease_epoch`、`instance_id`、`node_id`、`state`、`expires_at`、`renew_after_seconds`、`capabilities`；新建实例返回 201，复用返回 200。

#### Scenario: 首次申请新建实例
- **WHEN** 节点空闲时 Operator 申请其 persistent Profile
- **THEN** 返回 201，租约 `ACTIVE`，新 Chrome 已 `BUSY`

### Requirement: 调度顺序
调度 SHALL 依次执行：(1) 校验 Profile 授权；(2) 过滤可调度节点（`ONLINE`、非 Drain、标签匹配、Profile home node 约束）；(3) 复用该 Profile 的 `READY` 且无有效租约的实例；(4) 在有剩余容量的节点新建实例；(5) 驱逐空闲实例腾出容量后新建；(6) 按 `wait_timeout_seconds` 排队或返回 `RESOURCE_EXHAUSTED`。

#### Scenario: 优先复用
- **WHEN** Profile P 已有 `READY` 空闲实例，且 `reuse=true`
- **THEN** 系统复用该实例，不启动新 Chrome

#### Scenario: reuse=false
- **WHEN** Profile P 已有 `READY` 空闲实例，请求 `reuse=false`
- **THEN** 系统停止该实例后以同一 Profile 新建实例（因 Profile 独占），返回 201

### Requirement: 排除不可调度资源
调度 MUST 排除 `PENDING`、`DEGRADED`、`OFFLINE`、Drain 中或认证失效的节点，以及 `UNHEALTHY`、`FAILED`、`STOPPING` 的实例。

#### Scenario: 故障节点不被分配
- **WHEN** 节点 A 为 `DEGRADED`，节点 B 为 `ONLINE` 有容量
- **THEN** 请求被分配到节点 B

### Requirement: 节点选择策略
在多个候选节点中，系统 SHALL 优先选择 `有效容量 - 占用 - 预留` 最大的节点，平局时选择 CPU 使用率较低者。有效容量为 `min(reported_max_instances, policy_max_instances)`。

#### Scenario: 均衡分配
- **WHEN** 节点 A 已用 3/4，节点 B 已用 1/4
- **THEN** 新实例被创建在节点 B

### Requirement: 中央原子预留
中央 SHALL 在单个数据库事务中完成节点容量预留与 Profile 绑定预留，预留记录带过期时间（默认等于创建 deadline + 10 秒）。同一节点 `占用 + 有效预留` MUST NOT 超过有效容量。

#### Scenario: 中央并发预留
- **WHEN** 20 个申请同时竞争一个仅剩 1 个名额的节点
- **THEN** 中央只为其中 1 个请求创建预留，其余尝试其他节点或排队

### Requirement: 节点拒绝后换节点
Worker 以 `NODE_CAPACITY_REACHED` 或 `RESOURCE_EXHAUSTED` 拒绝创建时，中央 SHALL 释放该预留并尝试下一个候选节点；绑定 home node 的 Profile 不换节点而进入排队或失败。

#### Scenario: 一个节点满
- **WHEN** 3 个候选节点中节点 A 本地拒绝创建
- **THEN** 中央在节点 B 或 C 上创建实例并成功返回

### Requirement: 空闲实例超时回收
无有效租约的 `READY` persistent 实例空闲超过 `idle_instance_ttl_seconds`（默认 1800）时，系统 SHALL 停止该实例以释放容量，Profile 数据保留。

#### Scenario: 空闲超时
- **WHEN** 实例在释放租约后 30 分钟内未被再次使用
- **THEN** 系统下发 `STOP_BROWSER`，实例进入 `STOPPED`，节点名额释放，记录 `instance.idle_stop` 审计

### Requirement: 容量不足时 LRU 驱逐
当候选节点无剩余容量时，系统 SHALL 在该节点（或候选节点集合）中选择无有效租约、空闲时长不低于 `min_idle_before_evict_seconds`（默认 60）的 `READY` 实例，按最近使用时间最早优先驱逐，确认其 `STOPPED` 后再为新请求创建实例。BUSY 或持有 GRACE 租约的实例 MUST NOT 被驱逐。

#### Scenario: 驱逐最久未用的空闲实例
- **WHEN** 节点 4/4 中有 2 个空闲实例分别空闲 10 分钟和 2 分钟，新请求需新建
- **THEN** 系统停止空闲 10 分钟的实例并在该名额上创建新实例

#### Scenario: 无可驱逐实例
- **WHEN** 所有候选节点上的实例都是 BUSY
- **THEN** 系统不驱逐任何实例，转入排队或返回 `RESOURCE_EXHAUSTED`

### Requirement: 创建命令幂等与对账
中央下发 `CREATE_BROWSER` SHALL 预先分配 `instance_id` 作为幂等键并携带 deadline。超时未知结果时 SHALL 先查询对账再决定重试或清理，同一 acquire MUST NOT 导致两个实例。

#### Scenario: HTTP 重试
- **WHEN** 客户端因网络超时以同一 `Idempotency-Key` 重试 acquire
- **THEN** 返回同一租约与实例，节点上只存在一个对应 Chrome

### Requirement: 异步排队申请
当 `wait_timeout_seconds > 0` 且无法在同步等待窗口（默认 10 秒，可配置）内分配时，系统 SHALL 返回 202 与 `acquisition_request_id`；调用方 SHALL 能通过 `GET /api/v1/acquisition-requests/{id}` 查询（`WAITING`/`FULFILLED`/`CANCELLED`/`EXPIRED`/`FAILED`，完成时含 lease 信息），通过 `DELETE` 取消。

#### Scenario: 排队后获得资源
- **WHEN** 资源全满时以 `wait_timeout_seconds=120` 申请，60 秒后有实例被释放
- **THEN** 首次响应为 202；之后查询状态为 `FULFILLED` 并含 `lease_id`

#### Scenario: 不允许排队
- **WHEN** 资源全满时以 `wait_timeout_seconds=0` 申请
- **THEN** 立即返回 503 `RESOURCE_EXHAUSTED` 与 `Retry-After`

### Requirement: 排队取消与超时不留孤儿
排队请求被取消或超时 SHALL 释放其所有预留；若取消时实例已在创建中，系统 SHALL 在创建完成后将其转为空闲 READY（persistent）或停止（ephemeral），MUST NOT 产生孤儿租约。

#### Scenario: 创建中取消
- **WHEN** 排队请求在其 `CREATE_BROWSER` 执行中被取消
- **THEN** 请求状态为 `CANCELLED`，无 ACTIVE 租约归属该请求，实例按模式回收

### Requirement: 排队公平性
同一资源的排队请求 SHALL 按到达顺序（FIFO）满足。

#### Scenario: FIFO
- **WHEN** 请求 R1 早于 R2 排队等待同一 Profile
- **THEN** Profile 可用时 R1 先获得租约

### Requirement: 并发配额
系统 SHALL 支持按主体与按项目配置最大并发有效租约数（默认不限制），超限 MUST 返回 429 `QUOTA_EXCEEDED`，MUST NOT 影响其他主体。

#### Scenario: 用户配额满
- **WHEN** 用户 A 配额为 2 且已持有 2 个有效租约时再次申请
- **THEN** A 得到 429 `QUOTA_EXCEEDED`，用户 B 的申请正常分配

### Requirement: 可处理的失败原因
调度失败 SHALL 在 `error.details` 给出机器可读原因：`all_nodes_full`、`no_matching_tags`、`profile_in_use`、`home_node_offline`、`node_resource_pressure`、`quota_exceeded`，以及参与评估的节点数。

#### Scenario: 标签无匹配
- **WHEN** 请求 `node_tags=["mac"]` 而无节点具有该标签
- **THEN** 响应为 503 `RESOURCE_EXHAUSTED`，`details.reason` 为 `no_matching_tags`，`retryable=false`
