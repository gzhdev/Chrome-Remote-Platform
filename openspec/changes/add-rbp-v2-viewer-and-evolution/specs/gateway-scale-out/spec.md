# Spec Delta: gateway-scale-out

## Purpose

使中央服务（Control API、节点控制接入、CDP Gateway、Viewer Gateway）可多副本部署，正确处理节点控制连接归属、隧道配对路由、跨副本命令投递与副本 Drain，同时保持 V1 的全部一致性不变量。

## ADDED Requirements

### Requirement: 节点控制连接归属
每个节点在任意时刻 SHALL 只有一个副本持有其有效控制连接，归属记录（`node_id → replica_id, connection_epoch`）SHALL 持久化于共享存储；新连接接入其他副本时 SHALL 使旧副本的连接失效。

#### Scenario: 节点切换副本
- **WHEN** 节点从副本 A 断开后重连到副本 B
- **THEN** 归属更新为 B 且 epoch 递增，A 上的残留连接消息被丢弃

### Requirement: 跨副本命令投递
任一副本 SHALL 能向任意节点下发控制命令：命令经跨副本消息总线路由到归属副本投递，应答回传发起副本；归属副本不可达时命令 SHALL 以结果未知处理并走对账。

#### Scenario: 跨副本创建实例
- **WHEN** acquire 落在副本 A，而目标节点连接在副本 B
- **THEN** `CREATE_BROWSER` 经 B 下发，A 收到结果并返回客户端

### Requirement: 隧道跨副本配对
客户端 CDP 连接与节点数据隧道 SHALL 可落在不同副本；系统 SHALL 通过让节点按 `OPEN_TUNNEL` 指定的副本地址接入，或经副本间转发完成配对，配对校验规则与 V1 相同。

#### Scenario: 客户端与节点在不同副本
- **WHEN** 客户端连接副本 A 的 `/cdp/connect`，节点控制连接在副本 B
- **THEN** 隧道成功配对并透明转发

### Requirement: 副本 Drain
运维 SHALL 能 Drain 某副本：其停止接受新连接，现有节点控制连接被引导到其他副本重连，现有 CDP 隧道保持至自然结束或超时（默认 30 分钟）后关闭。

#### Scenario: 滚动升级
- **WHEN** 逐个 Drain 并升级副本
- **THEN** 节点始终保持可调度，持有租约的客户端最多经历一次重连

### Requirement: 多副本下不变量保持
多副本部署下 SHALL 继续保证 V1 的 I1（单控制方）、I2（Profile 单挂载）、I3（不超配），这些约束 MUST 依赖共享数据库约束而非副本内存状态。

#### Scenario: 多副本并发争抢
- **WHEN** 两个副本同时处理同一实例的 acquire
- **THEN** 只有一个 ACTIVE 租约
