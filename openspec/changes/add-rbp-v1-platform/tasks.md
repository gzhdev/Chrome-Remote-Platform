# Tasks: RBP V1 平台基线

> 顺序依据 SRS §12.1 工作包依赖：PoC 门禁 → 协议 → Worker / 中央核心 → Gateway → SDK / 截图 / 控制台 → 集成验证。
> 每组自带测试与文档；最后一组只做跨组件集成、压测与故障注入。

## 1. PoC 门禁（正式开发前必须通过，SRS §11.1）

- [ ] 1.1 在 `poc/` 编写最小中继：一个 FastAPI 进程提供 `/cdp/connect?ticket=` 与 `/poc/tunnel`，Windows 端脚本主动连接并把流量转发到本地 CfT 的 browser WS；验证方式：脚本可运行，并在 `poc/README.md` 记录启动步骤
- [ ] 1.2 PoC-1：在 `browser-use==1.0.6` 下用 `Browser(cdp_url=wss://...)` 完成打开网址、点击、读取文本、截图、新建标签页，并确认 Agent 结束后 Chrome 未被关闭；验证方式：`poc/reports/poc-1.md` 附日志与结论
- [ ] 1.3 PoC-2：断开 RDP（不注销）后执行页面操作与截图，锁屏行为单列记录；验证方式：`poc/reports/poc-2.md`
- [ ] 1.4 PoC-3：同一会话启动 4 个独立 Profile 的 CfT，验证 Cookie 隔离、动态端口无冲突，记录 CPU/内存占用；验证方式：`poc/reports/poc-3.md`
- [ ] 1.5 PoC-4：在 Windows 防火墙禁止全部入站的条件下，确认控制与数据两条 WSS 均由节点主动建立；验证方式：防火墙规则截图与连接日志写入 `poc/reports/poc-4.md`
- [ ] 1.6 PoC-5：撤销租约（中继关闭旧连接并拒绝旧 epoch）后，旧客户端无法继续发送 CDP；同时验证额外截图连接对 Browser Use 的干扰；验证方式：`poc/reports/poc-5.md`
- [ ] 1.7 PoC-6：经 Nginx TLS 反向代理测试长 CDP 消息（≥20 MiB）、连续截图、多标签页、ping/pong 与断线重连；验证方式：`poc/reports/poc-6.md`
- [ ] 1.8 汇总 PoC 结论，冻结 Browser Use 与 CfT 精确版本；对于设计受影响的项，通过 `openspec-update-change` 回写本变更；验证方式：`poc/reports/summary.md` 经评审签字

## 2. 仓库与工程脚手架

- [ ] 2.1 把根 `pyproject.toml` 改为 uv workspace，创建 `packages/rbp-protocol`、`packages/rbp-sdk`、`services/control-server`（共享领域层，含 `rbp-api` 与 `rbp-hub` 两个入口）、`services/windows-worker`，删除 `main.py` 与 `test_main.http`；验证方式：`uv sync` 成功，`rbp-api --help` 与 `rbp-hub --help` 均可执行
- [ ] 2.2 配置 ruff、mypy（strict，作用于 protocol/sdk）、pytest、pytest-asyncio 与 pre-commit；验证方式：`uv run ruff check .`、`uv run mypy packages` 与 `uv run pytest` 在空测试集上通过
- [ ] 2.3 建立 CI 工作流：lint、类型检查、单元测试，以及带 PostgreSQL service 的集成测试；Worker 在 Windows runner 上测试；验证方式：CI 在 PR 上全绿
- [ ] 2.4 创建 `web/admin-console`（Vue 3 + Vite + TS + Vue Router + Pinia + Element Plus）脚手架与 lint 配置；验证方式：`pnpm build` 产出静态文件

## 3. 共享协议包 rbp-protocol（WP-01）

- [ ] 3.1 定义控制消息信封模型与全部消息类型的 payload 模型（node-control-protocol 目录中列出的 16 种）；验证方式：每种类型都有序列化往返单元测试
- [ ] 3.2 定义错误码枚举、HTTP 映射与统一错误体模型（api-conventions）；验证方式：单元测试覆盖错误码与状态码的映射表
- [ ] 3.3 定义实例状态机与租约状态机的枚举与合法转移表，并提供 `assert_transition()`；验证方式：参数化测试覆盖每个合法转移和若干非法转移
- [ ] 3.4 实现协议版本协商（同 major 兼容，未知可选字段忽略），并设置控制消息 256 KiB 大小上限；验证方式：单元测试覆盖不兼容 major 与超大消息
- [ ] 3.5 导出 JSON Schema 到 `docs/protocol/` 并编写协议说明文档；验证方式：CI 检查导出的 Schema 与模型一致

## 4. 中央服务基础（WP-04 前置）

- [ ] 4.1 用 pydantic-settings 实现中央配置，各项默认值与 AD §11.3 一致；验证方式：单元测试校验默认值，以及环境变量覆盖生效
- [ ] 4.2 建立 SQLAlchemy async 模型与 Alembic 初始迁移，包含 SRS §7.1 的全部表，以及 design D3 中的部分唯一索引、`next_lease_epoch`/`fenced_epoch`/`generation`、`idempotency_records`、`console_sessions` 与审计哈希列；验证方式：`alembic upgrade head` 与 `downgrade base` 在 CI 的 PG 上通过
- [ ] 4.3 实现统一错误处理中间件、`X-Request-Id` 中间件、structlog JSON 日志与机密脱敏处理器；验证方式：测试确认错误体格式、请求 ID 回传，以及日志中不出现 Authorization 和 ticket
- [ ] 4.4 实现 `Idempotency-Key` 依赖，按主体隔离，冲突时返回 `IDEMPOTENCY_KEY_REUSED`，记录保留 24 小时；验证方式：集成测试覆盖重复请求、请求体不同、跨主体三种情况
- [ ] 4.5 实现游标分页工具与按主体的频率限制（返回 `RATE_LIMITED` 与 `Retry-After`）；验证方式：集成测试
- [ ] 4.6 实现 `/healthz` 与 `/readyz`；验证方式：测试 PG 不可用时 readyz 返回 503
- [ ] 4.7 在 rbp-api 中实现后台任务框架：租约扫描、排队分发、空闲回收、保留期清理，用 PG advisory lock 选主，支持优雅停止；验证方式：单元测试确认任务按周期执行且可停止，两个 api 进程同时运行时任务只执行一次
- [ ] 4.8 实现 PG LISTEN/NOTIFY 事件通道（`instance_changed`、`node_changed`、`lease_changed`，消息体只放 ID），支持断线重连后重新订阅；验证方式：集成测试确认事件送达，断开数据库连接后能恢复订阅
- [ ] 4.9 实现服务间令牌认证与内部路由隔离（`/internal/v1/*` 只监听内网，不经反向代理暴露）；验证方式：测试确认无令牌时返回 401，反向代理配置中不包含 `/internal`

## 5. 鉴权与审计（AUTH / audit-log）

- [ ] 5.1 实现 principals、projects 与 API Key 的创建、哈希存储、前缀查询和吊销，明文只返回一次；验证方式：集成测试确认数据库中没有明文
- [ ] 5.2 实现 Bearer 鉴权依赖、作用域检查与角色检查（取交集），并实现资源级授权辅助函数（所有者、ACL、Project Admin、Platform Admin，无权时返回 404 隐藏）；验证方式：权限矩阵参数化测试
- [ ] 5.3 实现 `rbp-control bootstrap-admin` 命令行工具：已存在管理员时拒绝执行，不提供默认凭据；验证方式：CLI 测试
- [ ] 5.4 实现控制台会话：`POST /auth/session`，HttpOnly/Secure/SameSite Cookie，空闲过期，double-submit CSRF；验证方式：集成测试覆盖过期与 CSRF 拒绝
- [ ] 5.5 实现审计写入服务：只追加，链式哈希，覆盖 audit-log 列出的动作；实现 `GET /audit-events`（含项目范围过滤）；验证方式：集成测试，以及篡改后校验命令报告断链
- [ ] 5.6 实现审计保留期清理与 `audit.purge` 事件；验证方式：测试用伪造时间验证清理范围
- [ ] 5.7 实现凭据吊销或主体禁用时触发的租约撤销钩子；该钩子在第 11 组接入 fencing。验证方式：本组单元测试确认钩子被调用

## 6. 节点注册与控制通道——中央侧（WP-03）

- [ ] 6.1 在 rbp-api 中实现 `POST /nodes/enroll-tokens` 与 `POST /nodes/enroll`：注册码一次性、可设有效期，中央生成 node_id 与凭据；验证方式：集成测试覆盖重复使用、过期、自报 ID 三种情况
- [ ] 6.2 在 rbp-hub 中实现 `/ws/v1/nodes/control` 握手：Bearer 节点凭据认证，HELLO/WELCOME/SNAPSHOT/SNAPSHOT_ACK 顺序，major 版本检查，`connection_epoch` 递增，新连接替换旧连接；验证方式：用模拟节点客户端做集成测试
- [ ] 6.3 在 rbp-hub 中实现节点连接注册表与请求/应答关联（按 `request_id` 等待 future，带 deadline），并暴露内部 RPC `POST /internal/v1/nodes/{id}/commands`；在 rbp-api 中实现对应的 `HubClient`，把 hub 不可达或节点未连接映射为 `NODE_OFFLINE`，超时映射为“结果未知”并进入对账；验证方式：集成测试覆盖成功、超时、hub 停止、节点未连接四种路径
- [ ] 6.4 在 rbp-hub 中实现心跳处理与节点状态机（PENDING/ONLINE/DEGRADED/OFFLINE），采用 10 秒心跳、35 秒失联阈值，并在断线时立即标为 DEGRADED，状态变化时发出 `node_changed`；hub 整体停止时由 rbp-api 依据 `last_seen_at` 兜底判定 OFFLINE；验证方式：用可控时钟测试状态转移时间，以及 hub 停止 35 秒后节点被判定为 OFFLINE
- [ ] 6.5 实现快照对账：按 node-registry“快照对账规则”处理孤儿、丢失与 generation 不一致的实例，结果写入审计；验证方式：表驱动集成测试覆盖三类差异
- [ ] 6.6 实现节点凭据轮换（`ROTATE_CREDENTIAL`，旧凭据在宽限期后失效）与吊销（立即断开连接）；验证方式：集成测试
- [ ] 6.7 实现 Drain/Resume、`PATCH /nodes/{id}/policy`、标签校验与修改；验证方式：集成测试覆盖有效容量 `min()` 计算与非法标签
- [ ] 6.8 实现控制通道故障日志与指标（TLS 失败、认证拒绝、握手超时、心跳超时）；验证方式：测试断言日志事件与计数器递增
- [ ] 6.9 编写 `docs/node-protocol.md`，内容包括握手时序、消息示例与错误码；验证方式：文档示例能通过 rbp-protocol 校验

## 7. Windows Worker——实例管理（WP-02）

- [ ] 7.1 实现 Worker 配置加载（runtime.yaml）与可热更新字段（max_instances、软阈值）的监听；验证方式：单元测试确认重载后新阈值生效
- [ ] 7.2 实现节点级单实例锁（命名互斥体 + 锁文件）与交互式会话检测；验证方式：Windows 测试中第二个进程退出，以 Session 0 运行时报告 `interactive_session=false`
- [ ] 7.3 实现本地 SQLite 模式（instances/profiles/commands/runtime_metadata/fenced_epoch）与迁移；验证方式：单元测试
- [ ] 7.4 实现 Profile 路径解析与安全校验（UUID 字符集、根目录约束、联接点和符号链接解析）；验证方式：参数化测试覆盖 `..`、绝对路径、UNC、设备名与联接点
- [ ] 7.5 实现 InstanceManager 原子容量预留（全部非终态计入占用）、本地 Profile 锁与资源软阈值准入；验证方式：10 个并发创建的测试中占用从不超过上限，内存不足时拒绝创建
- [ ] 7.6 实现 CfT 启动：按 design D7 的参数启动，放入 Job Object，读取 DevToolsActivePort 并用 `/json/version` 复核，校验 loopback 监听，启动超时时清理；验证方式：Windows 集成测试启动 4 个实例且端口互不相同，超时场景有对应测试
- [ ] 7.7 实现实例状态机落地与 generation 递增，非法转移返回 `INVALID_STATE_TRANSITION`；验证方式：单元测试
- [ ] 7.8 实现停止流程：先 `Browser.close`，再强杀进程树，然后确认端口与锁已释放；ephemeral 实例删除目录；验证方式：Windows 测试确认无残留 chrome.exe、端口已释放、目录已删除
- [ ] 7.9 实现 HealthMonitor：区分进程存活与 CDP 可用，处理 UNHEALTHY 与恢复，发送 `INSTANCE_EVENT`；验证方式：通过挂起 Chrome 进程模拟卡死，确认标为 UNHEALTHY
- [ ] 7.10 实现 Chrome 崩溃处理：转为 FAILED，保留诊断信息，再转为 STOPPED，BUSY 实例不自动重启；验证方式：kill chrome.exe 的测试
- [ ] 7.11 实现 Worker 崩溃恢复核验（PID、create_time、exe、cmdline、CDP），并对不符的实例做安全隔离；验证方式：kill Worker 后重启能重新关联实例；PID 被复用的场景用模拟测试
- [ ] 7.12 实现本地诊断 API（仅 127.0.0.1:8765，需本地令牌，单 worker）；验证方式：测试确认监听地址与令牌校验
- [ ] 7.13 实现日志轮转（50 MiB / 14 天），覆盖 Worker 与 Chrome 实例日志；验证方式：单元测试

## 8. Windows Worker——控制客户端与隧道（WP-03 节点侧）

- [ ] 8.1 实现 `rbp-worker enroll --token` 命令：凭据用 DPAPI 加密保存到 `node-credential.bin`；验证方式：Windows 测试确认其他用户无法解密
- [ ] 8.2 实现 ControlClient：主动 WSS 连接，自定义 CA，指数退避加抖动（上限 60 秒），完成握手并发送快照，定期心跳并上报资源指标；验证方式：对中央做集成测试，节点上线后状态为 ONLINE
- [ ] 8.3 实现命令分发、持久化 `message_id` 去重、deadline 拒绝、`connection_epoch` 校验；`CREATE_BROWSER` 以 instance_id 作为幂等键；验证方式：重放与过期命令的测试，以及 Worker 重启后的重放测试
- [ ] 8.4 实现 `GET_BROWSER`（支持 include_targets）与 `STOP_BROWSER` 处理器；验证方式：集成测试
- [ ] 8.5 实现 TunnelManager：校验 `OPEN_TUNNEL`（epoch 大于 fenced_epoch、generation 一致、拒绝任何地址字段），主动连接 `/ws/v1/nodes/tunnel` 并携带 nonce，连接本地 browser WS，双向转发并处理背压；验证方式：集成测试覆盖注入端口被拒、旧 epoch 被拒
- [ ] 8.6 实现 `CLOSE_TUNNEL{fence_epoch}`：先持久化 fenced_epoch，再关闭隧道，然后确认；实现控制连接断开时的自我 fencing；验证方式：测试确认断开控制连接后隧道全部关闭，且在 SNAPSHOT_ACK 之前拒绝 OPEN_TUNNEL
- [ ] 8.7 实现凭据到期前的自动轮换；验证方式：用可控时钟测试
- [ ] 8.8 编写 Windows 安装脚本（创建 browser-runner 账号、目录结构、固定版本 CfT、InteractiveToken 计划任务及失败重启）与 `docs/windows-node-setup.md`；验证方式：在干净 VM 上按文档完成安装并上线

## 9. Profile 管理（PROFILE）

- [ ] 9.1 实现 `POST/GET /profiles` 与 `GET /profiles/{id}`，不返回磁盘路径，无权时返回 404 隐藏；验证方式：集成测试
- [ ] 9.2 实现 Profile ACL 授予与撤销（user/manager），写入审计；验证方式：权限矩阵测试，用户 B 在获得授权前后的行为对比
- [ ] 9.3 实现未指定 Profile 时自动创建 ephemeral Profile，绑定到租约，租约结束后销毁；验证方式：集成测试确认不会被复用
- [ ] 9.4 实现 home node 绑定：首次物化时写入，之后只在该节点调度，节点不可用时返回 `NODE_OFFLINE`，管理员可解绑；验证方式：集成测试
- [ ] 9.5 实现 Profile 删除：要求 confirm 字段，使用中时返回 409，成功后通知 home node 删除目录并写审计；验证方式：集成测试

## 10. 调度与排队（SCHED）

- [ ] 10.1 实现 acquire 同步路径：授权 → 节点过滤 → 复用 READY → 按选择策略在事务中预留并新建（`FOR UPDATE` 节点行）→ 下发 CREATE 并等待；实现 `reuse=false` 语义；验证方式：集成测试，有空闲实例时不多建，新建时返回 201，复用时返回 200
- [ ] 10.2 实现节点拒绝后换节点与预留回收，home node 绑定的 Profile 不换节点；验证方式：集成测试中 3 个节点有 1 个拒绝
- [ ] 10.3 实现 CREATE 超时后用 GET_BROWSER 对账，而不是直接重建；验证方式：在模拟节点上注入延迟的测试
- [ ] 10.4 实现空闲实例超时回收（idle_instance_ttl_seconds）；验证方式：可控时钟测试，并检查审计 `instance.idle_stop`
- [ ] 10.5 实现 LRU 驱逐：只驱逐空闲且满足最小空闲时长的实例，先预留再停止，确认 STOPPED 后才创建；验证方式：集成测试覆盖可驱逐与不可驱逐两种情况
- [ ] 10.6 实现异步排队：返回 202，`GET/DELETE /acquisition-requests/{id}`，QueueDispatcher 事件唤醒与 FIFO，超时与取消时回收预留（创建中被取消时按模式处理）；验证方式：集成测试确认不留孤儿实例或租约
- [ ] 10.7 实现按主体和按项目的并发配额（`QUOTA_EXCEEDED`）；验证方式：集成测试
- [ ] 10.8 实现失败原因 details（all_nodes_full 等 6 类）；验证方式：每类原因各一条测试
- [ ] 10.9 编写调度并发测试：20 个并发 acquire 竞争有限容量，断言不超配，每个 Profile 只有一个实例；验证方式：测试通过

## 11. 独占租约与 fencing（LEASE）

- [ ] 11.1 实现租约授予：`lease_epoch` 取自实例的 `next_lease_epoch++`，用部分唯一索引保证唯一；验证方式：并发授予测试中只有一个 ACTIVE
- [ ] 11.2 实现 `GET /leases/{id}`、`POST /leases/{id}/renew`（校验 epoch，GRACE 时可恢复为 ACTIVE）；验证方式：集成测试
- [ ] 11.3 实现租约扫描器：ACTIVE→GRACE→REVOKING，使用条件 UPDATE；验证方式：可控时钟测试，并确认 GRACE 期间他人无法获得该实例
- [ ] 11.4 实现释放流程：DELETE 幂等 → RELEASING → rbp-api 调用 hub 内部 RPC `POST /internal/v1/leases/{id}/close`（关闭客户端连接，下发 `CLOSE_TUNNEL`，等待节点确认）→ RELEASED → 实例回到 READY 或被停止；验证方式：集成测试
- [ ] 11.4a 在 rbp-hub 中实现租约兜底收敛：订阅 `lease_changed`，并每 30 秒全量核对活动隧道与租约状态，关闭不再有效的连接后回报 fencing 结果；验证方式：故意丢弃 RPC 的测试中，撤销仍在 30 秒内完成
- [ ] 11.5 实现撤销流程与 `POST /leases/{id}/revoke`（reason 必填，写审计）；节点不可达时停留在 REVOKING，节点重连对账后完成；接入第 5.7 项的吊销钩子；验证方式：集成测试覆盖节点断开时撤销
- [ ] 11.6 实现管理员“强制判定 fencing”操作（高权限、必须填写原因、写审计）；验证方式：集成测试
- [ ] 11.7 实现实例故障联动：实例 FAILED 时租约转为 FAILED，并关闭隧道；验证方式：集成测试
- [ ] 11.8 编写不变量 I1 的属性测试：随机交错执行 acquire、renew、release、revoke、节点断线、CDP 重连，断言同一实例从不同时存在两个有效租约或两条活动写隧道；验证方式：hypothesis 测试通过

## 12. Secure CDP Gateway（CDP）

- [ ] 12.1 实现 `POST /leases/{id}/connect-ticket`：只对 ACTIVE/GRACE 租约签发，只存哈希，有效期 60 秒，绑定 epoch；验证方式：集成测试
- [ ] 12.2 在 rbp-hub 中实现 `/cdp/connect` 的 Upgrade：原子消费票据，校验租约 epoch，失败时关闭码 4401/4403；验证方式：测试覆盖重放、过期、错误租约
- [ ] 12.3 实现隧道协调：生成 tunnel_id 与 nonce，下发 OPEN_TUNNEL，配对时等待 8 秒，超时以 4504 关闭并发送 CLOSE_TUNNEL；验证方式：集成测试覆盖超时清理
- [ ] 12.4 实现 `/ws/v1/nodes/tunnel` 节点接入与配对校验（节点身份、tunnel、nonce 一次性、实例归属、epoch）；验证方式：测试覆盖冒用节点与 nonce 重放
- [ ] 12.5 实现透明双向转发：保持帧顺序与类型，帧上限 32 MiB（超限 1009），有界缓冲与背压，ping/pong 每 20 秒；验证方式：用模拟 CDP 回显服务测试 20 MiB 消息和背压
- [ ] 12.6 实现每租约单写连接替换（4409），以及撤销、失联、吊销时强制关闭（4403/4410）；验证方式：集成测试确认任意时刻只有一条活动连接
- [ ] 12.7 实现 tunnels 表的元数据与字节计数，中央启动时把残留记录标为 server_restart；验证方式：集成测试
- [ ] 12.8 编写反向代理配置（Nginx 与 Traefik 示例：`/api` 路由到 rbp-api，`/ws` 与 `/cdp` 路由到 rbp-hub，Upgrade、超时、query 脱敏）与 `docs/gateway.md`；验证方式：在 compose 环境中经反向代理完成 CDP 连接
- [ ] 12.9 用真实 Browser Use（PoC 冻结的版本）跑端到端测试：多标签页、auto-attach、导航、点击、截图；验证方式：在 Windows 节点上 e2e 测试通过
- [ ] 12.10 Playwright 兼容性用例：用 Playwright Python 的 `chromium.connect_over_cdp(cdp_url)`（固定精确版本）经网关连接，完成新建页面、导航、点击、读取文本、截图、关闭页面，断开后 Chrome 实例仍为 READY；验证方式：`tests/compat/test_playwright_cdp.py` 在 Windows 节点 e2e 中通过

## 13. 截图（MEDIA）

- [ ] 13.1 实现 `GET /instances/{id}/targets`（经 GET_BROWSER include_targets，带授权检查）；验证方式：集成测试
- [ ] 13.2 实现截图请求 API、captures 记录与一次性上传令牌，同步等待 15 秒，超时返回 202；验证方式：集成测试
- [ ] 13.3 实现 Worker 端 SCREENSHOT 处理：使用独立本地 CDP session，attach → capture → detach，不调用 bringToFront，超时 10 秒，整页截图按高度截断，通过 HTTPS PUT 上传；验证方式：Windows 测试确认截图前后页面状态不变
- [ ] 13.4 实现 `PUT /internal/captures/{id}/upload`：令牌单次有效、大小上限、sha256 校验；`CaptureStore` 本地卷实现，使用随机对象键；验证方式：集成测试覆盖令牌重用与哈希不一致
- [ ] 13.5 实现 `GET /captures/{id}`、`GET /captures/{id}/content`、`DELETE /captures/{id}`，带授权与审计，7 天保留期清理；验证方式：集成测试
- [ ] 13.6 编写测试：20 张并行整页截图期间心跳不超时；验证方式：集成测试

## 14. 管理 API（ADMIN）

- [ ] 14.1 实现节点管理 API 全集（列表/详情字段、凭据吊销、标签修改）；验证方式：集成测试确认列表与详情一致
- [ ] 14.2 实现实例管理 API（列表/详情、restart、DELETE 受租约保护）；验证方式：集成测试覆盖 `LEASE_CONFLICT`
- [ ] 14.3 实现租约列表过滤、排队请求列表与管理员取消；验证方式：集成测试
- [ ] 14.4 实现主体、项目、API Key 与 ACL 管理 API；验证方式：集成测试
- [ ] 14.5 生成并提交 OpenAPI 文档，CI 校验错误响应码与错误码目录一致；验证方式：CI 检查通过

## 15. 管理控制台 SPA（admin-console）

- [ ] 15.1 实现 OpenAPI 生成的 TS 客户端、登录页与会话管理（含 CSRF 头），以及按角色控制路由守卫与菜单；验证方式：Vitest 单测，Operator 登录后看不到管理菜单
- [ ] 15.2 实现节点列表与详情页（Drain/Resume、策略、标签），带 10 秒自动刷新；验证方式：Playwright e2e 中节点状态变化在 10 秒内可见
- [ ] 15.3 实现实例列表与详情页（目标列表、发起截图、截图历史、重启/停止）；验证方式：e2e
- [ ] 15.4 实现租约列表（撤销需填原因并二次确认）、本人租约自助释放、排队请求页；验证方式：e2e 中未填原因时按钮不可用
- [ ] 15.5 实现 Profile 与 ACL、主体与 API Key（明文只展示一次）、审计事件筛选分页页面；验证方式：e2e
- [ ] 15.6 配置反向代理在 `/console/` 下同源提供静态文件，编写 `docs/console.md`；验证方式：在 compose 环境中访问控制台并完成一次 Drain 操作

## 16. Client SDK（WP-06）

- [ ] 16.1 实现 SDK 配置（参数/环境变量、自定义 CA、日志中不出现 Key）与类型化异常映射；验证方式：单元测试
- [ ] 16.2 实现 `acquire`：自动生成并复用幂等键，202 时透明轮询，`wait_timeout` 处理；验证方式：用 respx mock 测试网络重试只产生一个租约
- [ ] 16.3 实现会话对象：`cdp_url()` 每次取新票据，后台续租（带退避、截止 grace_until），撤销回调与状态属性；同步与 async 双接口；验证方式：单元测试与对中央的集成测试
- [ ] 16.4 实现上下文管理器释放（不掩盖原异常）、atexit 时尽力释放、`list_targets()` 与 `screenshot()`；验证方式：单元测试
- [ ] 16.5 编写 Browser Use 集成示例 `packages/rbp-sdk/examples/browser_use_remote.py`（固定版本，保持浏览器存活）与 SDK README；验证方式：示例分别在开发机和 CI 主机上成功运行

## 17. 可观测性与运维（OPS）

- [ ] 17.1 实现 Prometheus 指标全集（platform-observability 列出的指标），rbp-api 与 rbp-hub 各自通过受保护的 `/metrics` 暴露，另外统计内部 RPC 的耗时与失败数；验证方式：测试断言指标存在，未授权时返回 401
- [ ] 17.2 Worker 心跳汇总资源与实例指标，中央将其转为节点维度的指标；验证方式：集成测试
- [ ] 17.3 编写 Prometheus 告警规则文件（7 类告警）并用 promtool 测试；验证方式：`promtool test rules` 通过
- [ ] 17.4 实现数据保留清理任务（租约/隧道 30 天，截图 7 天，审计 90 天，均可配置）；验证方式：可控时钟测试
- [ ] 17.5 编写 PG 备份恢复脚本与演练文档、Worker/CfT 升级回滚文档（不覆盖 Profile）；验证方式：按文档完成一次恢复演练与一次 CfT 回滚

## 18. 中央部署

- [ ] 18.1 编写 control-server Dockerfile（同一镜像，两个启动命令）与 `deploy/docker-compose.yml`，包括 rbp-api、rbp-hub、PG 持久卷、反向代理 TLS、截图卷、迁移 init 步骤，内部网络只供 api 与 hub 互通；验证方式：`docker compose up` 后 api 与 hub 的 `/readyz` 均返回 200
- [ ] 18.2 编写 `docs/deployment.md`，涵盖证书与自签 CA 分发、bootstrap-admin、首个节点注册的端到端步骤；验证方式：新人按文档从零部署成功

## 19. 集成验证（WP-09：UAT、压测、故障注入、安全回归）

- [ ] 19.1 自动化执行 UAT-001 至 UAT-026，在 Windows 节点环境中运行，每个用例生成报告；验证方式：`tests/uat` 全部通过并归档报告
- [ ] 19.2 故障注入：单独重启 rbp-api（断言 CDP 会话不中断）、单独重启 rbp-hub、两者同时重启、api 与 hub 之间 RPC 丢包、NOTIFY 丢失、节点断网、Worker 被杀、Chrome 被杀、Windows 注销，每种场景都断言不变量 I1–I4；验证方式：故障注入报告
- [ ] 19.3 单节点 1/2/4 Chrome 逐级压测（复杂页面、登录页面、图片、多标签页、连续截图），记录 VM 资源占用；验证方式：压测报告
- [ ] 19.4 6 节点 × 4 Chrome、24 并发、持续 60 分钟的压测，统计成功率、网关开销、配对耗时、截图 P95、断开率；验证方式：压测报告满足 NFR-001/005
- [ ] 19.5 安全回归：票据重放、越权访问 Profile/截图/租约、节点冒用、路径穿越、日志机密抽查、远程端口扫描；验证方式：安全测试报告，无严重问题
- [ ] 19.6 按 SRS §12.2 逐项核对质量门禁与 §14 DoD，形成发布检查单；验证方式：检查单经评审签字

## Workflow follow-up

- 如果 PoC 结论改变设计，先用 `openspec-update-change` 修订本变更，再继续第 2 组之后的任务。
- V1 发布并通过评审后，归档本变更（`openspec-archive-change`），使 14 个能力 spec 进入 `openspec/specs/`，作为 V2 变更的基线。
