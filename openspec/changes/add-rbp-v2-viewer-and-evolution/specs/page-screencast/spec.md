# Spec Delta: page-screencast

## Purpose

提供远程浏览器页面级（非桌面级）的实时画面回显：节点采集页面帧、编码并经独立 Viewer Gateway 推送给授权观察者，具备限帧率、背压与隐私提示，且不干扰 Agent 控制。

## ADDED Requirements

### Requirement: 页面帧订阅
具有 viewer 授权的主体 SHALL 能通过 `wss://<host>/viewer/stream?token=...` 订阅指定 `instance_id + target_id` 的页面帧流。帧内容 MUST 为页面视口画面，MUST NOT 包含 Windows 桌面或浏览器窗口外框。

#### Scenario: 观看页面
- **WHEN** 观察者以有效 viewer token 订阅 Target T
- **THEN** 持续收到 T 的页面帧，画面随页面变化更新

### Requirement: 与控制通道隔离
页面回显 SHALL 使用独立于 Agent CDP 隧道的节点本地 CDP 会话与独立的数据通道（Viewer Gateway），帧传输 MUST NOT 占用 Agent 隧道或节点控制通道带宽。回显启动、停止与故障 MUST NOT 中断 Agent 会话。

#### Scenario: 回显不打断 Agent
- **WHEN** Agent 执行任务期间观察者开始与停止观看
- **THEN** Agent 的 CDP 操作无错误、无断链

### Requirement: 限帧率与背压
每路订阅 SHALL 支持可配置最大帧率（默认 5 fps）、最大分辨率与质量；观察者消费慢时系统 SHALL 丢弃旧帧只保留最新帧，MUST NOT 无限缓冲。

#### Scenario: 慢观察者
- **WHEN** 观察者网络带宽不足
- **THEN** 其收到的帧率下降，节点与网关内存不增长，其他观察者不受影响

### Requirement: 回显不改变页面
启动回显 MUST NOT 激活、聚焦、导航或改变目标页面视口；若技术上无法对后台标签页回显，系统 SHALL 明确返回 `TARGET_NOT_VISIBLE` 而非切换标签页。

#### Scenario: 后台标签页
- **WHEN** 订阅一个后台标签页且无法采集
- **THEN** 返回 `TARGET_NOT_VISIBLE`，Agent 当前标签页不变

### Requirement: 隐私提示与审计
开始与结束观看 SHALL 写入审计（观察者、实例、目标、时长）；租约所有者 SHALL 能查询其会话当前的观察者列表。

#### Scenario: 查询观察者
- **WHEN** 租约所有者查询当前观察者
- **THEN** 返回正在观看的主体与开始时间
