# Spec Delta: audit-log

## Purpose

为 RBP 中的管理动作、高危操作与资源授予/回收提供可追溯、不可轻易篡改的审计记录，回答“谁、何时、对什么资源、做了什么、结果如何”。

## ADDED Requirements

### Requirement: 审计事件覆盖范围
系统 SHALL 为以下动作写入审计事件：节点注册码生成与消费、节点凭据签发/轮换/吊销、节点 Drain/Resume/策略修改、浏览器实例创建/停止/重启/驱逐/健康恢复、Profile 创建/删除/ACL 变更、租约授予/释放/过期撤销/强制撤销、connect ticket 签发、截图创建/下载/删除、API Key 创建/吊销、主体创建/禁用、初始管理员引导、登录成功与失败。

#### Scenario: 强制撤销租约被审计
- **WHEN** 管理员以原因 "stuck test" 强制撤销租约 L
- **THEN** 产生一条 `lease.revoke` 审计事件，含操作者、租约 ID、实例 ID、原因与结果

#### Scenario: 系统动作被审计
- **WHEN** 调度器为腾出容量驱逐空闲实例 I
- **THEN** 产生 `instance.evict` 审计事件，`actor` 为 `system`，含驱逐原因

### Requirement: 审计事件结构
每条审计事件 SHALL 包含：`id`、`occurred_at`（UTC）、`actor_type`（principal/node/system）、`actor_id`、`action`、`resource_type`、`resource_id`、`project_id`（适用时）、`result`（success/denied/failed）、`reason`（可选）、`request_id`、`source_ip`。事件 MUST NOT 包含机密明文或 CDP 正文。

#### Scenario: 被拒绝的操作也被审计
- **WHEN** Operator 尝试删除其无权限的 Profile
- **THEN** 产生 `result=denied` 的 `profile.delete` 审计事件

### Requirement: 审计防篡改
审计事件 SHALL 仅追加写入；应用层 MUST NOT 提供修改或删除单条审计的接口。每条事件 SHALL 携带基于前一事件哈希的链式哈希，使篡改或删除可被校验发现。按保留策略的批量清理 SHALL 只删除最早的连续区段并记录清理事件。

#### Scenario: 篡改检测
- **WHEN** 有人直接在数据库中修改某条审计记录的 `action`
- **THEN** 审计校验命令报告从该事件起哈希链断裂

### Requirement: 审计查询
系统 SHALL 提供 `GET /api/v1/audit-events`，支持按时间区间、actor、action、resource_type、resource_id、result 过滤与游标分页。Platform Admin 可查全部；Project Admin 仅可查其项目相关事件；需要 `admin:audit` 作用域。

#### Scenario: 项目范围过滤
- **WHEN** 项目 P1 的 Project Admin 查询审计
- **THEN** 结果只包含 `project_id = P1` 的事件

### Requirement: 审计保留期
审计事件 SHALL 按可配置保留期（默认 90 天）保留，到期由后台任务清理。

#### Scenario: 到期清理
- **WHEN** 保留期配置为 90 天且存在 91 天前的事件
- **THEN** 清理任务删除这些事件并写入一条 `audit.purge` 事件，记录清理范围
