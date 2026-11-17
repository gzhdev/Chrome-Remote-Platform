# Spec Delta: human-takeover

## Purpose

定义 Agent 与人工操作者之间安全转交浏览器控制权的流程：经 lease fencing 隔离旧控制方后才开放人工页面输入，结束后可交还或释放，全程保持单控制方不变量。

## ADDED Requirements

### Requirement: 发起接管
租约所有者或具有项目管理权限的主体 SHALL 能对 ACTIVE 租约发起接管请求，指定接管者；系统 SHALL 以新的 `lease_epoch` 创建接管租约，并对原 epoch 执行与撤销相同的 fencing 流程。

#### Scenario: 接管前旧 Agent 无控制能力
- **WHEN** 对 Agent 的租约发起接管
- **THEN** 在人工输入通道开放前，Agent 的 CDP 连接已被关闭且其重连被拒绝

### Requirement: 人工输入通道
接管者 SHALL 通过独立的人工输入 WSS 发送页面级鼠标、键盘、滚轮与导航事件，节点 SHALL 以 CDP `Input.*` 等方法在目标页面执行；事件 MUST 只作用于接管租约绑定的实例与目标。

#### Scenario: 人工点击
- **WHEN** 接管者在播放器中点击按钮
- **THEN** 远程页面上对应坐标的按钮被点击，画面回显中可见结果

### Requirement: 交还与释放
接管者 SHALL 能交还控制权（系统为原主体签发新租约 epoch，原 Agent 可重新领票连接）或直接释放租约。交还 MUST 先 fencing 人工输入通道。

#### Scenario: 交还给 Agent
- **WHEN** 接管者选择交还
- **THEN** 人工输入通道关闭，原主体获得新 epoch 的 ACTIVE 租约，可通过 SDK 重新连接

### Requirement: 接管审计与不变量
接管、交还、释放 SHALL 写入审计；任意时刻同一实例 MUST 至多一个控制方（Agent CDP 连接或人工输入通道）。

#### Scenario: 不变量
- **WHEN** 在接管与交还过程中随机注入网络断开
- **THEN** 不存在 Agent 与人工同时具备控制能力的时间窗口
