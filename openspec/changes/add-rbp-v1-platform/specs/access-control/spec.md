# Spec Delta: access-control

## Purpose

定义 RBP 的主体、凭据、角色与资源级授权模型，保证默认拒绝、客户端与节点凭据域分离、机密不泄露，并使凭据与权限变更即时生效。

## ADDED Requirements

### Requirement: 主体与凭据域分离
系统 SHALL 区分三类凭据域：客户端主体（user / service account）使用 API Key；Windows 节点使用节点凭据；CDP 连接使用一次性 connect ticket。任一域的凭据 MUST NOT 被其他域的入口接受。

#### Scenario: 客户端 API Key 接入节点控制通道
- **WHEN** 使用有效客户端 API Key 连接 `/ws/v1/nodes/control`
- **THEN** 握手被拒绝，记录认证失败事件

#### Scenario: 节点凭据调用客户端 API
- **WHEN** 使用节点凭据调用 `POST /api/v1/leases/acquire`
- **THEN** 响应为 401 `UNAUTHENTICATED`

### Requirement: API Key 鉴权
客户端与管理 REST API SHALL 要求 `Authorization: Bearer <api-key>`。缺失、格式错误、过期或已吊销的 Key MUST 返回 401 `UNAUTHENTICATED`。API Key 明文 MUST 仅在创建时返回一次，中央只存储其哈希（带不可逆加盐或 HMAC）与可识别前缀。

#### Scenario: 缺少凭据
- **WHEN** 请求不带 `Authorization` 头
- **THEN** 响应为 401 `UNAUTHENTICATED`

#### Scenario: Key 只显示一次
- **WHEN** 管理员创建 API Key 后再次查询该 Key
- **THEN** 查询结果只包含 Key ID、前缀、作用域、过期时间，不含明文

### Requirement: API Key 作用域
每个 API Key SHALL 携带作用域集合，至少包括：`browser:acquire`、`browser:screenshot`、`profile:read`、`profile:create`、`profile:admin`、`admin:nodes`、`admin:instances`、`admin:leases`、`admin:audit`、`admin:principals`。请求所需作用域缺失 MUST 返回 403 `FORBIDDEN`。

#### Scenario: 缺少截图作用域
- **WHEN** 只具有 `browser:acquire` 的 Key 调用截图 API
- **THEN** 响应为 403 `FORBIDDEN`

### Requirement: 角色模型
系统 SHALL 支持角色：Platform Admin（全局管理）、Project Admin（管理其项目的 Profile、ACL，查看项目实例与租约）、Operator（申请已授权 Profile、操作本人租约与截图）。角色授予的权限与 Key 作用域取交集后生效。管理员操作 MUST 同样写入审计。

#### Scenario: Project Admin 越权
- **WHEN** 项目 P1 的 Project Admin 修改项目 P2 的 Profile ACL
- **THEN** 响应为 403 `FORBIDDEN` 或 404，ACL 不变

#### Scenario: 作用域与角色取交集
- **WHEN** Platform Admin 使用仅含 `browser:acquire` 作用域的 Key 调用 `POST /nodes/{id}/drain`
- **THEN** 响应为 403 `FORBIDDEN`

### Requirement: 资源级默认拒绝
对 Profile、租约、实例、截图的访问 SHALL 以默认拒绝为原则：仅资源所有者、ACL 显式授权主体、资源所属项目的 Project Admin、Platform Admin 可按其角色访问。

#### Scenario: 读取他人租约
- **WHEN** 用户 A 调用 `GET /api/v1/leases/{B 的 lease_id}`
- **THEN** 响应为 404，且不返回任何租约信息

#### Scenario: 非管理员停止他人实例
- **WHEN** Operator 调用 `DELETE /api/v1/instances/{他人正在使用的 instance_id}`
- **THEN** 请求被拒绝，实例与租约状态不变

### Requirement: 凭据吊销即时生效
API Key 吊销、过期或主体被禁用后，后续请求 MUST 立即被拒绝；该主体持有的 ACTIVE/GRACE 租约 SHALL 被撤销，其活动 CDP 连接 MUST 在撤销流程中被关闭。

#### Scenario: 吊销 Key 后连接被关闭
- **WHEN** 管理员吊销某 Operator 的 API Key，该 Operator 正持有 ACTIVE 租约与 CDP 连接
- **THEN** 其后续 REST 请求返回 401，租约进入撤销流程，CDP WSS 被关闭

### Requirement: 初始管理员安全引导
系统 SHALL 提供一次性的初始管理员引导（服务端命令行），生成首个 Platform Admin 主体与 API Key 并仅输出一次。系统 MUST NOT 内置默认口令或默认 Key；引导在已存在 Platform Admin 时 MUST 拒绝执行（除非显式强制并记录审计）。

#### Scenario: 重复引导
- **WHEN** 已存在 Platform Admin 时再次执行引导命令
- **THEN** 命令失败退出，不创建新管理员

#### Scenario: 无默认凭据
- **WHEN** 新部署未执行引导即访问管理 API
- **THEN** 所有管理请求返回 401

### Requirement: 控制台会话
管理控制台 SHALL 以 API Key 登录换取服务端会话（HttpOnly、Secure、SameSite=Strict Cookie，默认 8 小时空闲过期），写操作 MUST 具备 CSRF 防护。会话权限等同于登录 Key 的角色与作用域。

#### Scenario: 会话过期
- **WHEN** 控制台会话空闲超过配置时间后发起操作
- **THEN** 响应为 401，控制台跳转登录页

### Requirement: 机密不进入日志
API Key、节点凭据、connect ticket、tunnel nonce、Cookie、CDP 消息正文 MUST NOT 以明文出现在访问日志、应用日志、审计、指标标签或错误响应中。URL query 中的 ticket SHALL 在中央与反向代理日志中脱敏。

#### Scenario: 日志抽查
- **WHEN** 执行完整 acquire → connect → 操作 → release 流程后检索全部日志
- **THEN** 不能检索到任何 API Key、ticket 或 nonce 明文

### Requirement: 节点凭据安全存储
节点凭据在 Windows 端 SHALL 使用 DPAPI（当前用户范围）加密保存；中央 SHALL 仅保存其哈希。

#### Scenario: 凭据文件被复制到其他机器
- **WHEN** 将 `node-credential.bin` 复制到另一台 Windows 机器上的不同用户运行 Worker
- **THEN** 凭据无法解密，Worker 无法认证
