# Design: RBP V1.1 / V2 演进

## Context

- 前置条件：`add-rbp-v1-platform` 已实施并归档，V1 的 14 个能力 spec 是本变更的基线。V1 中央服务已拆为无状态的 REST 面 rbp-api 与有状态的长连接面 rbp-hub（单实例），两者通过内部 RPC 与 PG LISTEN/NOTIFY 协作，后台任务已用 advisory lock 选主（V1 design D2）。截图使用 `CaptureStore` 抽象和本地卷（V1 design D9）。
- 动机见 proposal.md。具体需求见本变更 `specs/` 下的 5 个能力。

## Goals / Non-Goals

**Goals:**
- V1.1 工件服务可以先于 V2 单独交付。
- 页面回显、观察者、人工接管在协议和数据面上与 Agent 控制通道完全隔离。
- 多副本部署下继续保持 V1 的不变量 I1 到 I3。

**Non-Goals:**
- 不做 Windows 桌面级回显，不做原生 Windows 控件或文件选择器操控。
- 不做音频，不做视频录制归档。录屏留待独立评估。
- 不做跨地域多活。

## Decisions

### D1. 实施分期
- **阶段 A（V1.1）**：MinIO 与 ObjectStore 落地 → browser-artifacts → profile-backup。后两者共用对象存储和“一次性预签名传输”机制。
- **阶段 B（V2-a）**：page-screencast 与 viewer-access。
- **阶段 C（V2-b）**：human-takeover，依赖 B 的播放器。
- **阶段 D**：gateway-scale-out，按容量需要实施。
- 各阶段之间只有单向依赖，可以分别发布。

### D2. 工件存储与扫描
- 把 V1 的 `CaptureStore` 扩展为通用 `ObjectStore`，后端固定为 **MinIO**，按用途分桶：`rbp-captures`、`rbp-artifacts`、`rbp-profile-snapshots`。全部启用 SSE（MinIO KES 或 SSE-S3），并为每个桶配置生命周期规则，作为应用层保留清理之外的兜底。截图也迁移到这个存储，迁移脚本单独提供。
- MinIO 部署：V1.1 先在 compose 中单节点部署，数据卷持久化。生产推荐分布式部署（≥4 盘）并开启版本控制。MinIO 不经公网暴露，只通过反向代理提供一个预签名专用路径（`/objects/*`，按 Host 转发到 MinIO），Windows 节点经此路径出站访问。
- 预签名 URL 由 rbp-api 用 MinIO SDK（`minio` Python 包）生成，PUT/GET 各自单独签发，默认有效期 15 分钟，并绑定 Content-Length。
- 上传流程：客户端先上传到中央，中央存入对象存储，然后由 ClamAV（clamd sidecar）异步扫描，扫描通过后状态变为 `AVAILABLE`。
- 投递到节点时，节点用一次性令牌通过 HTTPS GET 拉取文件，与 V1 截图上传通道对称，不经过控制 WSS。
- 下载检测：Worker 监听 `downloads\<instance_id>`。另外通过 CDP `Browser.downloadProgress` 事件确认下载完成，避免上传半成品文件。
- **备选方案**：把文件经 CDP 隧道 base64 传输。该方案会阻塞 Agent 通道，且帧大小受限，因此否决。

### D2b. Profile 快照
```
 实例 STOPPED --> rbp-api 判断（目录有变化 && 距上次快照 >= 6h）
        |
        v  经 hub 下发 SNAPSHOT_PROFILE{snapshot_id, put_url, excludes}
 Worker: 获取 Profile 锁 -> robocopy /MIR 到 staging\<snapshot_id>（排除缓存目录）-> 释放锁
        |   （持锁时间只是本地复制，通常为秒级）
        v
 Worker: zstd 打包成 tar，计算 sha256，用预签名 PUT 直传 MinIO -> COMMAND_RESULT{sha256,size}
        |
        v
 rbp-api: 对 MinIO 对象执行 stat 与哈希校验 -> COMPLETED；执行保留策略（保留 3 份 / 30 天）
```
- **“目录有变化”的判定**：Worker 对 Profile 内若干关键文件（`Default\Cookies`、`Local State`、`Default\Preferences`、`Default\Login Data`）的 mtime 与大小计算指纹，与上次快照的指纹比较，避免对没有变化的 Profile 重复快照。
- **恢复**：Worker 先下载到 staging，校验 sha256，解压到 `<id>.restore`，然后 `rename` 原目录为 `<id>.bak-<ts>`，再 `rename` `<id>.restore` 为 `<id>`。新目录首次被实例成功加载（CDP 就绪）后删除 `.bak`；失败时反向 rename 回退。整个过程持有 Profile 锁。
- **协议**：新增 `SNAPSHOT_PROFILE`、`RESTORE_PROFILE` 两种消息，协议 minor 版本升级。
- **DPAPI 限制**：Windows 上的 Chrome 用 DPAPI 保护 `Local State` 中的 `os_crypt` 密钥，Cookie 与密码依赖该密钥解密。DPAPI 主密钥属于具体的 Windows 用户和机器（非域漫游配置时），所以跨 VM 或跨用户恢复后，Cookie 很可能无法解密。快照元数据中记录 `source_node_id` 与 `source_windows_user`，跨环境恢复要求 Platform Admin 显式确认风险。V2 不做 `os_crypt` 密钥的导出或重新封装，那样会在中央持有可解密 Cookie 的密钥，安全代价过高。
- **备选方案**：
  - 增量备份（restic 等）：能节省存储，但恢复链路复杂，V2 不采用。
  - Volume Shadow Copy 在线快照：可以不等实例停止，但 Chrome 运行时 SQLite 文件可能不一致，因此否决。

### D3. Screencast 采集与传输
- 节点为每个被订阅的 Target 建立一个独立的本地 CDP session，调用 `Page.startScreencast{format: jpeg, quality, maxWidth, everyNthFrame}`。每收到一帧都必须执行 `screencastFrameAck`，节点用 ack 节奏控制帧率。
- 节点主动连接 Viewer Gateway 的 `/ws/v1/nodes/screencast`（同样是节点出站），一个 Target 对应一条上行连接，由网关扇出给多个观察者。
- 网关为每个观察者维护“只保留最新帧”的单槽缓冲，实现丢旧帧的背压策略。
- **PoC 前置项**：后台标签页 screencast 的帧行为，以及与 Browser Use 同时操作同一 Target 时是否互相干扰。如果后台标签页不可行，返回 `TARGET_NOT_VISIBLE`，不切换标签页。
- **备选方案**：WebRTC。延迟更低，但需要 TURN 服务，复杂度高。待 V2 体验评估后再决定。

### D4. Viewer 授权
- 新增 `viewer_grants` 表，字段包括 grantee、instance、target（可选）、`expires_at`、`revoked_at`。viewer token 复用 V1 connect ticket 的实现（一次性、只存哈希），但签发方与校验入口相互隔离。
- 观看连接上只允许 `subscribe`、`unsubscribe`、`ping` 三种上行消息，其他任何上行消息都导致连接以 1008 关闭。

### D5. 人工接管与租约扩展
- 租约增加 `holder_kind`（`agent` | `human`）与 `parent_lease_id`。接管流程：
```
 ACTIVE(agent, e=8) --takeover--> REVOKING(e=8) --fence ack--> RELEASED
                                   \--> ACTIVE(human, e=9, parent=L8)
 ACTIVE(human, e=9) --handback--> REVOKING(e=9) --fence ack--> ACTIVE(agent, e=10)
```
- 这需要对 V1 `browser-leases` 补充 MODIFIED delta：状态转移表与 holder 字段。V1 归档后补写。
- 人工输入通道 `/takeover/input` 由节点在本地执行 `Input.dispatchMouseEvent`、`Input.dispatchKeyEvent` 等。坐标按 screencast 帧的元数据（deviceWidth/Height、偏移）换算。
- 接管者在接管期间同时获得 viewer 能力，不需要另建 grant。

### D6. 多副本
- **rbp-api**：V1 已经无状态，后台任务也已用 advisory lock 选主，直接扩副本即可。
- **rbp-hub 多副本**是本阶段的主要工作：
  - **归属表**：`node_connections(node_id PK, hub_replica_id, connection_epoch, connected_at)`，通过条件 UPDATE 抢占。
  - **命令路由**：rbp-api 的 `HubClient` 先查归属表，再把内部 RPC 发到对应的 hub 副本。V1 是单 hub 固定地址，这里改为按 `hub_replica_id` 解析内部地址。
  - **跨副本事件**：继续使用 V1 的 PG LISTEN/NOTIFY，消息体只携带 ID。如果压测发现吞吐不足，再切换到 Redis Streams。
  - **隧道路由**：`OPEN_TUNNEL` 携带 `gateway_replica_url`（带副本标识的公开路径，例如 `/r/<replica>/ws/v1/nodes/tunnel`），反向代理按路径前缀路由到对应 hub 副本。这样节点直接接入客户端所在副本，避免副本间转发数据。客户端的 `/cdp/connect` 可以落在任意 hub 副本。

## Risks / Trade-offs

- [screencast 带宽：每路约 0.3–2 Mbps × 观察者数] → 默认 5 fps 与分辨率上限，按租约统计带宽，设置上限告警。
- [后台标签页无法回显] → 明确返回错误，不切换标签页。在文档中说明只支持可见目标。
- [人工输入坐标偏移（DPI 缩放、窗口尺寸）] → 以帧元数据为准换算坐标，PoC 中验证 Windows 125%/150% 缩放。
- [LISTEN/NOTIFY 负载上限] → 消息体只放 ID，保留切换到 Redis 的抽象层。
- [MinIO 引入新的运维成本] → V1.1 先单节点部署，生产推荐分布式部署并开启版本控制。MinIO 自身的备份（跨站复制或 `mc mirror`）写入运维文档。
- [跨 VM 恢复 Profile 后登录状态丢失（DPAPI）] → spec 中明确不承诺，跨环境恢复需要显式确认；PoC 实测同一 VM 重建 Windows 用户、以及不同 VM 两种情况，结果写入文档。
- [Profile 快照体积大，占用 WAN 带宽] → 排除缓存目录，zstd 压缩，最少间隔 6 小时，只在有变化时快照；统计快照体积指标。
- [快照暂存占用节点磁盘] → 上传完成后立即删除 staging；磁盘可用空间低于阈值时跳过本次快照并告警。
- [恶意文件扫描有漏报] → 工件只在远程浏览器和授权客户端之间流转，在文档中声明扫描并非绝对保证。

## Migration Plan

1. 阶段 A：在 compose 中加入 MinIO 并创建桶与生命周期规则；执行数据库迁移，新增 `artifacts`、`profile_snapshots` 表；部署 clamd；把截图存储迁移到 MinIO（可回滚到本地卷）；Worker 升级后支持 `SNAPSHOT_PROFILE`/`RESTORE_PROFILE`，旧版 Worker 不声明该能力，调度时跳过快照。
2. 阶段 B/C：协议 minor 升级到 1.1，新增消息类型。旧版 Worker 不声明 screencast 能力，调度时据此识别。Viewer Gateway 可以与 API 同进程，也可以独立部署。
3. 阶段 D：先在单个 hub 上启用归属表与按副本路由，行为与 V1 等价；再扩到 2 个 hub 副本灰度。回滚时缩回单个 hub 即可。

## Open Questions

- 页面回显的编码格式最终选 JPEG 帧还是 WebRTC，等阶段 B 的体验评估后定。两种方案的 spec 行为一致，不影响 spec。
