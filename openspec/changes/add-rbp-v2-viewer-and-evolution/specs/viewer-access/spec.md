# Spec Delta: viewer-access

## Purpose

定义只读观察者的授权模型：viewer grant 的签发与撤销、viewer token 的作用范围，并保证观察者无法获得或注入任何 CDP 与控制能力。

## ADDED Requirements

### Requirement: Viewer Grant
租约所有者、项目 Project Admin 与 Platform Admin SHALL 能为指定主体创建针对 `instance_id`（可选 `target_id`）的 viewer grant，带有效期（默认 1 小时）。grant 可被创建者或管理员撤销，撤销后对应观看流 MUST 在 5 秒内被关闭。

#### Scenario: 撤销观看
- **WHEN** 撤销观察者 V 的 grant
- **THEN** V 的观看流在 5 秒内关闭，重新订阅被拒绝

### Requirement: Viewer Token 作用域
viewer token SHALL 一次性、短期（默认 60 秒）用于建立观看连接，仅绑定 grant 的实例/目标范围。viewer token MUST NOT 被 `/cdp/connect`、截图写操作或任何管理 API 接受。

#### Scenario: viewer token 连接 CDP
- **WHEN** 使用 viewer token 连接 `/cdp/connect`
- **THEN** 握手被拒绝

### Requirement: 只读不可注入
观看通道 SHALL 为单向下行；观察者发送的任何上行数据（除协议级 ping/pong 与订阅控制消息外）MUST 被丢弃并可导致连接关闭，MUST NOT 转化为 CDP 命令或输入事件。

#### Scenario: 注入尝试
- **WHEN** 观察者在观看连接上发送 `{"method":"Runtime.evaluate"}`
- **THEN** 消息被丢弃，连接以策略违规关闭，页面不受影响

### Requirement: 多观察者
同一目标 SHALL 支持多个观察者并发观看（默认上限 5，可配置），节点对同一目标 SHALL 只采集一路帧并在网关侧扇出。

#### Scenario: 三人同时观看
- **WHEN** 三个授权观察者同时订阅同一目标
- **THEN** 三人均收到帧，节点只运行一个采集会话
