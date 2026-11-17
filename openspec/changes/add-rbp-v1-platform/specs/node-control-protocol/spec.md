# Spec Delta: node-control-protocol

## Purpose

定义 Windows 节点主动建立的控制 WSS 上的消息协议：统一信封、消息类型目录、请求/应答关联、幂等、deadline、连接代际与版本兼容，保证命令在断线、重放与迟到情况下仍然安全。

## ADDED Requirements

### Requirement: 控制消息信封
每条控制消息 SHALL 为 JSON 文本帧，包含字段：`protocol_version`（`"<major>.<minor>"`）、`type`、`message_id`（UUID，发送方唯一）、`request_id`（请求/应答关联）、`node_id`、`connection_epoch`、`sent_at`（UTC RFC3339）、`deadline_at`（命令类必填）、`payload`（对象）。结构不合法的消息 MUST 返回 `ERROR`（`code=PROTOCOL_INVALID`）且不执行任何操作。

#### Scenario: 缺少必填字段
- **WHEN** 中央收到缺少 `message_id` 的消息
- **THEN** 中央回复 `ERROR`，`code` 为 `PROTOCOL_INVALID`，不更新任何状态

#### Scenario: 双方使用同一 Schema
- **WHEN** 任一消息类型在中央序列化后由 Worker 反序列化
- **THEN** 双方使用同一共享 Schema 校验通过，字段值无损往返

### Requirement: 消息类型目录
协议 SHALL 定义以下消息类型及方向：`NODE_HELLO`(N→C)、`NODE_WELCOME`(C→N)、`NODE_SNAPSHOT`(N→C)、`SNAPSHOT_ACK`(C→N)、`HEARTBEAT`(N→C)、`HEARTBEAT_ACK`(C→N)、`CREATE_BROWSER`(C→N)、`STOP_BROWSER`(C→N)、`GET_BROWSER`(C→N，可选 `include_targets`)、`OPEN_TUNNEL`(C→N)、`CLOSE_TUNNEL`(C→N，携带 `fence_epoch`)、`SCREENSHOT`(C→N)、`ROTATE_CREDENTIAL`(N→C)、`INSTANCE_EVENT`(N→C，实例状态异步变化)、`COMMAND_RESULT`(双向应答)、`ERROR`(双向)。未知 `type` MUST 以 `ERROR`（`UNKNOWN_TYPE`）应答。

#### Scenario: 未知类型
- **WHEN** Worker 收到 `type=REBOOT_VM`
- **THEN** Worker 回复 `ERROR`，`code=UNKNOWN_TYPE`，不执行任何动作

#### Scenario: 实例异步事件
- **WHEN** Worker 检测到某 Chrome 进程意外退出
- **THEN** Worker 发送 `INSTANCE_EVENT`，携带 `instance_id`、`generation`、新状态与原因

### Requirement: 握手顺序
连接建立后 Worker SHALL 首先发送 `NODE_HELLO`（含协议版本与能力）；中央 SHALL 回复 `NODE_WELCOME`（含分配的 `connection_epoch`、心跳/超时策略、服务端时间）；Worker SHALL 随后发送 `NODE_SNAPSHOT`；中央完成对账后回复 `SNAPSHOT_ACK`。在 `SNAPSHOT_ACK` 之前中央 MUST NOT 向该连接下发 `CREATE_BROWSER`/`OPEN_TUNNEL`。

#### Scenario: 握手前收到命令
- **WHEN** Worker 在收到 `NODE_WELCOME` 前收到 `CREATE_BROWSER`
- **THEN** Worker 以 `ERROR`（`HANDSHAKE_INCOMPLETE`）拒绝

### Requirement: 命令幂等与去重
接收方 SHALL 按 `message_id` 去重：重复的命令消息 MUST NOT 重复执行副作用，而 SHALL 重发首次执行的 `COMMAND_RESULT`。Worker 的去重记录 SHALL 持久化于本地存储，至少保留 24 小时并在 Worker 重启后仍有效。`CREATE_BROWSER` 另 SHALL 以 `instance_id` 作为幂等键。

#### Scenario: 重放 CREATE_BROWSER
- **WHEN** 中央重复发送 `message_id` 相同的 `CREATE_BROWSER`
- **THEN** Worker 不新增 Chrome 实例，回复与首次相同的 `COMMAND_RESULT`

#### Scenario: Worker 重启后重放
- **WHEN** Worker 执行 `CREATE_BROWSER` 后崩溃重启，中央重发同一消息
- **THEN** Worker 依据持久化记录返回既有结果，不重复启动

### Requirement: Deadline 与结果不确定性
命令 SHALL 携带 `deadline_at`；Worker 在 deadline 之后收到的命令 MUST 以 `ERROR`（`DEADLINE_EXCEEDED`）拒绝。中央在 deadline 前未收到结果 MUST 将该命令视为“结果未知”，并通过 `GET_BROWSER` 或下一次快照对账确定真实状态，MUST NOT 推定命令未执行。

#### Scenario: 创建超时后对账
- **WHEN** `CREATE_BROWSER` 在 deadline 前未收到结果
- **THEN** 中央发送 `GET_BROWSER` 查询该 `instance_id`，依据实际状态确认实例或下发清理，不另行创建第二个实例

### Requirement: 连接代际隔离
所有命令与应答 SHALL 携带 `connection_epoch`。中央 MUST 丢弃来自非当前 `connection_epoch` 的消息；Worker MUST 拒绝 `connection_epoch` 小于其已知最大值的命令（`STALE_EPOCH`）。

#### Scenario: 旧连接的迟到应答
- **WHEN** 节点已以 epoch 13 重连，中央收到来自 epoch 12 连接的 `COMMAND_RESULT`
- **THEN** 中央丢弃该消息，状态以 epoch 13 的快照为准

### Requirement: 协议版本兼容
协议版本 SHALL 采用 `major.minor`。同一 major 内新增字段 MUST 向后兼容（接收方忽略未知可选字段）；major 不一致时中央 MUST 在 `NODE_HELLO` 阶段以 `ERROR`（`PROTOCOL_VERSION_UNSUPPORTED`）拒绝并关闭连接。

#### Scenario: 不兼容 major
- **WHEN** 协议版本为 `2.0` 的 Worker 连接仅支持 `1.x` 的中央
- **THEN** 中央拒绝连接，节点不上线，中央与 Worker 日志均记录版本不兼容

### Requirement: 控制通道不承载大负载
单条控制消息 MUST NOT 超过可配置上限（默认 256 KiB）。截图图像与 CDP 数据 MUST NOT 经控制通道传输。

#### Scenario: 超大消息
- **WHEN** 一端发送超过 256 KiB 的控制帧
- **THEN** 接收方拒绝该帧并返回 `ERROR`（`MESSAGE_TOO_LARGE`），心跳不受阻塞

### Requirement: 控制通道故障可诊断
控制连接的 TLS 失败、认证拒绝、握手超时、心跳超时与异常关闭 SHALL 在 Worker 与中央两侧产生结构化日志与计数指标，包含 `node_id`、`connection_epoch` 与原因码。

#### Scenario: 证书不受信任
- **WHEN** Worker 不信任中央证书
- **THEN** Worker 日志记录 TLS 校验失败原因，按退避重试，不降级为明文
