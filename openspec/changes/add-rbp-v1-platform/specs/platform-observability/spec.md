# Spec Delta: platform-observability

## Purpose

为 RBP 的中央服务与 Windows Worker 提供跨组件可关联的结构化日志、运行指标、健康诊断、告警、日志与数据保留、备份恢复以及固定版本升级回滚能力，使故障可定位、容量可度量。

## ADDED Requirements

### Requirement: 结构化日志与关联字段
中央与 Worker SHALL 输出 JSON 结构化日志，每条日志包含 `ts`、`level`、`component`、`event`，并在适用时包含 `trace_id`、`request_id`、`node_id`、`instance_id`、`lease_id`、`tunnel_id`、`connection_epoch`。同一业务流程 SHALL 可通过 `request_id` 跨客户端、中央、Worker 检索。

#### Scenario: 跨组件追踪
- **WHEN** 以 `request_id=R` 执行一次 acquire 导致新建实例
- **THEN** 中央与对应 Worker 日志中均可检索到 `request_id=R` 的记录

### Requirement: 运行指标
中央 SHALL 在受保护的 `/metrics` 端点（Prometheus 格式，需 `admin:audit` 或独立抓取令牌）暴露至少：在线/各状态节点数、节点心跳延迟、每节点 used/effective_max、实例各状态数、创建成功/失败数与耗时、排队长度与等待时长、各状态租约数、过期/撤销租约计数、活动 CDP 隧道数、隧道配对耗时、隧道字节数、CDP 异常断开数、截图成功/失败数与耗时、Profile 锁冲突数、HTTP 请求耗时分布。指标标签 MUST NOT 包含机密或高基数 URL。

#### Scenario: 容量指标
- **WHEN** 节点 A 运行 3 个实例且有效容量为 4
- **THEN** `/metrics` 中节点 A 的 used=3、max=4

#### Scenario: 未授权抓取
- **WHEN** 无凭据访问 `/metrics`
- **THEN** 响应为 401

### Requirement: 健康与版本端点
中央 SHALL 提供 `/healthz`（进程存活）与 `/readyz`（数据库可用、迁移已执行）；中央与 Worker SHALL 暴露自身版本与构建信息。Worker 本地诊断接口 SHALL 返回实例状态摘要与最近错误。

#### Scenario: 数据库不可用
- **WHEN** PostgreSQL 不可达
- **THEN** `/readyz` 返回 503，`/healthz` 返回 200

### Requirement: 告警
系统 SHALL 提供告警规则（Prometheus 规则文件）覆盖：节点 OFFLINE、控制连接频繁重连、节点磁盘可用空间低于阈值（默认 10%）、实例长时间 UNHEALTHY、租约长时间 REVOKING、排队等待超阈值、CDP 隧道配对失败率超阈值。

#### Scenario: 节点离线告警
- **WHEN** 节点 OFFLINE 持续超过 1 分钟
- **THEN** 告警规则触发，标签包含 `node_id`

### Requirement: 日志级别与轮转
日志级别 SHALL 可配置；Worker 日志与 Chrome 实例日志 SHALL 按大小与天数轮转（默认单文件 50 MiB、保留 14 天），MUST NOT 无限占用 Windows 磁盘。

#### Scenario: 日志轮转
- **WHEN** Worker 日志超过 50 MiB
- **THEN** 日志被轮转，超过保留期的旧文件被删除

### Requirement: 数据保留配置
日志、审计（默认 90 天）、截图（默认 7 天）、已终止租约与隧道记录（默认 30 天）的保留期 SHALL 可配置并由后台任务执行。

#### Scenario: 隧道记录清理
- **WHEN** 隧道记录关闭超过 30 天
- **THEN** 清理任务删除该记录

### Requirement: 备份与恢复
系统 SHALL 提供 PostgreSQL 备份与恢复脚本及演练文档；节点配置与 Profile 数据备份 SHALL 与中央数据库备份分开说明。

#### Scenario: 恢复演练
- **WHEN** 在新环境用备份恢复中央数据库
- **THEN** 节点、策略、Profile 元数据与 ACL 被找回，节点以原凭据重连并完成对账

### Requirement: 固定版本与升级回滚
Chrome for Testing 与 Browser Use 版本 SHALL 被精确固定并在部署清单中声明；SHALL 提供 Worker 与 Chrome 的升级与回滚说明，升级 MUST NOT 覆盖或删除 Profile 数据，升级前 SHALL 执行 CDP 回归用例。

#### Scenario: 回滚 Chrome
- **WHEN** 升级 Chrome 后回归失败，执行回滚
- **THEN** 节点恢复旧版本 Chrome，Profile 数据仍可加载

### Requirement: REST 面与长连接面独立重启
中央的 REST 面（对外 REST、调度、租约管理）与长连接面（节点控制连接、CDP 数据转发）SHALL 能独立部署、独立重启。仅 REST 面重启时，已建立的节点控制连接与 CDP 会话 MUST NOT 被中断，在 REST 面恢复前到期的租约 SHALL 在其恢复后按 GRACE/REVOKING 规则处理。长连接面不可用时，REST 面 SHALL 对需要下发节点命令的请求返回 503（`NODE_OFFLINE` 或 `RESOURCE_EXHAUSTED`），且不得留下未对账的预留。

#### Scenario: 只重启 REST 面
- **WHEN** Agent 正经 CDP 操作远程浏览器时重启 REST 面进程
- **THEN** Agent 的 CDP 连接不中断；REST 面恢复后续租成功

#### Scenario: 长连接面不可用
- **WHEN** 长连接面进程停止期间提交需要新建实例的 acquire
- **THEN** 响应为 503 且 `retryable=true`，没有遗留的容量预留

#### Scenario: 撤销在命令丢失时仍然生效
- **WHEN** 租约被撤销，但 REST 面发往长连接面的关闭命令丢失
- **THEN** 长连接面依据租约状态变更通知或周期核对关闭该租约的 CDP 连接，租约最终进入 `RELEASED`

### Requirement: 中央重启恢复
中央服务重启后 SHALL 从 PostgreSQL 恢复租约与实例影子状态，等待节点重连对账，MUST NOT 无条件停止或重启节点上的 Chrome。中央重启期间到期的租约 SHALL 按 GRACE/REVOKING 规则处理。

#### Scenario: 中央重启
- **WHEN** 中央服务重启，节点与客户端重新连接
- **THEN** 节点完成对账并恢复可调度，原 READY 实例未被清理，持有租约的客户端可重新领票连接
