# 量潮数字资产云软件工程档案

qtcloud-asset 是可视化对象存储应用，把散落在阿里云 OSS 上的桶和文件以可视化方式呈现，让非技术人员无需 CLI 也能查看资源状态。

## 系统架构

系统由三个组件构成，围绕只读的 OSS 资产发现组织：

```
Studio (Flutter Web) ← HTTP → Provider (Go) ← SDK → 阿里云 OSS
                              ↑
                         CLI (Rust 管理工具)
```

Provider 通过 `OssAdapter` 只读发现 OSS 桶与对象；Studio 负责可视化展示；CLI 负责本地资产扫描、验证与归档，并可通过 `oss` 子命令复用 Provider 接口。

## 组件与技术栈

| 组件 | 语言与框架 | 职责 |
|:--|:--|:--|
| Provider | Go 1.25，net/http，aliyun-oss-go-sdk | 服务端，HTTP API 与 OSS 只读发现 |
| Studio | Flutter 3.24.5，Riverpod，go_router | Web 客户端，桶与文件浏览 |
| CLI | Rust 1.75，clap | 本地资产扫描、验证、归档与 OSS 操作 |

Provider 内部按 API、Service、Repository、Schema 分层，`OssAdapter` 是 `SourceAdapter` 接口的第一个实现，为后续接入其他资产源（GitHub、飞书）预留扩展点。Studio 侧按屏幕组织，平台相关逻辑（下载链接、HTTP 客户端）通过 io/web 双实现隔离。

## 工程实践

质量门禁按组件各自执行：Studio 用 `flutter analyze` 与 `dart format`，Provider 用 `go vet` 与 `gofmt`，CLI 用 `clippy` 与 `rustfmt`。Provider 的 handler 层有按场景拆分的测试文件（鉴权、CORS、限流、分享、排序等），CLI 在 Rust 重写后保留 56 个测试。

运维与部署遵循平台统一命名：Studio 静态站点发布到 `qtcloud-asset-studio` 桶，正式入口 `asset.cloud.quanttide.com`，兼容入口 `asset.quanttide.com` 暂不下线；Provider API 通过 `https://api.quanttide.com/qtcloud-asset` 暴露，生产部署于阿里云函数计算，审计日志输出结构化 `qtcloud_asset_audit` 并经 SLS 持久查询。基础设施以 Terraform 代码管理（`manifests/iac/`）。

## 权限与安全模型

账号体系支持 SSO 与本地账号密码（PBKDF2-SHA256 哈希存储）两种模式，角色分 `viewer` 与 `admin`。`viewer` 隐藏 `-private` 桶与 `quanttide-terraform-state`；`admin` 可查看全部但不能生成私密桶对象链接。私密桶只展示元数据，公开桶支持生成直链与文件/文件夹分享页（单文件下载与 ZIP 打包下载）。

## 当前状态

组件成熟度不均衡：CLI 最完整（`cli/v0.1.0-alpha.2`，run/scan/validate/config/version 五命令）；Provider 完成 Go 重写与只读端点、账号门禁、审计，Service/Repository 层仍在补全；Studio 已具备桶列表、文件浏览、登录、分享等能力。文档体系处于 v0.1.x 对齐阶段，BRD/PRD/QA 已有内容，ADD 与 IXD 待补齐。
