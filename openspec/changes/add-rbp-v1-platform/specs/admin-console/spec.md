# Spec Delta: admin-console

## Purpose

定义中央管理 API 的资源操作面与独立单页管理控制台的功能范围，使管理员在不直接访问数据库的前提下完成节点、实例、租约、Profile 授权、排队、截图证据与审计的日常维护。

## ADDED Requirements

### Requirement: 节点管理 API
系统 SHALL 提供 `GET /nodes`、`GET /nodes/{id}`（状态、版本、OS、Chrome 版本、Windows 会话、有效/上报/策略容量、占用数、CPU/内存/磁盘、最后心跳、`connection_epoch`、Drain 标志、标签、最近错误）、`POST /nodes/enroll-tokens`、`POST /nodes/{id}/drain`、`POST /nodes/{id}/resume`、`PATCH /nodes/{id}/policy`、`PATCH /nodes/{id}/tags`、`POST /nodes/{id}/credentials/revoke`，需 `admin:nodes`。

#### Scenario: 列表与详情一致
- **WHEN** 存在 3 个节点时分别调用列表与详情
- **THEN** 同一节点的状态、容量与占用数一致

### Requirement: 实例管理 API
系统 SHALL 提供 `GET /instances`（按节点、状态、Profile 过滤）、`GET /instances/{id}`（节点、Profile、状态、generation、当前租约、Chrome 版本、健康、最近错误）、`POST /instances/{id}/restart`、`DELETE /instances/{id}`，需 `admin:instances`。重启与停止 MUST 只作用于无有效租约的实例，否则返回 409 `LEASE_CONFLICT`，管理员需先撤销租约。

#### Scenario: 停止使用中的实例
- **WHEN** 管理员对持有 ACTIVE 租约的实例调用 DELETE
- **THEN** 响应为 409 `LEASE_CONFLICT`，实例不变

#### Scenario: 重启空闲实例
- **WHEN** 管理员重启 READY 空闲实例
- **THEN** 实例 generation 加一并重新进入 READY，Profile 不变

### Requirement: 租约与排队管理 API
系统 SHALL 提供 `GET /leases`（过滤主体/实例/节点/状态）、`POST /leases/{id}/revoke`，以及 `GET /acquisition-requests`（排队请求、等待时长、失败原因）与管理员取消排队请求，需 `admin:leases`。

#### Scenario: 查看排队
- **WHEN** 有 5 个请求排队
- **THEN** 管理员可看到每个请求的主体、Profile、约束、已等待时长

### Requirement: 主体与凭据管理 API
系统 SHALL 提供主体（创建、禁用、角色分配、项目成员关系）、项目（创建、列出）、API Key（创建、列出、吊销）与 Profile ACL 管理接口，需 `admin:principals` 或相应项目管理权限。

#### Scenario: 吊销 Key
- **WHEN** 管理员通过 API 吊销 Key K
- **THEN** K 的后续请求返回 401，并记录审计

### Requirement: 本人租约自助
Operator SHALL 能在控制台查看本人租约与租期并释放本人租约；释放后显示的实例状态与隧道状态 MUST 一致。

#### Scenario: 自助释放
- **WHEN** Operator 在控制台释放其租约
- **THEN** 租约显示 `RELEASED`，对应实例显示 `READY`，无活动隧道

### Requirement: 独立 SPA 控制台
系统 SHALL 提供独立构建与部署的单页 Web 控制台，仅通过公开的 `/api/v1` 管理与客户端 API 工作，包含页面：登录、节点列表/详情、实例列表/详情（含目标列表与截图操作）、租约列表、排队请求、Profile 与 ACL、主体与 API Key、审计事件。控制台 MUST 按当前会话角色隐藏或禁用无权操作。

#### Scenario: 不依赖数据库
- **WHEN** 管理员执行 Drain 节点、撤销租约、修改 Profile ACL
- **THEN** 全部通过控制台完成，操作结果在控制台可见并有审计

#### Scenario: Operator 视图受限
- **WHEN** Operator 登录控制台
- **THEN** 不显示节点管理、主体管理与全局审计菜单

### Requirement: 高危操作确认
控制台对撤销租约、停止实例、删除 Profile、吊销凭据、Drain 节点 SHALL 要求二次确认并填写原因（撤销/停止/Drain 时必填），原因随请求提交至审计。

#### Scenario: 撤销需要原因
- **WHEN** 管理员在控制台点击撤销租约但未填写原因
- **THEN** 提交按钮不可用

### Requirement: 列表筛选与刷新
控制台列表 SHALL 支持筛选、游标分页，且节点/实例/租约视图 SHALL 至少每 10 秒自动刷新或支持手动刷新。

#### Scenario: 状态变化可见
- **WHEN** 节点由 ONLINE 变为 OFFLINE
- **THEN** 控制台节点列表在 10 秒内显示新状态

### Requirement: 截图证据查看
控制台 SHALL 允许有权用户对实例目标发起截图、查看截图元数据与图像、按租约/实例检索历史截图。

#### Scenario: 发起截图
- **WHEN** 租约所有者在实例详情页对某目标点击截图
- **THEN** 页面展示新截图与 `captured_at`、`sha256`
