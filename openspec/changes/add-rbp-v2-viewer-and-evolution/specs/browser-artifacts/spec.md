# Spec Delta: browser-artifacts

## Purpose

在客户端（Agent 机器）与远程 Windows 浏览器文件系统之间提供受控的文件交换：上传文件供页面文件输入使用、归档浏览器下载产物并授权取回，同时执行大小、类型、路径、恶意内容与保留策略。

## ADDED Requirements

### Requirement: 上传工件
租约所有者 SHALL 能通过 `POST /api/v1/leases/{id}/artifacts/uploads` 上传文件（默认单文件上限 100 MiB，可配置），获得 `artifact_id`。文件在扫描通过前 MUST NOT 被投递到节点。

#### Scenario: 上传成功
- **WHEN** 租约所有者上传 2 MiB 的 PDF
- **THEN** 返回 `artifact_id`，扫描通过后状态变为 `AVAILABLE`

#### Scenario: 超出大小
- **WHEN** 上传 200 MiB 文件
- **THEN** 响应为 413 `ARTIFACT_TOO_LARGE`

### Requirement: 投递到浏览器文件输入
租约所有者 SHALL 能请求将 `AVAILABLE` 上传工件投递到实例并设置到指定 Target 的文件输入元素（`DOM.setFileInputFiles`）。节点 SHALL 将文件写入 `uploads\<instance_id>\<random>` 后使用，路径 MUST 由节点生成，客户端 MUST NOT 指定节点路径。

#### Scenario: 设置文件输入
- **WHEN** 客户端以 `artifact_id` 与目标元素选择器请求投递
- **THEN** 页面文件输入获得该文件，节点路径不出现在响应中

#### Scenario: 指定路径被拒
- **WHEN** 请求携带 `node_path` 字段
- **THEN** 响应为 422 `VALIDATION_FAILED`

### Requirement: 下载产物归档
实例下载目录中的新完成文件 SHALL 被节点检测并上传为下载工件，关联 `instance_id`、`lease_id`、原始文件名（仅作元数据，已净化）、大小与 sha256。租约所有者 SHALL 能列出并下载这些工件。

#### Scenario: 页面下载文件
- **WHEN** Agent 触发页面下载 report.csv
- **THEN** 下载完成后 `GET /leases/{id}/artifacts?kind=download` 列出该文件，可授权下载

### Requirement: 工件授权与隔离
工件 SHALL 仅对其租约所有者、项目 Project Admin 与 Platform Admin 可见；跨租约访问 MUST 返回 404。临时 Profile 的工件 SHALL 随租约结束按保留策略处理。

#### Scenario: 他人访问工件
- **WHEN** 其他用户请求某工件
- **THEN** 响应为 404

### Requirement: 恶意内容扫描
所有上传与下载工件 SHALL 经过恶意内容扫描；被判定恶意的 MUST 标记为 `QUARANTINED` 且不可下载或投递，并写入审计。

#### Scenario: 隔离恶意文件
- **WHEN** 上传 EICAR 测试文件
- **THEN** 状态为 `QUARANTINED`，投递与下载请求返回 409

### Requirement: 工件保留
工件 SHALL 按可配置保留期（默认 7 天）清理，节点侧的上传与下载目录 SHALL 在实例停止或工件归档后清理。

#### Scenario: 节点目录清理
- **WHEN** 下载工件已归档到中央
- **THEN** 节点下载目录中的对应文件被删除
