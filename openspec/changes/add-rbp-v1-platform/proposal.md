# Proposal: Remote Browser Pool（RBP）V1 平台基线

## Why

Browser Use 1.0 通过原生 CDP 控制浏览器，但目前浏览器必须与 Agent 同机运行，无法集中管理 Windows 可见浏览器、持久登录态、并发容量与访问安全。需要一套与 Agent 解耦的远程浏览器资源平台：Agent 运行在用户电脑或 CI，Chrome 运行在远程 Windows VM，由中央服务统一调度、鉴权、独占租约并安全代理 CDP。仓库当前只有 FastAPI 模板，本变更建立 PoC + V1 的完整需求与实施基线（依据 `reference/01-总体架构设计说明书.md`、`reference/02-详细需求规格说明书.md`）。

## What Changes

- 新增 **Windows Browser Service（Worker）**：在交互式登录会话中以单进程管理 Chrome for Testing（可见窗口）实例；每实例独立 Profile/下载目录/动态 loopback CDP 端口；本地 SQLite 持久化；节点本地原子容量预留（默认 `max_instances=4`）；Worker 崩溃后基于多因子核验恢复实例。
- 新增 **节点接入**：一次性注册码 enrollment、可轮换节点凭据、节点**主动** WSS 控制长连接、心跳/失联判定、`connection_epoch` 代际、快照对账、Drain。
- 新增 **控制协议**：统一消息信封、消息类型目录、幂等去重、deadline、主版本兼容规则；统一以 `CLOSE_TUNNEL{fence_epoch}` 取代文档中不一致的 `REVOKE_TUNNEL`。
- 新增 **Browser Control Server**：PostgreSQL 持久化；Profile 与 ACL；“先授权、再复用、再创建、最后排队”调度；双层容量准入；**按需 LRU 驱逐空闲实例**与空闲超时；**持久化 Profile 绑定首次落地节点（home node）**。
- 中央服务拆分为两个进程：**REST 面（rbp-api）**负责对外 REST、调度、租约与后台任务，自身无状态；**长连接面（rbp-hub）**负责节点控制 WSS、节点数据隧道与 CDP Gateway。两者通过内部 RPC 与 PostgreSQL LISTEN/NOTIFY 协作，CDP 数据面的负载不再影响调度与心跳处理。
- 新增 **独占控制租约**：每实例至多一个有效租约，`lease_epoch` 单调递增 fencing；补全文档缺失的状态转移（如 `GRACE→RELEASING`、`REQUESTED→FAILED`）；明确只有续租超时进入 GRACE，CDP 断线不改变租约状态。
- 新增 **Secure CDP Gateway**：一次性短期 connect ticket；浏览器级 CDP WSS 入口（兼容 `Browser(cdp_url=...)`，不依赖自定义握手 Header）；节点主动建立数据隧道并以一次性 nonce 配对；1:1 透明转发、背压、帧上限；撤销/失联时强制关闭；节点在控制连接丢失时自我 fencing。
- 新增 **截图**：按 `instance_id + target_id` 的非破坏性 CDP 截图，媒体走独立 HTTPS 上传通道，授权下载与保留期。
- 新增 **鉴权与审计**：作用域 API Key、角色（Platform Admin / Project Admin / Operator / Node）、默认拒绝、机密不落日志、不可轻易篡改的审计事件。
- 新增 **管理 API 与独立 SPA 管理控制台**（节点、实例、租约、Profile ACL、排队、审计、截图证据）。
- 新增 **RBP Python Client SDK**：acquire / connect-ticket / 后台续租 / finally 释放，提供 Browser Use 接入示例。
- 新增 **可观测性**：结构化日志关联字段、Prometheus 指标、告警、保留与轮转、升级回滚指南。
- 新增 **API 通用约定**：统一错误体与错误码表、`X-Request-Id`、`Idempotency-Key`、异步 acquire 升级路径。
- 新增 **PoC 门禁**：SRS §11.1 六项必过验证作为正式开发前置条件；结论可通过 update-change 回写本变更。
- 不在本变更内：页面实时回显、Viewer 授权、人工接管、工件上传下载、多 Gateway 高可用（见后续变更 `add-rbp-v2-viewer-and-evolution`）。

## Capabilities

### New Capabilities

- `api-conventions`: 对外 REST API 通用约定——基础路径、错误体、错误码与 HTTP 映射、请求追踪、幂等键语义、分页。
- `access-control`: 主体、API Key 作用域、角色与资源级授权（默认拒绝）、初始管理员引导、凭据吊销即时生效、机密脱敏。
- `audit-log`: 高危与管理操作的审计事件记录、查询与保留。
- `node-registry`: 节点注册（一次性注册码）、节点凭据生命周期、在线状态判定、能力上报、标签、Drain/Resume。
- `node-control-protocol`: 节点主动控制 WSS 的消息信封、消息类型、幂等、deadline、代际与版本兼容规则。
- `browser-instance-runtime`: Windows Worker 上 Chrome for Testing 实例的生命周期、状态机、本地容量、CDP 发现与 loopback 约束、健康探测、崩溃恢复。
- `browser-profiles`: Profile 身份、模式（persistent/ephemeral）、ACL、独占挂载、路径安全、home node 归属、删除与清理。
- `browser-scheduling`: 浏览器申请的调度算法、双层容量准入、空闲实例复用与 LRU 驱逐、排队/取消/超时、配额与失败原因。
- `browser-leases`: 独占控制租约状态机、TTL/续租/宽限、`lease_epoch` fencing、撤销与释放、与浏览器生命周期解耦。
- `cdp-gateway`: 一次性 connect ticket、浏览器级 CDP WSS 入口、节点主动数据隧道配对、透明转发、背压与限额、强制关闭。
- `browser-screenshots`: Target 列表、非破坏性 CDP 截图、独立媒体传输、截图元数据、授权下载与保留。
- `admin-console`: 管理 API 资源面与独立 SPA 管理控制台的功能范围。
- `client-sdk`: RBP Python SDK 的申请、连接、续租、释放与 Browser Use 集成契约。
- `platform-observability`: 结构化日志、指标、告警、日志/数据保留、备份恢复与版本升级回滚。

### Modified Capabilities

（无。`openspec/specs/` 当前为空。）

## Impact

- **代码结构**：仓库由单文件 FastAPI 模板改为 uv workspace 多包：`packages/rbp-protocol`（共享 Pydantic Schema）、`services/control-server`（同一代码库提供两个入口：`rbp-api` 为 REST 面，`rbp-hub` 为长连接面）、`services/windows-worker`、`packages/rbp-sdk`、`web/admin-console`（SPA）。现有 `main.py`/`test_main.http` 将被移除。
- **依赖**：FastAPI、Pydantic v2、SQLAlchemy 2 + Alembic、asyncpg、PostgreSQL 16、websockets、httpx、psutil、pywin32（Worker）、prometheus-client、structlog；SPA 采用 Vue 3 + Vite + TypeScript（Pinia、Vue Router、Element Plus）；客户端验证固定 `browser-use==1.0.6`（PoC 后冻结）。
- **Python 版本**：`rbp-protocol` 与 `rbp-sdk` 须兼容 Python ≥3.11（客户端环境），服务端运行 3.13（与现有 `pyproject.toml` 一致）。
- **部署**：中央以 Docker Compose 部署，包括反向代理（TLS/WSS，`/api` 路由到 rbp-api，`/ws` 与 `/cdp` 路由到 rbp-hub）、rbp-api、rbp-hub 和 PostgreSQL；Windows VM 部署 `C:\BrowserRuntime\` 目录结构与任务计划程序登录启动项。
- **外部系统**：无入站端口要求于 Windows VM；反向代理须开启 WebSocket Upgrade 并对 query 中的 ticket 脱敏。
