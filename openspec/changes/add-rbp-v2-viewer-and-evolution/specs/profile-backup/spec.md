# Spec Delta: profile-backup

## Purpose

为绑定在 home node 本地磁盘上的持久化 Profile 提供快照备份与恢复：在 Profile 未被使用时把目录快照到中央对象存储（MinIO），支持保留策略、授权恢复与审计，降低节点磁盘损坏、误删或浏览器升级失败造成的登录状态丢失风险。

## ADDED Requirements

### Requirement: 快照触发
persistent Profile 的实例进入 `STOPPED` 且 Profile 锁已释放后，如果目录自上次快照后发生变化，且距上次快照已超过 `min_snapshot_interval_seconds`（默认 21600），系统 SHALL 自动创建快照。具有 `manager` 角色或管理员权限的主体 SHALL 能对未运行的 Profile 手动触发快照。ephemeral Profile MUST NOT 被快照。

#### Scenario: 停止后自动快照
- **WHEN** Profile P 的实例被空闲回收停止，且 P 距上次快照已超过 6 小时
- **THEN** 系统为 P 创建一个新快照，状态最终为 `COMPLETED`

#### Scenario: 运行中不能手动快照
- **WHEN** 对处于 BUSY 实例上的 Profile 手动触发快照
- **THEN** 响应为 409 `PROFILE_IN_USE`

#### Scenario: 临时 Profile 不快照
- **WHEN** 一个 ephemeral 实例停止
- **THEN** 不产生任何快照

### Requirement: 快照一致性且不长时间阻塞使用
快照 SHALL 只在持有该 Profile 本地锁时读取目录，且只用于在节点本地做一份暂存副本（排除 `Cache`、`Code Cache`、`GPUCache` 等可重建的缓存目录）。暂存完成后 MUST 立即释放锁，压缩与上传在锁外异步进行。持锁期间到达的 acquire SHALL 等待锁释放（不超过 `snapshot_lock_timeout_seconds`，默认 30），不得因快照失败。

#### Scenario: 快照期间申请
- **WHEN** 快照正在本地暂存时，有用户申请该 Profile
- **THEN** 申请在暂存结束后继续执行并成功，快照的上传不受影响

#### Scenario: 缓存目录被排除
- **WHEN** 检查快照内容
- **THEN** 其中不包含 `Cache`、`Code Cache`、`GPUCache` 目录

### Requirement: 快照存储与传输
快照 SHALL 被压缩为单一归档文件，存储在 S3 兼容对象存储（MinIO）中，启用服务端加密，对象键为不可推测的随机值。节点 SHALL 使用中央签发的一次性、短期（默认 15 分钟）、只针对该对象的上传授权（预签名 URL）上传，节点 MUST NOT 持有对象存储的长期凭据。上传完成后中央 SHALL 校验 sha256 与大小，校验通过才标记为 `COMPLETED`。

#### Scenario: 节点不持有存储凭据
- **WHEN** 检查节点的配置与凭据存储
- **THEN** 不存在 MinIO 的 access key 或 secret key

#### Scenario: 哈希不一致
- **WHEN** 上传后的对象 sha256 与节点声明不一致
- **THEN** 快照被标记为 `FAILED`，对象被删除，并写入审计

### Requirement: 快照元数据与查询
每个快照 SHALL 记录 `snapshot_id`、`profile_id`、`source_node_id`、`source_windows_user`、`chrome_version`、`created_at`、`size_bytes`、`sha256`、`state`（`PENDING` | `UPLOADING` | `COMPLETED` | `FAILED` | `EXPIRED`）。Profile 的所有者、`manager` 与管理员 SHALL 能通过 `GET /api/v1/profiles/{id}/snapshots` 查询；无权主体 MUST 得到 404。

#### Scenario: 查询快照
- **WHEN** Profile 所有者查询快照列表
- **THEN** 结果按时间倒序返回，每条包含来源节点与 Chrome 版本

### Requirement: 快照保留
系统 SHALL 为每个 Profile 保留最近 `max_snapshots_per_profile`（默认 3）个 `COMPLETED` 快照，并删除超过 `snapshot_max_age_days`（默认 30）的快照。Profile 被删除时，其全部快照 MUST 一并删除。保留清理 SHALL 写入审计。

#### Scenario: 超出数量
- **WHEN** Profile 已有 3 个快照时又生成第 4 个
- **THEN** 最旧的快照被删除

#### Scenario: Profile 删除连带清理
- **WHEN** Profile 被成功删除
- **THEN** 其所有快照对象从 MinIO 中删除，元数据标记为 `EXPIRED`

### Requirement: 恢复到 home node
`manager` 与管理员 SHALL 能把指定 `COMPLETED` 快照恢复到 Profile 的 home node（`POST /api/v1/profiles/{id}/snapshots/{snapshot_id}/restore`，必须提交 `confirm` 字段）。恢复要求该 Profile 没有非 STOPPED 实例。节点 SHALL 先下载并校验哈希，再解压到暂存目录，然后原子替换原目录；原目录 SHALL 保留为备份，直到新目录首次被实例成功加载。恢复失败 MUST 保持原目录不变。

#### Scenario: 恢复成功
- **WHEN** 在同一 home node、同一 Windows 用户下恢复昨天的快照
- **THEN** 之后申请该 Profile 时实例加载恢复后的目录，快照时已有的登录状态仍然有效（站点 Cookie 未过期的前提下）

#### Scenario: 恢复失败回退
- **WHEN** 解压过程中节点磁盘写满
- **THEN** 恢复以 `FAILED` 结束，原 Profile 目录不变且仍可使用

#### Scenario: 使用中禁止恢复
- **WHEN** Profile 正在被使用时请求恢复
- **THEN** 响应为 409 `PROFILE_IN_USE`

### Requirement: 跨节点恢复的限制
把快照恢复到与 `source_node_id` 或 `source_windows_user` 不同的节点或用户时，系统 SHALL 要求 Platform Admin 权限，并且请求中必须显式确认“登录状态可能丢失”（`accept_credential_loss: true`）。系统 MUST 在响应与审计中注明这是跨环境恢复，MUST NOT 承诺登录状态得以保留。恢复成功后 Profile 的 `home_node_id` SHALL 更新为目标节点。

#### Scenario: 未确认风险
- **WHEN** Platform Admin 请求把快照恢复到另一台节点，但未提交 `accept_credential_loss`
- **THEN** 响应为 422 `VALIDATION_FAILED`

#### Scenario: 跨节点恢复后迁移 home node
- **WHEN** 确认风险后，把快照恢复到节点 B
- **THEN** Profile 的 `home_node_id` 变为 B，审计记录标注 `cross_environment=true`

### Requirement: 快照审计
快照的创建、失败、恢复、跨环境恢复与删除 SHALL 写入审计，内容包含操作者（自动触发时为 `system`）、Profile、快照 ID 与结果。

#### Scenario: 恢复被审计
- **WHEN** 管理员恢复一个快照
- **THEN** 产生 `profile.snapshot.restore` 审计事件，包含快照 ID 与目标节点
