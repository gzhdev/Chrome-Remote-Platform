# Spec Delta: browser-instance-runtime

## Purpose

规定 Windows Worker 如何在交互式登录会话中管理 Chrome for Testing 实例：启动环境、实例隔离、本地容量、CDP 端口发现与 loopback 约束、统一状态机、停止与清理、健康探测以及 Worker 崩溃后的安全恢复。

## ADDED Requirements

### Requirement: 交互式会话可见浏览器
Worker SHALL 运行于专用低权限 Windows 账号的交互式登录会话（由任务计划程序在用户登录后启动），以可见窗口（非 headless）启动固定版本的 Chrome for Testing。Worker MUST 检测自身是否处于交互式会话，否则拒绝启动实例并上报原因。

#### Scenario: RDP 可见
- **WHEN** 通过 RDP 登录节点并创建实例
- **THEN** 桌面上可见对应 Chrome 窗口，且可经 CDP 进行网页操作

#### Scenario: RDP 断开后仍可用
- **WHEN** 关闭 RDP 窗口（断开而非注销）后经 CDP 导航与截图
- **THEN** 操作成功，Chrome 进程存活

#### Scenario: 非交互式会话
- **WHEN** Worker 以 Session 0 服务方式运行
- **THEN** Worker 拒绝创建实例并在 `NODE_HELLO` 中标记 `interactive_session=false`

### Requirement: 单写进程
每个节点 MUST 只有一个 Worker 进程拥有实例管理写权限。Worker 启动 SHALL 获取节点级本地独占锁，获取失败 MUST 立即退出；Worker 的本地 HTTP 服务 MUST 以单进程运行。

#### Scenario: 启动第二个 Worker
- **WHEN** 已有 Worker 运行时再启动一个 Worker
- **THEN** 第二个进程因锁获取失败退出，第一个不受影响

### Requirement: 实例隔离
每个实例 SHALL 使用独立的 `--user-data-dir`（对应 Profile 目录）、独立下载目录 `downloads\<instance_id>\`、独立日志文件，并记录其主进程 PID、进程启动时间与可执行路径。不同实例的 Cookie、LocalStorage、进程树 MUST NOT 混用。

#### Scenario: 两实例 Cookie 隔离
- **WHEN** 实例 A 登录站点 S，实例 B 访问站点 S
- **THEN** 实例 B 不具有 A 的登录 Cookie

### Requirement: 本地容量上限
Worker SHALL 从本地配置读取 `max_instances`（默认 4），配置变更 SHALL 支持重载生效。Worker SHALL 将 `CREATING`、`STARTING`、`READY`、`BUSY`、`UNHEALTHY`、`STOPPING` 状态的实例全部计入占用；只有确认进程退出且 Profile 锁释放后才释放名额。超额创建 MUST 以 `NODE_CAPACITY_REACHED` 拒绝。

#### Scenario: 第五个实例被拒
- **WHEN** 节点已占用 4 个名额时收到第 5 个 `CREATE_BROWSER`
- **THEN** Worker 回复 `NODE_CAPACITY_REACHED`，不启动 Chrome

#### Scenario: 下调上限不强杀
- **WHEN** 运行 3 个实例时将 `max_instances` 由 4 改为 2 并重载
- **THEN** 现有 3 个实例继续运行，新的创建请求被拒直至占用低于 2

### Requirement: 原子容量预留
容量检查与预留 SHALL 在单一临界区中原子完成。

#### Scenario: 并发创建
- **WHEN** 10 个 `CREATE_BROWSER` 并发到达 `max_instances=4` 的节点
- **THEN** 同时处于占用状态的实例数从不超过 4，其余请求得到 `NODE_CAPACITY_REACHED`

### Requirement: 资源软阈值准入
Worker SHALL 在创建前检查 CPU 使用率（默认软上限 85%）与可用内存（默认最低 2048 MB），越界时 MUST 拒绝创建并返回 `RESOURCE_EXHAUSTED` 及具体原因，即使未达 `max_instances`。

#### Scenario: 内存压力
- **WHEN** 可用内存低于 2048 MB 时收到创建请求
- **THEN** Worker 拒绝创建，`details.reason` 为 `low_memory`

### Requirement: 动态 CDP 端口发现
Worker SHALL 以 `--remote-debugging-port=0` 与 `--remote-debugging-address=127.0.0.1` 启动 Chrome，从 Profile 目录的 `DevToolsActivePort` 读取端口与浏览器 WS 路径，并通过 `http://127.0.0.1:<port>/json/version` 复核 `webSocketDebuggerUrl` 一致。启动发现超时（默认 20 秒）MUST 终止该进程树并返回 `CHROME_START_TIMEOUT`。

#### Scenario: 多实例无端口冲突
- **WHEN** 同一节点连续启动 4 个实例
- **THEN** 每个实例获得不同端口，全部复核成功

#### Scenario: 发现超时
- **WHEN** Chrome 在 20 秒内未写出 `DevToolsActivePort`
- **THEN** Worker 结束该进程树、释放名额与 Profile 锁，回复 `CHROME_START_TIMEOUT`

### Requirement: CDP 仅限 loopback
实例的 CDP 端口 MUST 只监听 loopback。Worker 启动后 SHALL 校验实际监听地址，发现非 loopback 监听 MUST 立即终止该实例并标为 `FAILED`。Worker MUST NOT 向中央或客户端返回原始 CDP URL、端口或 browser GUID。

#### Scenario: 远程无法直连
- **WHEN** 从其他主机扫描节点上 Chrome 的调试端口
- **THEN** 连接失败

### Requirement: 实例状态机
实例 SHALL 遵循状态：`CREATING`、`STARTING`、`READY`、`BUSY`、`UNHEALTHY`、`STOPPING`、`STOPPED`、`FAILED`。允许转移为：CREATING→STARTING|FAILED；STARTING→READY|FAILED；READY→BUSY|STOPPING|UNHEALTHY；BUSY→READY|UNHEALTHY|STOPPING（仅限租约撤销后）；UNHEALTHY→READY|FAILED|STOPPING；STOPPING→STOPPED；FAILED→STOPPED。其他转移 MUST 以 `INVALID_STATE_TRANSITION` 拒绝。每次 Chrome 启动 SHALL 分配递增的 `generation`。

#### Scenario: 非法转移
- **WHEN** 对 `STOPPED` 实例请求进入 `BUSY`
- **THEN** 返回 `INVALID_STATE_TRANSITION`，状态不变

#### Scenario: 状态可查询
- **WHEN** 中央发送 `GET_BROWSER`
- **THEN** Worker 返回当前状态、`generation`、最近状态变更时间与最近错误

### Requirement: 停止与清理
`STOP_BROWSER` SHALL 先尝试经 CDP `Browser.close` 优雅关闭，超过宽限（默认 10 秒）后强制结束整个进程树。只有确认进程树全部退出、CDP 端口不再监听且 Profile 锁释放后，实例 SHALL 进入 `STOPPED` 并释放名额。命令回执本身 MUST NOT 被视为完成依据。

#### Scenario: 强杀无孤儿
- **WHEN** Chrome 未响应优雅关闭被强制终止
- **THEN** 无残留 chrome.exe 子进程，原端口不再监听，名额被释放

### Requirement: 临时 Profile 实例清理
ephemeral 模式的实例停止后 SHALL 删除其 Profile 目录与下载目录。

#### Scenario: 临时环境无残留
- **WHEN** 一个 ephemeral 实例被停止
- **THEN** 其 user-data-dir 与下载目录被删除，后续新实例不含其 Cookie

### Requirement: 健康探测
Worker SHALL 周期性（默认 15 秒）分别探测进程存活与 CDP 可用性（`Browser.getVersion` 在 5 秒内响应）。进程存活但 CDP 无响应 MUST 将实例标为 `UNHEALTHY` 而非 `READY`。健康状态变化 SHALL 通过 `INSTANCE_EVENT` 上报并留审计。

#### Scenario: Chrome 卡死
- **WHEN** Chrome 进程存活但 CDP 连续 2 次探测超时
- **THEN** 实例进入 `UNHEALTHY`，中央不再将其分配给新租约

#### Scenario: 空闲实例恢复
- **WHEN** `UNHEALTHY` 的空闲实例后续探测恢复正常
- **THEN** 实例回到 `READY` 并记录健康恢复事件

### Requirement: Chrome 崩溃处理
Chrome 意外退出时，Worker SHALL 将实例标为 `FAILED`、保留诊断信息（退出码、最近日志片段路径）、清理后转为 `STOPPED`。处于 `BUSY` 的实例 MUST NOT 被自动重启；空闲实例 MAY 按配置策略重新创建。

#### Scenario: Kill Chrome
- **WHEN** 手动结束一个 BUSY 实例的 chrome.exe
- **THEN** 实例变为 `FAILED`，相关租约失败，系统不静默复用该实例

### Requirement: Worker 崩溃恢复
Worker 重启 SHALL 读取本地记录并逐一核验存活实例：PID 存在、进程启动时间一致、可执行路径为配置的 CfT、命令行包含该实例的 `--user-data-dir`、CDP 探测成功。全部通过的 SHALL 重新关联；任一不符的 SHALL 安全隔离（终止匹配到的进程树或标为 `FAILED`），MUST NOT 盲目重启或重复占用 Profile。

#### Scenario: Worker 被杀，Chrome 存活
- **WHEN** 结束 Worker 进程后重新启动 Worker
- **THEN** 存活 Chrome 被重新关联为原 `instance_id`，未产生新的 Chrome 进程

#### Scenario: PID 被复用
- **WHEN** 记录中的 PID 已被其他程序复用
- **THEN** 核验因启动时间或命令行不符失败，Worker 不对该进程执行任何操作，实例标为 `FAILED`

### Requirement: Windows 会话丢失
Windows 注销或重启后，实例 MUST 被视为全部不可用；Worker 在下次登录启动时 SHALL 将残留记录标为 `STOPPED` 并在快照中如实上报。

#### Scenario: 重启后上线
- **WHEN** VM 重启并由人工重新登录
- **THEN** Worker 上线时快照中无运行实例，中央将原实例影子标为 `STOPPED`，持久化 Profile 数据保留

### Requirement: 本地管理接口仅 loopback
Worker 本地 FastAPI 诊断接口 SHALL 只绑定 `127.0.0.1`（默认端口 8765），且需本地令牌访问；中央 MUST NOT 依赖该接口。

#### Scenario: 远程访问本地接口
- **WHEN** 从其他主机访问节点 8765 端口
- **THEN** 连接失败
