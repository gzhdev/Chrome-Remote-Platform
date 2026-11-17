# Proposal: RBP V1 之后的演进（V1.1 工件传输 + V2 页面回显、观察者、人工接管、多 Gateway）

## Why

V1（变更 `add-rbp-v1-platform`）只提供单控制租约下的 CDP 控制与静态截图。测试人员还需要三类能力：实时看到远程页面、在 Agent 卡住时安全地人工接管、在 Agent 机器和 Windows 浏览器之间交换上传与下载文件。另外，并发规模超过单 Gateway 承载能力后，需要多副本扩容。依据 AD §9、§13 与 SRS §3 的版本划分，这些能力分别属于 V1.1 和 V2。本变更在 V1 基线之上定义它们。

## What Changes

**V1.1（先行交付）**
- 新增受控**浏览器工件服务**：客户端上传文件，供远程浏览器的文件输入使用；浏览器下载产物在中央归档后，由客户端授权取回。包含大小、类型、路径、权限、恶意内容扫描和保留策略。
- 专项验证 Browser Use 文件上传 API 在远程 CDP 场景下的行为，并在 SDK 中提供对应封装。
- 新增**持久化 Profile 快照备份与恢复**：Profile 不在使用时，把目录快照到 MinIO，带保留策略和授权恢复，降低节点磁盘故障、误删或 Chrome 升级失败导致的登录状态丢失风险。同节点、同 Windows 用户下的恢复是主要场景。跨节点恢复受 DPAPI 限制，登录状态可能丢失，所以只作为需要显式确认的高权限操作提供。

**V2**
- 新增**页面实时回显（Screencast）**：基于 CDP `Page.startScreencast` 或经验证的替代方案，由节点编码后转发到独立的 Viewer Gateway，按 Target 订阅，支持限帧率、背压与隐私提示。与 Agent 控制通道隔离。
- 新增**观察者授权**：独立的 viewer grant 与 viewer token，只能接收页面帧，不能注入任何 CDP 或控制事件；支持多个观察者同时观看。
- 新增**人工接管**：通过租约 fencing 把控制权从 Agent 转给人工操作者，旧 Agent 被隔离后才开放人工输入（页面级鼠标键盘事件），结束后可以交还或释放。
- 新增**多 Gateway 水平扩容**：解决节点控制连接归属、隧道路由、跨副本消息、数据库锁与 Drain 迁移问题，使中央服务可以多副本运行。
- 管理控制台增强：页面回显播放器、观察者管理、接管操作、工件浏览。

## Capabilities

### New Capabilities

- `browser-artifacts`：上传与下载工件的存储、授权、校验、扫描与保留，以及工件与远程浏览器文件系统之间的受控交换。
- `profile-backup`：持久化 Profile 的快照触发、MinIO 存储、保留、授权恢复，以及跨环境恢复的限制。
- `page-screencast`：页面帧的采集、编码、订阅、限流与传输。
- `viewer-access`：观察者授权模型、viewer token，以及“只读、不可注入”的保证。
- `human-takeover`：控制权在 Agent 与人工之间转交的流程、fencing 与人工输入通道。
- `gateway-scale-out`：多副本中央服务下的连接归属、路由与一致性保证。

### Modified Capabilities

（暂无。本变更以 V1 变更归档后的 spec 为基线。V1 归档后，如果需要修改 `cdp-gateway`、`browser-leases`、`browser-screenshots`、`admin-console`、`client-sdk`、`node-control-protocol` 中的既有需求，应在实施前以 MODIFIED delta 补充到本变更。预计涉及：租约新增 `TAKEOVER` 相关状态，控制协议新增 screencast、artifact、input 消息类型。）

## Impact

- **前置依赖**：变更 `add-rbp-v1-platform` 必须已实施并归档。
- **代码**：control-server 新增 artifact、profile-backup、viewer、takeover 模块；Viewer Gateway 并入 V1 已有的长连接面 rbp-hub，也可以单独部署；windows-worker 新增 screencast 编码、文件交换、Profile 快照与恢复、输入注入；rbp-protocol 新增消息类型并升级 minor 版本；SDK 新增工件 API；admin-console 新增播放器与接管页面。
- **依赖**：对象存储使用 MinIO（S3 兼容），工件、截图与 Profile 快照共用；恶意内容扫描引擎（例如 ClamAV）。跨副本消息总线复用 V1 已有的 PostgreSQL LISTEN/NOTIFY。
- **容量**：Profile 快照按每个 Profile 数十到数百 MiB、保留 3 份估算 MinIO 容量。Windows 节点需要预留快照暂存空间。
- **带宽**：页面回显会显著增加 WAN 带宽占用，需要容量评估。
