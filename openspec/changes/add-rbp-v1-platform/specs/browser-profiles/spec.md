# Spec Delta: browser-profiles

## Purpose

管理浏览器 Profile 的逻辑身份、归属与访问控制、运行时独占挂载、磁盘路径安全、home node 绑定以及删除与清理，使登录状态可在授权范围内持久复用且不在用户之间泄漏。

## ADDED Requirements

### Requirement: Profile 实体
Profile SHALL 具有中央生成的 `profile_id`、所属主体 `owner_id`、可选 `project_id`、`mode`（`persistent` | `ephemeral`）、可选 `home_node_id`、`status`（`ACTIVE` | `DELETING` | `DELETED`）、显示名与创建时间。具有 `profile:create` 的主体 SHALL 能通过 `POST /api/v1/profiles` 创建 persistent Profile。

#### Scenario: 创建 Profile
- **WHEN** Operator 调用 `POST /profiles` 创建名为 "test-a" 的 persistent Profile
- **THEN** 返回新的 `profile_id`，`owner_id` 为该 Operator，`home_node_id` 为空

### Requirement: 未指定 Profile 时使用临时 Profile
acquire 请求未指定 `profile_id` 时，系统 SHALL 自动创建 ephemeral Profile，其生命周期绑定到该次租约；租约结束后该 Profile 及其实例 SHALL 被销毁，MUST NOT 被任何后续租约复用。

#### Scenario: 临时 Profile 不复用
- **WHEN** 用户 A 不指定 Profile 申请并释放后，用户 B 也不指定 Profile 申请
- **THEN** B 获得一个全新的 Chrome 实例与空白 Profile

### Requirement: Profile ACL
Profile 默认私有，仅所有者可用。所有者或项目 Project Admin SHALL 能通过 ACL 向其他主体授予角色：`user`（可申请使用）、`manager`（可管理 ACL 与删除）。跨主体使用 MUST 有显式 ACL 记录。

#### Scenario: 未授权使用
- **WHEN** 用户 B 以用户 A 私有 Profile 申请浏览器
- **THEN** 响应为 404 `PROFILE_NOT_FOUND`，无实例被创建

#### Scenario: 显式授权后共享
- **WHEN** A 将 Profile 以 `user` 角色授权给 B，B 再申请
- **THEN** B 的申请被接受，且产生 `profile.acl.grant` 审计记录

### Requirement: Profile 全局独占挂载
同一 Profile 在任意时刻 MUST 至多绑定一个非 `STOPPED` 的实例（含 CREATING/STARTING），该约束 SHALL 在中央数据库以唯一约束保证，并由节点本地 Profile 锁二次保证。

#### Scenario: 并发申请同一 Profile
- **WHEN** 两个客户端同时以同一 Profile 申请且该 Profile 未运行
- **THEN** 只有一个请求获得实例与租约，另一个得到 409 `PROFILE_IN_USE`（或按策略排队）

#### Scenario: 节点本地二次校验
- **WHEN** Worker 收到为已锁定 Profile 创建实例的命令
- **THEN** Worker 拒绝并返回 `PROFILE_IN_USE`

### Requirement: Profile 绑定 home node
persistent Profile 首次在某节点物化时 SHALL 记录 `home_node_id`，此后 MUST 只在该节点运行。home node 不可调度时，申请该 Profile SHALL 返回 503 `NODE_OFFLINE`（或按请求排队），MUST NOT 在其他节点以空白目录启动。Platform Admin SHALL 能解除绑定（明确接受登录状态丢失）。

#### Scenario: home node 离线
- **WHEN** Profile P 的 home node 为 OFFLINE 时申请 P
- **THEN** 响应为 503 `NODE_OFFLINE`，`details.home_node_id` 指出节点，其他节点未启动 P

#### Scenario: 管理员解除绑定
- **WHEN** 管理员解除 P 的 home node 绑定后再次申请 P
- **THEN** P 可在任意匹配节点以新的空白目录启动，并记录审计

### Requirement: Profile 路径安全
Worker SHALL 仅以 `profiles_root\<profile_id>` 形式由 `profile_id` 推导 Profile 目录，`profile_id` MUST 匹配严格字符集（UUID 格式）。包含 `..`、绝对路径、盘符、UNC、保留设备名或经符号链接/联接点解析后越出 `profiles_root` 的路径 MUST 被拒绝。客户端与中央 MUST NOT 提交任意文件路径。

#### Scenario: 路径穿越
- **WHEN** `CREATE_BROWSER` 携带 `profile_id = "..\\..\\Windows"`
- **THEN** Worker 以 `VALIDATION_FAILED` 拒绝，不访问任何目录

#### Scenario: 符号链接越界
- **WHEN** `profiles_root\<id>` 是指向 `profiles_root` 之外的联接点
- **THEN** Worker 拒绝使用该目录并标记 Profile 异常

### Requirement: 登录状态持久化
persistent Profile 在其实例停止后 SHALL 保留磁盘数据；同一授权主体再次申请时 SHALL 加载同一目录，Cookie 与登录状态按 Chrome 与目标站点规则保留。

#### Scenario: 重新加载保留登录
- **WHEN** 用户在 Profile P 中登录站点后实例被停止，再次申请 P
- **THEN** 新实例加载同一目录，站点仍处于登录状态（若站点 Cookie 未过期）

### Requirement: Profile 删除
具有 `manager` 角色或 Project/Platform Admin SHALL 能删除 Profile。删除 MUST 要求请求携带确认字段（`confirm: <profile_id>`），存在非 STOPPED 实例时 MUST 返回 409 `PROFILE_IN_USE`。删除成功后中央标记 `DELETED` 并通知 home node 删除目录，写入审计。

#### Scenario: 使用中删除
- **WHEN** Profile 正被 BUSY 实例使用时请求删除
- **THEN** 响应为 409 `PROFILE_IN_USE`，数据不变

#### Scenario: 缺少确认
- **WHEN** 删除请求未携带匹配的 `confirm`
- **THEN** 响应为 422 `VALIDATION_FAILED`

### Requirement: Profile 列表
`GET /api/v1/profiles` SHALL 返回调用方拥有或被授权的 Profile，包含 `mode`、`home_node_id`、当前运行实例（若有且可见）与状态，MUST NOT 返回 Windows 磁盘路径。

#### Scenario: 不泄露路径
- **WHEN** 查询 Profile 列表
- **THEN** 响应中不出现任何 `C:\` 路径字段
