# Spec Delta: cdp-gateway

## Purpose

提供 Browser Use 可直接连接的、经鉴权的浏览器级 CDP WebSocket 入口：签发一次性短期票据，按租约请求节点主动建立数据隧道并安全配对，1:1 透明转发 CDP 帧，并在租约失效或节点失联时强制关闭，使 Chrome 调试端口永不暴露。

## ADDED Requirements

### Requirement: Connect Ticket 签发
`POST /api/v1/leases/{id}/connect-ticket` SHALL 仅对 `ACTIVE` 或 `GRACE` 租约的所属主体签发票据，返回 `cdp_url`（`wss://<public_host>/cdp/connect?ticket=<opaque>`）、`expires_in_seconds`（默认 60）与 `lease_id`。票据 MUST 为高熵不透明值，中央仅存哈希，并绑定 `lease_id`、`lease_epoch`、`instance_id`、`principal_id` 与网关标识。

#### Scenario: 非活动租约申请票据
- **WHEN** 对 `RELEASED` 租约申请票据
- **THEN** 响应为 409 `LEASE_EXPIRED`

#### Scenario: 票据内容不可推测
- **WHEN** 检查签发的 `cdp_url`
- **THEN** 其中不包含节点 IP、端口、Profile 路径或 browser GUID

### Requirement: 票据一次性与有效期
票据 MUST 在 WSS Upgrade 时被原子消费，消费后、过期后或对应租约 epoch 已变化时 MUST 被拒绝。票据只用于建连；已建立连接的寿命由租约决定。

#### Scenario: 重放票据
- **WHEN** 同一票据第二次用于 Upgrade
- **THEN** 握手被拒绝（HTTP 401 或 WSS 关闭码 4401），无隧道建立

#### Scenario: 过期票据
- **WHEN** 签发 61 秒后使用票据
- **THEN** 握手被拒绝

#### Scenario: 连接寿命超过票据
- **WHEN** 连接在票据有效期内建立并持续使用 30 分钟（租约持续续租）
- **THEN** 连接保持可用

### Requirement: 浏览器级 CDP 兼容
`/cdp/connect` SHALL 作为浏览器级 CDP WebSocket（等价于 `/devtools/browser/<guid>`），支持 Target 发现、`Target.attachToTarget`/`setAutoAttach`（含 flatten session）、多标签页与事件推送。V1 MUST NOT 要求客户端先访问 `/json/version`，MUST NOT 依赖自定义 WebSocket 握手 Header，MUST NOT 转换为非 CDP 协议。

#### Scenario: Browser Use 直接连接
- **WHEN** Browser Use 1.0.6 以 `Browser(cdp_url=<cdp_url>)` 运行 Agent
- **THEN** 完成打开网址、点击、读取文本、新建标签页与截图，无协议错误

### Requirement: 节点主动数据隧道
Gateway 接受客户端 Upgrade 后 SHALL 生成 `tunnel_id` 与一次性 `tunnel_nonce`，经该节点当前已认证控制连接下发 `OPEN_TUNNEL{tunnel_id, tunnel_nonce, instance_id, lease_epoch, instance_generation, connection_epoch, expires_at}`；节点 SHALL 主动连接 `/ws/v1/nodes/tunnel` 并以节点凭据与 nonce 认证。中央 MUST NOT 主动连接节点。

#### Scenario: 无入站端口完成 CDP 连接
- **WHEN** Windows VM 无任何入站端口，客户端以有效票据连接
- **THEN** CDP 连接成功建立并可操作页面

### Requirement: 隧道配对校验
Gateway SHALL 在配对前校验：节点身份与 `OPEN_TUNNEL` 目标节点一致、`tunnel_id` 存在且未配对、nonce 匹配且未使用、实例归属该节点、`lease_epoch` 为当前有效值、未超时。任一不符 MUST 拒绝该节点连接并让客户端连接失败关闭。

#### Scenario: 冒用节点
- **WHEN** 节点 B 携带发给节点 A 的 `tunnel_id` 与 nonce 连接
- **THEN** Gateway 拒绝，客户端连接不被配对，记录安全事件

#### Scenario: nonce 重放
- **WHEN** 同一 nonce 被第二次用于数据隧道
- **THEN** 连接被拒绝

### Requirement: 节点只连接本地实例
节点 SHALL 仅以 `instance_id` 在本地解析真实 CDP 端口与 GUID，并只连接 `ws://127.0.0.1:<port>/devtools/browser/<guid>`。`OPEN_TUNNEL` 中任何主机、端口、URL 或路径字段 MUST 被拒绝。

#### Scenario: 注入目标端口
- **WHEN** `OPEN_TUNNEL` payload 附加 `target_port=3389`
- **THEN** 节点以 `VALIDATION_FAILED` 拒绝，不建立任何连接

### Requirement: 建连超时
从客户端 Upgrade 被接受到隧道就绪 SHALL 有独立超时（默认 8 秒，可配置）。超时 MUST 以关闭码 4504（原因 `TUNNEL_TIMEOUT`）关闭客户端连接，并通知节点 `CLOSE_TUNNEL` 清理。Gateway MUST NOT 无限等待节点。

#### Scenario: 节点未及时接入
- **WHEN** 节点在 8 秒内未建立数据隧道
- **THEN** 客户端连接以 `TUNNEL_TIMEOUT` 关闭，网关与节点均无残留隧道

### Requirement: 透明转发
配对后 Gateway SHALL 双向透明转发文本与二进制帧，保持帧顺序与边界，MUST NOT 解析、修改或注入 CDP 消息内容；任一侧关闭 SHALL 以对应关闭码关闭另一侧。

#### Scenario: 长时间无串线
- **WHEN** 20 个并发隧道各连续运行 60 分钟 CDP 操作
- **THEN** 每个客户端只收到其对应浏览器的响应与事件，无错配

### Requirement: 帧大小、缓冲与背压
单帧上限默认 32 MiB（可配置），超限 MUST 以关闭码 1009 关闭该隧道。每方向缓冲有上限（默认 64 MiB），满时 SHALL 暂停读取对端以传递背压；Gateway MUST NOT 持久化或完整记录 CDP 帧正文。

#### Scenario: 大消息
- **WHEN** 客户端请求返回 20 MiB 的 `DOM.getOuterHTML` 结果
- **THEN** 消息完整送达

#### Scenario: 超大帧
- **WHEN** 单帧超过 32 MiB
- **THEN** 该隧道以 1009 关闭，其他隧道不受影响

### Requirement: 保活
Gateway 与两侧 SHALL 使用 WebSocket ping/pong 保活（默认间隔 20 秒、超时 20 秒），超时 SHALL 关闭隧道并更新隧道状态。

#### Scenario: 客户端网络中断
- **WHEN** 客户端网络静默中断
- **THEN** 不超过 40 秒内隧道被关闭，租约状态不因此改变

### Requirement: 每租约单写连接
同一租约 V1 SHALL 至多一条活动 CDP 连接（`max_open_connections_per_lease` 默认 1）。新连接以有效票据建立时，系统 SHALL 先关闭并确认旧连接已关闭，再配对新连接。

#### Scenario: 第二条连接
- **WHEN** 租约已有活动 CDP 连接时以新票据再次连接
- **THEN** 旧连接被关闭后新连接才生效，任何时刻不存在两条可写连接

### Requirement: 撤销与失联时强制关闭
租约进入 `RELEASING`/`REVOKING`/`FAILED`、主体凭据被吊销或节点 `OFFLINE` 时，Gateway MUST 关闭对应隧道（关闭码 4403/4410）并拒绝新连接。

#### Scenario: 撤销后不能操作
- **WHEN** 租约被撤销
- **THEN** 客户端 WSS 被关闭，以旧票据或新申请票据重连均失败

### Requirement: 节点自我 fencing
节点控制连接丢失时，Worker SHALL 立即关闭其所有数据隧道并拒绝在新控制连接完成对账前建立任何隧道。

#### Scenario: 控制通道断开
- **WHEN** 节点控制 WSS 断开
- **THEN** 该节点上所有 CDP 隧道被关闭，Chrome 实例保持运行

### Requirement: 不暴露内部地址
任何面向客户端的响应、错误消息与关闭原因 MUST NOT 包含原始 CDP URL、节点 IP、Windows 路径或 browser GUID。

#### Scenario: 错误信息检查
- **WHEN** 隧道因 CDP 不可用失败
- **THEN** 客户端得到 `CDP_UNHEALTHY` 与逻辑 `instance_id`，无内部地址

### Requirement: 隧道元数据记录
系统 SHALL 记录每条隧道的 `tunnel_id`、租约、实例、节点、`connection_epoch`、建立/关闭时间、关闭原因、双向字节数与帧数、配对耗时，MUST NOT 记录帧正文。

#### Scenario: 审计定位链路
- **WHEN** 查询某租约的隧道记录
- **THEN** 可获得每条隧道的建立时间、关闭原因与字节数

### Requirement: 反向代理兼容
部署 SHALL 在 TLS 反向代理后保持与直连一致的行为：正确处理 Upgrade、Host/Origin、长连接超时（≥ 租约最长时长）与 ping/pong；Gateway SHALL 不依赖 Origin 进行授权但 SHALL 拒绝配置为禁止的 Origin。

#### Scenario: 经反向代理连接
- **WHEN** 客户端通过 Nginx/Traefik 连接 `/cdp/connect` 并持续 60 分钟
- **THEN** 连接行为与直连 Gateway 一致，无代理超时断链
