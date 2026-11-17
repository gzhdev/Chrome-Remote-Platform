# Design: RBP V1 平台基线

## Context

- 仓库现状：只有 FastAPI 模板（`main.py`、`test_main.http`），`pyproject.toml` 要求 Python ≥3.13，依赖 fastapi/uvicorn，使用 uv 管理（存在 `uv.lock`）。没有任何现有 spec。
- 设计依据：`reference/01-总体架构设计说明书.md`（下称 AD）与 `reference/02-详细需求规格说明书.md`（下称 SRS）。动机见 proposal.md 的 Why 部分；具体需求见本变更 `specs/` 下的 14 个能力。
- 硬约束：Windows 节点只能主动出站；Chrome CDP 只能监听 loopback；Browser Use 1.0 的 `Browser(cdp_url=...)` 不能携带自定义握手 Header；V1 中央服务为单副本。

## Goals / Non-Goals

**Goals:**
- 在单副本中央服务上，使“每个实例至多一个有效控制方”（I1）、“每个 Profile 至多一个运行实例”（I2）、“节点占用 ≤ 有效容量”（I3）、“CDP 不离开 loopback”（I4）四个不变量在所有故障路径上成立，并能通过测试证明。
- 让 PoC 门禁的 6 项验证可独立执行，并在正式实现之前完成。
- 协议与数据模型按 V2 预留扩展点（Viewer、多 Gateway），但 V1 不实现这些功能。

**Non-Goals:**
- 不保证中央高可用，不做 Gateway 多副本路由。
- 不做 Profile 跨节点迁移或同步（Profile 绑定 home node）。
- 不做 CDP 消息级的审查、过滤或只读模式。
- 不提供 HTTP `/json/version` discovery 代理（列为扩展兼容项）。

## Decisions

### D1. 仓库结构：uv workspace 多包

```
chrome-remote-platform/
  pyproject.toml                # workspace 根（[tool.uv.workspace]）
  packages/
    rbp-protocol/               # 共享 Pydantic 模型：控制消息、错误码、状态枚举（py>=3.11）
    rbp-sdk/                    # 客户端 SDK（py>=3.11，依赖 httpx + rbp-protocol）
  services/
    control-server/             # 共享领域层 + 两个入口：rbp-api（REST、调度、后台任务）/ rbp-hub（控制 WSS、隧道、CDP Gateway）
    windows-worker/             # Worker: 控制客户端 + Instance Manager + 本地诊断 API
  web/admin-console/            # Vue 3 + Vite + TypeScript SPA
  deploy/                       # docker-compose、反向代理配置、Windows 安装脚本、Prometheus 规则
  poc/                          # PoC 脚本与报告
  tests/                        # 跨组件集成/故障注入测试
```
- **理由**：中央和 Worker 必须用同一份协议 Schema（CTRL-002），共享包是最直接的保证。SDK 需要单独发布到客户端环境，所以最低版本放宽到 3.11。
- **备选方案**：单包，或分成多个仓库。单包会让 Worker 带上服务端依赖；多个仓库会让协议 Schema 的版本难以同步。

### D2. 中央拆分为 REST 面（rbp-api）与长连接面（rbp-hub）

```
                       +---------------- Reverse Proxy (TLS) ----------------+
   Client/SDK/Console  |  /api/*, /console/*        /ws/*, /cdp/*            |  Windows Node
         |             +------+---------------------------+-----------------+       |
         v                    v                           v                         v
                     +----------------+   内部 RPC   +--------------------+   出站 WSS
                     |    rbp-api     | -----------> |      rbp-hub       | <--------+
                     | REST / 调度 /  |   (HTTP)     | 节点控制连接注册表 |
                     | 租约 / 后台任务| <----------- | 隧道配对 / CDP 转发|
                     +-------+--------+  LISTEN/     +---------+----------+
                             |           NOTIFY 事件            |
                             +-------------> PostgreSQL <-------+
```

- **rbp-api**（无状态）：提供全部 `/api/v1` REST（包括 enroll、截图上传、connect-ticket 签发），负责调度、租约状态机、排队分发和保留期清理。它不持有任何长连接，V1 运行 1 个副本，但设计上允许多副本。后台任务通过 PG advisory lock 选主执行。
- **rbp-hub**（有状态，V1 单实例）：负责 `/ws/v1/nodes/control`、`/ws/v1/nodes/tunnel`、`/cdp/connect`，在内存中维护节点控制连接注册表和待配对隧道表。它处理心跳、节点状态转移、快照对账，校验 ticket 并做隧道转发。这些逻辑与 api 共用同一套领域层和数据库模型，状态直接写入 PG。
- **api → hub 命令**：使用内部 HTTP RPC，`POST /internal/v1/nodes/{node_id}/commands`，请求体包含控制消息与 deadline，hub 同步等待节点的 `COMMAND_RESULT` 后返回。另一个接口是 `POST /internal/v1/leases/{lease_id}/close`（关闭客户端连接，下发 `CLOSE_TUNNEL{fence_epoch}`，等待节点确认后返回结果）。hub 不可达或节点未连接时，返回 `NODE_OFFLINE` 或“结果未知”，api 按 node-control-protocol 的对账规则处理。
- **事件广播**：使用 PG `LISTEN/NOTIFY`，消息体只携带 ID，详细内容从数据库读取。
  - hub → api：`instance_changed`、`node_changed`，用于唤醒 QueueDispatcher。
  - api → hub：`lease_changed`，作为兜底。比如租约扫描器把租约推进到 REVOKING 后，hub 会主动关闭对应连接，即使 api 的 RPC 调用丢失也能生效。hub 启动时及每 30 秒还会全量核对一次活动隧道与租约状态。
- **内部接口安全**：内部 RPC 只在 compose 内网监听，不经反向代理暴露，使用独立的服务间令牌（配置注入，不复用任何客户端或节点凭据）。
- **理由**：CDP 数据面（20 多条隧道、20 MiB 级消息、高频帧）与调度、心跳、租约扫描分处两个事件循环，数据面的抖动不再导致节点被误判失联或租约被误判过期。rbp-api 重启（发版的主要场景）不再中断已建立的 CDP 会话和节点连接。同时 V2 多副本最难的部分（REST 面无状态化）在 V1 就已完成。如果压测发现 Python 转发开销过大，可以只用 Go 重写 rbp-hub，协议 Schema 已导出为 JSON Schema，换语言不受影响。
- **代价**：多出一跳内部 RPC（局域网内为毫秒级），api 需要处理 hub 不可用的情况，部署时多一个服务。
- **备选方案**：单进程（原方案，实现最简单，但数据面与控制面互相干扰，发版时会断开所有会话）；Gateway 单独拆出而节点控制连接留在 api（`OPEN_TUNNEL` 需要再绕一跳，且 api 仍然有状态，因此否决）。

### D3. 数据库：PostgreSQL + SQLAlchemy 2（async）+ Alembic；不变量落在约束上
关键约束（基于 SRS §7.1 扩展）：
- `leases`：部分唯一索引 `UNIQUE(instance_id) WHERE state IN ('ACTIVE','GRACE','RELEASING','REVOKING')` 保证 I1。
- `browser_instances`：部分唯一索引 `UNIQUE(profile_id) WHERE state <> 'STOPPED'` 保证 I2 的中央侧；另有 `next_lease_epoch BIGINT`、`fenced_epoch BIGINT`、`generation INT`、`last_used_at`。
- `capacity_reservations`：容量检查在事务中执行。先用 `SELECT ... FROM nodes WHERE id=? FOR UPDATE` 锁住节点行，再统计 `占用 + 有效预留 < effective_max` 后插入预留，保证 I3 的中央侧。
- `idempotency_records(principal_id, key, request_hash, response_json, created_at)`，主键为 `(principal_id, key)`。
- `acquisition_requests` 支撑排队：FIFO 按 `created_at` 排序，`state` 字段跟踪状态。
- `audit_events` 增加 `prev_hash`、`hash` 列，形成哈希链。
- **理由**：并发正确性交给数据库约束，而不是依赖应用层锁。将来扩成多副本时这一点仍然成立。
- **备选方案**：Redis 分布式锁。它会引入额外组件，并且没有事务一致性。

### D4. 租约状态机细化（对文档的修正）
- 用 `REVOKING`（撤销中，等待 fencing 确认）替代 AD 中的 `REVOKED`，与 `RELEASING` 对称。这两个状态都必须在节点确认 `CLOSE_TUNNEL{fence_epoch}` 之后才能进入 `RELEASED`。
- 补充的转移：`REQUESTED→FAILED`、`GRACE→RELEASING`、`ACTIVE→FAILED`（实例故障）。
- **触发源**：后台租约扫描器每 5 秒运行一次。扫描用 `expires_at < now()` 驱动 ACTIVE→GRACE，用 `grace_until < now()` 驱动 GRACE→REVOKING。转移通过条件 UPDATE 实现（`WHERE state=? AND lease_epoch=?`），防止并发覆盖。
- **CDP 断线不进入 GRACE**：AD §5.2 写的是“心跳/连接中断→GRACE”，这里修正为只有续租超时才进入 GRACE，以满足 LEASE-009。

### D5. Fencing 实现
```
 Central                                  Worker
   | lease L(epoch=8) -> RELEASING          |
   | Gateway: close client WSS(L)           |
   |---- CLOSE_TUNNEL{instance, fence=8} -->|  1. fenced_epoch[instance] = 8 (persist SQLite)
   |                                        |  2. close all tunnels epoch<=8
   |<--- COMMAND_RESULT{ok, fenced=8} ------|  3. reject OPEN_TUNNEL epoch<=8 afterwards
   | instance.fenced_epoch=8; L->RELEASED   |
   | instance READY (assignable)            |
```
- 下一次授予租约时，`lease_epoch = next_lease_epoch++`，所以新 epoch 一定大于 `fenced_epoch`。
- Worker 处理 `OPEN_TUNNEL` 时要求 `lease_epoch > fenced_epoch`，并且 `instance_generation` 与当前实例一致。
- 节点不可达时租约停留在 REVOKING。节点重连后，快照中携带每个实例的 `fenced_epoch`；中央对比后补发 `CLOSE_TUNNEL`，或直接判定已经 fencing。
- **自我 fencing**：控制连接断开时，Worker 关闭全部隧道。在新连接完成 `SNAPSHOT_ACK` 之前，Worker 拒绝所有 `OPEN_TUNNEL`。

### D6. CDP Gateway 数据面（运行在 rbp-hub）
- 客户端 Upgrade 流程：先校验 ticket。ticket 消费在 PG 中原子完成：`UPDATE connect_tickets SET consumed_at=now() WHERE hash=? AND consumed_at IS NULL AND expires_at>now() RETURNING ...`。之后比对租约当前 epoch，再 `accept()` 客户端连接。
- 接着生成 `tunnel_id` 与 32 字节 nonce，下发 `OPEN_TUNNEL`，在 `asyncio.Event` 上等待节点接入，超时 8 秒。
- 转发使用两个协程，各自执行 `async for frame in a: await b.send(frame)`。依靠 websockets 库的有界写缓冲（`max_queue`、`write_limit`）实现背压，帧上限配置在 `max_size`。不解析、不记录帧内容，只累计字节数与帧数。
- **库选择**：FastAPI/Starlette 的 WebSocket 处理客户端侧和节点侧接入；Worker 侧使用 `websockets` 客户端连接中央和本地 Chrome。
- **关闭码约定**：4401 票据无效，4403 无权或已撤销，4409 被同租约的新连接替换，4410 租约终结，4504 隧道超时，1009 帧过大，1011 节点或 CDP 错误。

### D7. Worker 进程模型
- 单进程 asyncio 程序，组件包括：`ControlClient`（WSS、退避重连）、`InstanceManager`（唯一写者，用 asyncio.Lock 串行化容量与状态变更）、`HealthMonitor`、`TunnelManager`、`LocalApi`（Uvicorn 单 worker，绑定 127.0.0.1:8765）。
- 节点级独占锁：用命名互斥体 `Global\RBP-Worker-<node_id>`（pywin32 实现），同时在 data_dir 下放一个锁文件作为兜底。
- Profile 锁：本地 SQLite 的 `profiles.locked_by` 字段加锁定行。另外 Chrome 自身的 `SingletonLock` 也作为辅助证据。
- **Chrome 启动参数**：`--user-data-dir=<profiles_root>\<id> --remote-debugging-port=0 --remote-debugging-address=127.0.0.1 --no-first-run --no-default-browser-check --window-size=1280,900 --disable-features=Translate --download-default-directory`（下载目录通过 CDP `Browser.setDownloadBehavior` 设置为 `downloads\<instance_id>`）。
- **进程树**：用 Windows Job Object 包住 Chrome 主进程（`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` 不启用，否则 Worker 崩溃会连带杀掉 Chrome，违反恢复需求）。强杀时枚举 Job 内的进程，或用 psutil 遍历子孙进程。
- **恢复核验**：依次比对 PID、`create_time`、`exe`、`cmdline` 中的 `--user-data-dir`，再用 `DevToolsActivePort` 与 `/json/version` 复核。
- **任务计划程序**：“用户登录时”触发，使用 `InteractiveToken`，配置失败自动重启（1 分钟间隔，无限次）。

### D8. 调度器实现
- acquire 在一个事务内完成授权、复用尝试和预留。如果需要新建实例，事务提交后再经 hub 内部 RPC 下发 `CREATE_BROWSER`。等待结果的过程不持有数据库事务。
- 同步等待窗口默认 10 秒，超过则返回 202。后台的 `QueueDispatcher` 运行在 rbp-api，收到 `instance_changed`、`node_changed` 通知时唤醒（对应实例释放、实例停止、节点上线等事件），同时每 2 秒兜底轮询一次，按 FIFO 尝试满足排队请求。
- LRU 驱逐时，先在事务中把被驱逐实例标为 `STOPPING`，并为新请求预留容量。等节点确认 `STOPPED` 后才用这份预留下发创建。如果驱逐失败，释放预留，转为排队。
- 每个节点的有效容量缓存在内存中，但准入检查一律以数据库为准。

### D9. 截图路径
- 中央创建 `captures` 记录（PENDING）和一次性上传令牌，然后经控制通道下发 `SCREENSHOT{capture_id, target_id, upload_token, upload_url}`。
- Worker 为截图建立一条独立的本地浏览器级 CDP 连接：`Target.attachToTarget(flatten)` → `Page.captureScreenshot` → `detachFromTarget`。完成后用 HTTPS PUT 上传，中央校验 sha256 后把状态改为 COMPLETED。
- 存储：`CaptureStore` 抽象接口，V1 用本地卷 `/var/lib/rbp/captures/<random>`，以后可以换成 S3/MinIO。
- **风险**：后台标签页可能截图挂起，所以截图强制 10 秒超时（见风险 R3）。

### D10. 鉴权实现
- API Key 格式为 `rbp_<prefix8>_<secret32>`，存储 `HMAC-SHA256(server_pepper, secret)`，按 prefix 索引。
- 节点凭据格式为 `rbpn_<node>_<secret>`，存哈希。节点以 `Authorization: Bearer` 头建立 WSS（Worker 的 websockets 客户端支持自定义 Header，只有 Browser Use 侧不能用）。
- connect ticket 和 upload token 都是 32 字节随机值经 base64url 编码，只存 SHA-256。
- 控制台登录：`POST /api/v1/auth/session` 用 API Key 换取服务端会话，存储在 PG 的 `console_sessions` 表。CSRF 使用 double-submit token。
- 初始管理员：`rbp-control bootstrap-admin` 命令行工具。

### D11. 管理控制台
- Vue 3 + Vite + TypeScript + Vue Router + Pinia，UI 组件库用 Element Plus。API 客户端由 OpenAPI 自动生成（`openapi-typescript`）。
- 构建产物部署为静态文件，由反向代理在 `/console/` 下提供，与 API 同源，以避免 CORS 问题和第三方 Cookie 问题。
- **备选方案**：Jinja2 服务端渲染（用户已否决，选择独立 SPA）。

### D12. 可观测性
- 日志用 `structlog` 输出 JSON。用 contextvars 注入关联字段，中间件统一脱敏 `ticket`、`Authorization`、`Cookie`。
- 指标用 `prometheus-client`。rbp-api 与 rbp-hub 各自暴露 `/metrics`：隧道、配对、心跳类指标来自 hub，调度、租约、HTTP 类指标来自 api。内部 RPC 的耗时与失败数单独统计。Worker 的指标通过心跳汇总到中央，V1 不需要直接抓取 Worker。
- 反向代理的访问日志格式中不记录 query string，或对 `ticket=` 做脱敏。

### D13. 配置
- 中央使用 `pydantic-settings`（环境变量 + YAML）；Worker 使用 `runtime.yaml`（结构见 AD §11.3），由 `watchfiles` 监听 `max_instances` 等可热更新字段，不可热更新的字段需要重启。

## Risks / Trade-offs

- **[R1] Browser Use 1.0.6 通过网关代理连接时的兼容性**（多 Target、auto-attach、长消息、外部浏览器不被关闭）尚未验证。→ 以 PoC 门禁（tasks 第 1 组）作为前置条件。如果失败，就在网关增加 `/json/version` 鉴权代理并改写地址，或调整 SDK 的接入方式，并通过 update-change 回写到 spec。
- **[R2] RDP 断开或锁屏后渲染与截图行为**未知。→ PoC 实测；必要时配置 `tscon` 把会话重定向到 console，或调整组策略，并写入部署文档。
- **[R3] 后台标签页截图挂起，以及额外 CDP 连接对 Browser Use 的干扰**。→ 截图强制超时、使用独立 session、不发送 bringToFront；PoC 第 5 项验证。如果干扰确实存在，V1 降级为“仅前台目标截图”。
- **[R4] rbp-hub 单实例是单点**：hub 重启会断开全部节点连接与 CDP 会话。→ V1 不承诺 HA。rbp-api 可以独立重启而不影响会话；hub 依靠快速重启与重连对账恢复；PG 每日备份。
- **[R4b] api 与 hub 之间状态可能不一致**：RPC 可能丢失，NOTIFY 在连接断开期间会丢消息。→ RPC 结果未知时按对账处理；NOTIFY 只作为唤醒信号，正确性以 PG 状态为准；hub 定期全量核对活动隧道与租约。用故障注入测试覆盖“单独重启 api”和“单独重启 hub”两种场景。
- **[R5] Profile 绑定 home node 后，节点故障期间该 Profile 不可用**。→ 管理员可以解绑（接受登录状态丢失）；迁移能力在后续版本实现。
- **[R6] LRU 驱逐会让用户的“热”实例冷启动**，损失首屏速度，但不会丢失登录状态。→ 设置 `min_idle_before_evict_seconds` 与 `idle_instance_ttl_seconds` 两个可调参数，并统计驱逐次数指标。
- **[R7] 节点失联时 REVOKING 租约长期卡住，实例不可用**。→ 这是为保证 I1 有意牺牲的可用性；配有告警，管理员可在确认节点 VM 已重启后手动“强制判定 fencing”（需要高权限并留审计）。
- **[R8] 同一 Windows 用户下多个 Profile 只是逻辑隔离**。→ 文档明确说明；对于不可信租户，要求拆分到不同 VM 或不同 Windows 用户。
- **[R9] rbp-hub 重启时内存中的待配对隧道表丢失**。→ 隧道本身随重启断开，客户端重新领票即可；hub 启动时把数据库中残留的隧道记录统一标为 `CLOSED(reason=hub_restart)`。
- **[R10] Job Object 不能设置 kill-on-close**，所以 Worker 崩溃时 Chrome 会继续存活。→ 这正是恢复需求所期望的行为；孤儿 Chrome 由恢复核验和快照对账处理。

## Migration Plan

1. 移除 `main.py` 与 `test_main.http`，把根 `pyproject.toml` 改为 workspace 根。
2. 中央部署：`docker compose up`（PostgreSQL、rbp-api、rbp-hub、反向代理）→ 执行 `alembic upgrade head` → 运行 `bootstrap-admin`。
3. 节点部署：运行安装脚本，创建 `browser-runner` 账号、`C:\BrowserRuntime\` 目录、固定版本 CfT 与计划任务 → 人工 RDP 登录 → `rbp-worker enroll --token ...` → 自动上线。
4. 发版顺序：先发 rbp-api（不影响会话），再在低峰期发 rbp-hub。
5. 回滚：中央按镜像标签回滚，数据库迁移提供 downgrade；Worker 与 Chrome 按版本目录切换（`bin\chrome-win64-<ver>`），Profile 数据不动。

## Open Questions

- 锁屏状态下的截图行为只需要“单列记录”，不影响验收结论，等 PoC 结果出来再补充到部署文档即可。
- 审计保留 90 天和截图保留 7 天都已做成可配置项，最终取值等组织规范确认，不影响实现。
- VM 规格（起测 8 vCPU / 16 GiB）由压测确定，不影响代码。
