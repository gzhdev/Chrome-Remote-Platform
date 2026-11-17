# Spec Delta: client-sdk

## Purpose

定义 RBP Python Client SDK 的对外契约：申请浏览器、获取 CDP 连接地址、后台续租、可靠释放与 Browser Use 集成，使用户电脑与 CI 使用同一套 SDK 透明接入远程 Windows 浏览器。

## ADDED Requirements

### Requirement: SDK 运行环境
SDK SHALL 作为独立 Python 包发布，支持 Python ≥ 3.11，同时提供同步与 asyncio 接口；SDK MUST NOT 依赖 Browser Use（集成示例除外），MUST NOT 处理或上传任何 LLM 凭据。

#### Scenario: 无 Browser Use 环境安装
- **WHEN** 在仅安装 SDK 的 Python 3.11 环境中导入 SDK
- **THEN** 导入成功，无 Browser Use 依赖

### Requirement: 客户端配置
SDK SHALL 通过参数或环境变量（`RBP_BASE_URL`、`RBP_API_KEY`、`RBP_CA_BUNDLE`）配置服务地址、API Key 与自定义 CA；API Key MUST NOT 被 SDK 写入日志或异常消息。

#### Scenario: 自签 CA
- **WHEN** 配置 `RBP_CA_BUNDLE` 指向企业 CA
- **THEN** SDK 以该 CA 校验 HTTPS 与 WSS，且不禁用证书校验

### Requirement: 申请浏览器
SDK SHALL 提供 `acquire(profile_id=None, reuse=True, wait_timeout=0, node_tags=None)`，自动生成并在重试中复用 `Idempotency-Key`，在服务端返回 202 时透明轮询 acquisition request 直到成功、失败或 `wait_timeout` 到期，并以类型化异常暴露错误码（如 `ResourceExhausted`、`ProfileInUse`、`Forbidden`）。

#### Scenario: 透明排队
- **WHEN** 资源已满，调用 `acquire(wait_timeout=120)`，60 秒后资源可用
- **THEN** 调用阻塞约 60 秒后返回会话对象，无需调用方轮询

#### Scenario: 网络重试不重复申请
- **WHEN** acquire 请求在响应前网络中断，SDK 自动重试
- **THEN** 重试使用相同 `Idempotency-Key`，服务端只产生一个租约

### Requirement: 获取 CDP 地址
会话对象 SHALL 提供 `cdp_url()`，每次调用领取新的一次性 connect ticket 并返回 `wss://.../cdp/connect?ticket=...`。SDK MUST NOT 缓存或复用已用过的票据。

#### Scenario: 重连取新票据
- **WHEN** CDP 连接断开后调用方再次调用 `cdp_url()`
- **THEN** 返回包含新票据的 URL

### Requirement: 后台续租
会话对象 SHALL 在活动期间按服务端返回的 `renew_after_seconds` 在后台自动续租；续租暂时失败 SHALL 指数退避重试直至 `grace_until`；租约被确认失效（`LEASE_EXPIRED`/`FAILED`/撤销）时 SHALL 停止续租并通过回调与状态属性通知调用方。

#### Scenario: 长任务不过期
- **WHEN** 使用会话运行 1 小时的 Agent 任务
- **THEN** 租约始终 ACTIVE

#### Scenario: 被撤销通知
- **WHEN** 管理员撤销租约
- **THEN** SDK 停止续租，会话状态变为 `revoked`，已注册回调被调用

### Requirement: 可靠释放
会话对象 SHALL 支持上下文管理器（`with` / `async with`），退出时在 `finally` 语义下调用释放，释放失败 MUST 不掩盖调用方原始异常。SDK SHALL 在进程正常退出时尽力释放未释放的会话。服务端 MUST NOT 依赖 SDK 释放（由 TTL/宽限保证回收）。

#### Scenario: 异常退出仍释放
- **WHEN** `with` 块内 Agent 抛出异常
- **THEN** SDK 调用 `DELETE /leases/{id}`，原异常继续向外抛出

### Requirement: 截图与目标
SDK SHALL 提供 `list_targets()` 与 `screenshot(target_id, full_page=False) -> bytes` 便捷方法，并返回截图元数据。

#### Scenario: SDK 截图
- **WHEN** 调用 `session.screenshot(target_id)`
- **THEN** 返回 PNG 字节与 `capture_id`、`sha256`

### Requirement: Browser Use 集成
SDK SHALL 提供文档与可运行示例，展示以 `Browser(cdp_url=session.cdp_url())` 驱动 Browser Use 1.0.x Agent，并确保 Agent 结束不关闭远程 Chrome（使用 Browser Use 的保持浏览器存活选项）。示例 SHALL 固定 Browser Use 精确版本。

#### Scenario: 示例端到端
- **WHEN** 在用户电脑与 CI 主机上分别运行示例
- **THEN** 两者均成功控制远程 Windows Chrome 完成导航，结束后实例仍为 READY
