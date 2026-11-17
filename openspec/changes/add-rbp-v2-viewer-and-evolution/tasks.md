# Tasks: RBP V1.1 / V2 演进

> 前置：`add-rbp-v1-platform` 已归档。阶段 A–D 可分别发布。

## 0. 基线对齐

- [ ] 0.1 V1 归档后，对照 `openspec/specs/` 为 `browser-leases`（holder_kind、接管转移）、`node-control-protocol`（screencast/artifact/input 消息）、`cdp-gateway`、`admin-console`、`client-sdk` 补写 MODIFIED delta；验证方式：`openspec validate add-rbp-v2-viewer-and-evolution --strict` 通过
- [ ] 0.2 PoC：测试后台标签页的 screencast 行为、screencast 与 Browser Use 同时操作同一 Target 是否互相干扰、`DOM.setFileInputFiles` 在远程 CDP 下的效果、Windows DPI 缩放下的输入坐标；验证方式：`poc/reports/v2-poc.md`
- [ ] 0.3 PoC：Profile 快照恢复后的登录状态保留情况，分三种环境测试（同 VM 同用户、同 VM 重建用户、不同 VM），并记录快照典型体积与本地暂存耗时；验证方式：`poc/reports/v2-profile-backup.md`

## 1. 阶段 A：浏览器工件（V1.1）

- [ ] 1.0 在 compose 中加入 MinIO（持久卷、SSE、三个桶及生命周期规则），反向代理开放预签名专用路径 `/objects/*`，并编写 MinIO 运维与备份文档；验证方式：`docker compose up` 后桶已创建，预签名 PUT/GET 经反向代理可用
- [ ] 1.1 实现 `ObjectStore` 抽象（本地卷实现与 MinIO 实现，支持预签名 PUT/GET），提供截图存储迁移脚本；验证方式：两种实现都通过同一套契约测试，迁移脚本可回滚
- [ ] 1.2 实现 `artifacts` 表迁移与上传 API（大小上限 413，授权检查）；验证方式：集成测试
- [ ] 1.3 接入 clamd 异步扫描，管理 `QUARANTINED`、`AVAILABLE` 状态并写审计；验证方式：上传 EICAR 文件后被隔离
- [ ] 1.4 实现投递命令：节点用一次性令牌拉取文件，写入节点生成的路径，执行 `DOM.setFileInputFiles`，并拒绝客户端指定路径；验证方式：Windows e2e 测试中页面文件输入获得该文件
- [ ] 1.5 实现下载检测（结合 downloadProgress 事件与目录监听），完成后上传归档并清理节点目录；验证方式：e2e 测试页面下载 CSV 后可取回，节点目录被清空
- [ ] 1.6 SDK 增加 `upload_file()`、`set_file_input()`、`list_downloads()`、`download()`，并补充 Browser Use 文件上传示例；验证方式：示例运行通过
- [ ] 1.7 实现工件保留期清理，控制台增加工件浏览页；验证方式：可控时钟测试，以及控制台 e2e

## 1b. 阶段 A：Profile 快照备份（V1.1，profile-backup）

- [ ] 1b.1 新增 `profile_snapshots` 表迁移，协议增加 `SNAPSHOT_PROFILE`/`RESTORE_PROFILE` 消息与节点能力声明（minor 升级）；验证方式：迁移的 upgrade/downgrade 通过，消息序列化往返测试通过
- [ ] 1b.2 Worker 实现变化指纹计算与快照流程：持锁期间 robocopy 到 staging（排除缓存目录）后立即释放锁，然后用 zstd 打包、计算 sha256、经预签名 PUT 上传，最后清理 staging；磁盘不足时跳过并上报；验证方式：Windows 测试确认持锁时间小于 30 秒，期间到达的 acquire 能成功，快照中不含缓存目录
- [ ] 1b.3 rbp-api 实现自动触发（实例 STOPPED、目录有变化、距上次快照 ≥6 小时）与手动触发 API（使用中返回 409），完成 MinIO 对象哈希校验与状态流转；验证方式：集成测试覆盖自动触发、手动触发与哈希不一致三种情况
- [ ] 1b.4 实现快照查询 API（带权限检查，无权时返回 404）与保留策略（3 份 / 30 天，Profile 删除时连带删除），写审计；验证方式：可控时钟测试
- [ ] 1b.5 实现恢复：同 home node 恢复需 `confirm`；跨环境恢复需 Platform Admin 权限加 `accept_credential_loss`，成功后迁移 home node；Worker 端执行“下载、校验、解压到 `.restore`、rename 交换、首次加载成功后删除 `.bak`”，失败时回退；验证方式：Windows e2e 测试中同节点恢复后登录状态保留，磁盘写满场景回退到原目录
- [ ] 1b.6 增加快照体积、耗时、失败数指标与快照失败告警；控制台的 Profile 详情页增加快照列表、手动快照、恢复入口（跨环境恢复时显示风险确认）；验证方式：promtool 测试与控制台 e2e
- [ ] 1b.7 编写 `docs/profile-backup.md`，说明 DPAPI 限制、恢复步骤与容量估算；验证方式：按文档完成一次同节点恢复演练

## 2. 阶段 B：页面回显与观察者（V2-a）

- [ ] 2.1 协议升级到 1.1，新增 `START_SCREENCAST`/`STOP_SCREENCAST` 消息与节点能力声明；验证方式：新旧 Worker 的兼容性测试
- [ ] 2.2 实现 `viewer_grants` 与 viewer token（一次性，与 CDP 票据入口隔离），撤销后 5 秒内关闭观看流；验证方式：集成测试，并确认 viewer token 无法连接 `/cdp/connect`
- [ ] 2.3 实现 Worker 端 screencast 采集：使用独立 session，startScreencast 并逐帧 ack，限帧率，主动连接 `/ws/v1/nodes/screencast`；验证方式：Windows 测试确认 Agent 会话不受影响
- [ ] 2.4 实现 Viewer Gateway：一个 Target 一路上行，扇出给多个观察者，每个观察者使用最新帧单槽缓冲，上行消息白名单（违规时以 1008 关闭）；验证方式：慢观察者测试中内存不增长，注入尝试导致连接关闭
- [ ] 2.5 实现观看审计与 `GET /leases/{id}/viewers`；验证方式：集成测试
- [ ] 2.6 控制台增加页面回显播放器与观察者管理；验证方式：e2e
- [ ] 2.7 编写带宽指标与告警规则；验证方式：promtool 测试

## 3. 阶段 C：人工接管（V2-b）

- [ ] 3.1 租约模型增加 `holder_kind` 与 `parent_lease_id`，实现接管与交还的状态转移（经 fencing）；验证方式：状态机单元测试
- [ ] 3.2 实现接管与交还 API，并写审计；验证方式：集成测试
- [ ] 3.3 实现人工输入 WSS 与节点 `Input.*` 执行，坐标按帧元数据换算；验证方式：Windows e2e 测试在 100%/125%/150% 缩放下点击准确
- [ ] 3.4 控制台播放器增加接管模式（获取输入焦点、交还、释放）；验证方式：e2e
- [ ] 3.5 编写单控制方属性测试：在接管与交还过程中随机注入断线，断言 Agent 与人工从不同时具备控制能力；验证方式：hypothesis 测试通过

## 4. 阶段 D：多 Gateway 扩容

- [ ] 4.1 实现 `node_connections` 归属表与抢占，旧 hub 副本的连接失效；验证方式：双 hub 副本集成测试
- [ ] 4.2 rbp-api 的 `HubClient` 改为按归属表路由内部 RPC 到对应 hub 副本，归属副本不可达时回退到对账；验证方式：跨副本创建实例的测试
- [ ] 4.3 实现带副本前缀的隧道路由（`OPEN_TUNNEL` 携带 hub 副本 URL，反向代理按前缀路由）；验证方式：客户端与节点落在不同 hub 副本时配对成功
- [ ] 4.4 把 rbp-api 扩到多副本（无状态，后台任务沿用 V1 的 advisory lock）并做验证；验证方式：3 个 api 副本加 2 个 hub 副本时，任务只执行一次且请求均可处理
- [ ] 4.5 实现副本 Drain 与滚动升级流程，并编写文档；验证方式：滚动升级演练中节点始终可调度
- [ ] 4.6 编写多副本下 I1–I3 的故障注入与压测；验证方式：报告确认不变量成立

## Workflow follow-up

- 每个阶段发布并评审后，可以把已完成阶段拆出来单独归档；也可以等全部完成后一次性归档本变更。
