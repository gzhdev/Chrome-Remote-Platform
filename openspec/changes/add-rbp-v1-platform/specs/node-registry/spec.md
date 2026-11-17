# Spec Delta: node-registry

## Purpose

管理 Windows 浏览器节点在中央的身份与生命周期：一次性注册、节点凭据、主动上线、能力上报、心跳与失联判定、标签以及 Drain/Resume，使中央只向可信且健康的节点调度。

## ADDED Requirements

### Requirement: 一次性注册码
Platform Admin SHALL 能通过 `POST /api/v1/nodes/enroll-tokens` 生成注册码，可指定有效期（默认 24 小时）、节点显示名与初始标签。注册码 MUST 为高熵随机值、只返回一次、中央仅存哈希，且只能被成功消费一次。

#### Scenario: 注册码重复使用
- **WHEN** 一个注册码已被成功用于注册节点后再次提交 `POST /api/v1/nodes/enroll`
- **THEN** 请求被拒绝为 401，且不创建新节点

#### Scenario: 注册码过期
- **WHEN** 在有效期之后使用注册码
- **THEN** 请求被拒绝，并记录 `node.enroll` 失败审计

### Requirement: 节点注册产生稳定身份
`POST /api/v1/nodes/enroll` 成功后，中央 SHALL 分配由中央生成的 `node_id` 与节点凭据并返回给 Worker。节点 MUST NOT 自选或自报 `node_id`；后续所有认证以中央分配的身份为准。

#### Scenario: 节点尝试自报 ID
- **WHEN** enroll 请求体中携带 `node_id` 字段
- **THEN** 中央忽略或拒绝该字段，返回中央生成的 `node_id`

### Requirement: 节点凭据生命周期
节点凭据 SHALL 具有签发时间与过期时间（默认 90 天）。Worker SHALL 在过期前（默认剩余 ≤ 20%）通过控制通道请求轮换；轮换成功后旧凭据 MUST 在宽限期（默认 5 分钟）后失效。Platform Admin SHALL 能吊销节点凭据，吊销后该节点的现有控制连接 MUST 立即断开。

#### Scenario: 凭据轮换不影响其他节点
- **WHEN** 节点 A 完成凭据轮换
- **THEN** 节点 A 的旧凭据在宽限期后无法认证，节点 B 的连接不受影响

#### Scenario: 吊销凭据
- **WHEN** 管理员吊销节点 A 的凭据
- **THEN** 节点 A 的控制连接被断开，以该凭据重连被拒绝，节点状态变为 `OFFLINE`

### Requirement: 节点主动上线
Worker SHALL 主动以 WSS/TLS 连接 `/ws/v1/nodes/control` 并以节点凭据认证；中央 MUST NOT 主动向节点发起任何网络连接。节点 VM 不开放任何入站端口时 MUST 仍能上线。

#### Scenario: 无入站端口上线
- **WHEN** Windows VM 防火墙阻止全部入站连接，Worker 启动
- **THEN** 节点完成认证、快照对账并显示为 `ONLINE`

### Requirement: 节点能力与状态上报
节点上线后 SHALL 上报：Worker 版本、协议版本、OS 版本、Windows 会话 ID 与用户名、Chrome for Testing 版本、`max_instances` 配置值、CPU/内存总量、现存实例摘要与 Profile 绑定摘要；心跳 SHALL 携带 CPU/内存使用率、磁盘可用空间、实例计数与 `max_instances` 当前值。中央 SHALL 以这些值更新节点记录。

#### Scenario: 管理端看到一致信息
- **WHEN** 节点以 `max_instances: 4` 与 Chrome 版本 X 上线
- **THEN** `GET /api/v1/nodes/{id}` 返回的 `reported_max_instances` 为 4、`chrome_version` 为 X

#### Scenario: 本地配置变更被上报
- **WHEN** 节点本地 `max_instances` 由 4 改为 2 并重载
- **THEN** 下一次心跳后中央的 `reported_max_instances` 变为 2

### Requirement: 节点状态模型
节点 SHALL 处于以下状态之一：`PENDING`（已注册未完成对账）、`ONLINE`（可调度）、`DEGRADED`（控制连接中断但未超失联阈值，或资源软阈值越界）、`OFFLINE`（超过失联阈值或凭据吊销）。`DRAINING` SHALL 作为独立的调度标志与上述状态并存。只有 `ONLINE` 且非 Drain 的节点 SHALL 可接受新调度。

#### Scenario: 对账前不可调度
- **WHEN** 节点认证成功但尚未完成 `NODE_SNAPSHOT` 对账
- **THEN** 节点状态为 `PENDING`，调度器不向其分配新实例

### Requirement: 心跳与失联判定
节点心跳间隔（默认 10 秒）与失联阈值（默认 35 秒）SHALL 可配置。控制连接断开时节点 SHALL 立即变为 `DEGRADED`；自最后一次有效心跳起超过失联阈值 SHALL 变为 `OFFLINE`。状态变化 MUST 在阈值后 5 秒内生效。

#### Scenario: 心跳中断
- **WHEN** 节点停止发送心跳超过 35 秒
- **THEN** 节点状态变为 `OFFLINE`，不再分配新任务，其上 ACTIVE 租约不被立即转给他人

### Requirement: 节点重连
节点重连 SHALL 获得新的、单调递增的 `connection_epoch`，新连接 MUST 取代旧连接（旧连接被关闭）。节点在完成新的快照对账后 SHALL 恢复为 `ONLINE`。重连 SHALL 采用指数退避加抖动，上限默认 60 秒。

#### Scenario: 重复连接
- **WHEN** 同一节点在旧连接未关闭时以有效凭据建立新连接
- **THEN** 中央关闭旧连接，新连接 `connection_epoch` 大于旧值，旧连接后续消息被丢弃

### Requirement: Drain 与 Resume
Platform Admin SHALL 能对节点执行 `POST /nodes/{id}/drain` 与 `POST /nodes/{id}/resume`。Drain 节点 MUST NOT 接受新实例创建或新租约分配，但现有 ACTIVE 租约 SHALL 可继续续租直至释放；Drain 节点上的空闲实例 SHALL 可由管理员停止。

#### Scenario: Drain 不影响现有租约
- **WHEN** 节点 A 被 Drain，其上存在 ACTIVE 租约 L
- **THEN** L 可以续租与使用；新的 acquire 不会分配到节点 A

#### Scenario: Resume 后恢复调度
- **WHEN** 管理员对 Drain 中的节点 A 执行 Resume
- **THEN** 节点 A 重新参与调度

### Requirement: 节点标签
节点 SHALL 支持标签（如 `win11`、`test-intranet`、`project:<id>`），标签键值 MUST 匹配 `^[a-z0-9][a-z0-9:._-]{0,62}$`。标签由注册码初始化并可由 Platform Admin 修改；调度 SHALL 能按标签筛选节点。

#### Scenario: 非法标签
- **WHEN** 管理员为节点设置标签 `Win 11!`
- **THEN** 响应为 422 `VALIDATION_FAILED`，标签不变

### Requirement: 节点中央策略
Platform Admin SHALL 能通过 `PATCH /nodes/{id}/policy` 设置 `policy_max_instances`、CPU/内存软阈值。节点有效容量 SHALL 为 `min(reported_max_instances, policy_max_instances)`（未设置策略时取上报值）。

#### Scenario: 中央下调容量
- **WHEN** 节点上报 4，管理员设置 `policy_max_instances=2`，节点上已有 3 个实例
- **THEN** 有效容量为 2；不新建实例，已有实例不被强制停止

### Requirement: 快照对账规则
中央收到 `NODE_SNAPSHOT` 后 SHALL 以节点实况为物理真相、以中央租约为控制真相进行对账：节点存在而中央无记录的实例 SHALL 标为孤儿并下发停止（persistent Profile 数据保留）；中央记录存在而节点不存在的实例 SHALL 标为 `STOPPED`，其有效租约转为 `FAILED`；双方都存在但 `generation` 不一致的，以节点为准并使旧代际租约失败。对账结果 SHALL 写入审计。

#### Scenario: 中央未知的实例
- **WHEN** 快照中包含中央数据库不存在的 `instance_id`
- **THEN** 中央下发 `STOP_BROWSER` 回收该实例，且不分配给任何租约

#### Scenario: 节点已丢失的实例
- **WHEN** 中央记录实例 I 为 BUSY，但快照中无 I
- **THEN** I 被标为 `STOPPED`，其租约变为 `FAILED`，客户端 CDP 连接被关闭

#### Scenario: 对账不重复创建
- **WHEN** 节点重连且快照包含仍存活的实例
- **THEN** 中央保留这些实例与其租约关联，不下发新的 `CREATE_BROWSER`
