# Spec Delta: browser-screenshots

## Purpose

提供对远程浏览器页面目标的授权查询与非破坏性 CDP 截图能力，截图经独立媒体通道传输并带完整元数据、授权下载与保留期，不干扰 Agent 控制与节点心跳。

## ADDED Requirements

### Requirement: 页面目标列表
`GET /api/v1/instances/{instance_id}/targets` SHALL 返回实例的页面目标列表（`target_id`、`type`、`title`、`url`、`attached`）。仅该实例当前租约的所属主体、该 Profile 项目的 Project Admin 与 Platform Admin 可访问。

#### Scenario: 列出目标
- **WHEN** 租约所有者查询实例目标
- **THEN** 返回当前所有 `type=page` 的目标

#### Scenario: 无权查看
- **WHEN** 其他 Operator 查询该实例目标
- **THEN** 响应为 404

### Requirement: 截图请求
`POST /api/v1/instances/{instance_id}/targets/{target_id}/screenshots`（需 `browser:screenshot`）SHALL 接受 `format`（V1 仅 `png`）、`full_page`（默认 false）、`max_height_px`（整页时上限默认 16384），同步等待（默认 15 秒）完成后返回 `capture_id`、`state`、元数据与下载地址；超出同步窗口返回 202 与 `capture_id` 供轮询。

#### Scenario: 视口截图
- **WHEN** 租约所有者对目标 T 请求视口截图
- **THEN** 返回 `state=COMPLETED`，下载内容为有效 PNG

#### Scenario: 整页超高
- **WHEN** 请求整页截图且页面高度超过 `max_height_px`
- **THEN** 图像高度被截断至上限，元数据 `truncated=true`

### Requirement: 截图授权
截图 MUST 校验调用方持有该实例的有效控制租约，或为该 Profile 项目的 Project Admin / Platform Admin。截图权限独立于 CDP 控制通道，MUST NOT 通过任意实例 ID 截取他人页面。

#### Scenario: 越权截图
- **WHEN** 用户 B 对用户 A 正在使用的实例请求截图
- **THEN** 响应为 404，节点未执行截图

### Requirement: 非破坏性截图
截图 SHALL 由节点使用独立于用户隧道的本地 CDP 会话，附加到指定目标后执行 `Page.captureScreenshot` 并分离；MUST NOT 调用 `Page.bringToFront`、导航、关闭、激活或修改视口等改变页面状态的方法。截图在超时（默认 10 秒）内未完成 SHALL 以失败返回且不影响 Agent 会话。

#### Scenario: 执行中截图
- **WHEN** Agent 正在操作目标 T 时对 T 截图
- **THEN** 截图前后 T 的 URL、焦点标签与 Agent 会话状态不变，Agent 后续操作正常

#### Scenario: 截图超时
- **WHEN** 后台标签页截图 10 秒内未返回
- **THEN** 截图 `state=FAILED`，`error.code=CDP_UNHEALTHY` 或 `CAPTURE_TIMEOUT`，Agent 连接不受影响

### Requirement: 独立媒体传输
节点 SHALL 通过 HTTPS `PUT /api/v1/internal/captures/{capture_id}/upload` 上传图像，使用随 `SCREENSHOT` 命令下发的单次、短期（默认 120 秒）、绑定 `capture_id` 的上传令牌；图像 MUST NOT 经控制 WSS 传输。上传大小 SHALL 受限（默认 50 MiB）。

#### Scenario: 截图不阻塞心跳
- **WHEN** 并行执行 20 次整页截图
- **THEN** 节点心跳持续按时到达，节点不被判定失联

#### Scenario: 上传令牌复用
- **WHEN** 已使用的上传令牌被再次用于上传
- **THEN** 请求被拒绝为 401

### Requirement: 截图元数据
每个截图 SHALL 记录 `capture_id`、`instance_id`、`target_id`、`lease_id`（若有）、`requester_id`、`captured_at`、`format`、`width`、`height`、`size_bytes`、`sha256`、`truncated`、`expires_at`。中央 SHALL 校验上传内容的 sha256 与节点声明一致。

#### Scenario: 哈希校验
- **WHEN** 上传内容哈希与节点声明不一致
- **THEN** 截图标记为 `FAILED`，不提供下载

### Requirement: 授权下载
`GET /api/v1/captures/{capture_id}` SHALL 返回元数据，`GET /api/v1/captures/{capture_id}/content` SHALL 返回 PNG；仅截图请求者、项目 Project Admin 与 Platform Admin 可访问，下载 SHALL 记录审计。

#### Scenario: 他人下载
- **WHEN** 无权限用户使用他人的 `capture_id` 下载
- **THEN** 响应为 404

### Requirement: 截图保留与删除
截图 SHALL 按可配置保留期（默认 7 天）自动清理；请求者与管理员 SHALL 能提前删除（`DELETE /api/v1/captures/{id}`），删除写入审计。

#### Scenario: 到期清理
- **WHEN** 截图超过 7 天
- **THEN** 图像文件被删除，元数据标记 `EXPIRED`，下载返回 404

### Requirement: 截图存储隔离
截图存储目录 SHALL 仅由中央服务账户可读写，对象键 MUST 为不可推测的随机值，不得包含用户提供的文件名。

#### Scenario: 对象键
- **WHEN** 检查存储中的截图文件名
- **THEN** 文件名为随机标识，与 URL、页面标题无关
