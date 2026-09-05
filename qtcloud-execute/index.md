# 量潮执行云软件工程档案

qtcloud-execute 是量潮执行云应用，以「清单 × 任务」的泳道看板承载业务的执行管理——任务可在四状态间自由推进与回退（真看板语义），主要使用者为执行业务的团队成员与 AI/脚本。

认证接入方案见 [auth.md](./auth.md)，支付接入方案见 [pay.md](./pay.md)。

## 系统架构

系统由四个组件构成，围绕服务端一份 JSON 任务文档组织：

```
Site (React 介绍页)   Studio (Flutter Web)    CLI (Rust，AI/脚本入口)
        │                    │ HTTP                │ HTTP
        │                    ▼                     ▼
        │              Provider (Go，阿里云 FC 3.0)
        │                    │ Store 接口
        │                    ▼
        └──►         OSS 桶 qtcloud-execute-provider
                          data/tasks.json（任务数据文档）
```

Provider 是契约的事实源：`data/tasks.json` 一份 JSON 文档即全部任务数据（无数据库，load/save 全量读写）。Studio 是团队日常看板；CLI 是纯服务端薄客户端，主要供 AI/脚本调用；Site 是介绍页，主 CTA 通向 Studio。

## 组件与技术栈

| 组件 | 语言与框架 | 版本 | 职责 |
|:--|:--|:--|:--|
| Studio | Flutter Web，flutter_bloc，go_router | 0.1.0-beta.4 | 泳道看板客户端，清单导航 + 任务卡片/弹窗 |
| Provider | Go，net/http，FC 3.0 custom-container | 0.1.0-alpha.2 | 任务清单读写 API，契约事实源 |
| CLI | Rust，clap，crates.io | 0.1.0-alpha.3 | AI/脚本入口：lists/tasks/add/update |
| Site | React + Vite | 0.1.0-alpha.3 | 介绍页，通向 Studio |

## 领域模型

最小模型：**TaskList（一个业务一个清单）→ Task（一个事项）**。Task 六字段：id、title、description、status（notStarted/inProgress/reviewing/done）、priority（urgent/high/medium/low，AI 建议 + 人确认）、category（业务自定义分类，自由字符串）。

建模要点：

- **状态自由流转**：`moveTo` 取代早期的「只前进」约束——执行允许回退重做；服务端不校验状态机，执行语义由客户端承载
- **清单与分类分离**：清单是业务隔离单元，category 是清单内「看分组」的次级视角
- **契约单一事实源**：JSON 形状在 Provider 定义，Studio（Dart）/ CLI（Rust）各自镜像，无派生逻辑

## 工程实践

质量门禁按组件各自执行：Studio 用 `flutter analyze` 与 `dart format`，Provider 用 `go vet` 与 `go test`，CLI 用 `cargo clippy` 与 `cargo test`。

部署与运维遵循平台统一命名：API 经系统级网关 `https://api.quanttide.com/qtcloud-execute` 暴露，应用层零 CORS（对齐 qtcloud-delib 规范）；Studio 部署 `studio.execute.cloud.quanttide.com`（原 `execute.cloud.quanttide.com` 移交介绍站）；API 基地址经 `--dart-define=QTCLOUD_EXECUTE_API_BASE_URL` / 环境变量注入。基础设施 Terraform 管理（state 在 `quanttide-terraform-state/qtcloud-execute/terraform.tfstate`）。发布按 scope 分离：`deploy-studio.yml` / `deploy-provider.yml` / `deploy-site.yml` / `release-cli`（多平台二进制 + crates.io）。

## 当前状态

组件处于 v0.1.0 对齐阶段，2026-08-23 一天内完成核心链路：Provider 上线 FC（`/health`、`/api/lists` 正常），Studio 完成真看板重构并接入服务端 API（beta.3 移除内置种子与本地持久化），CLI 重写为纯服务端客户端并发布 crates.io，Site 上线介绍页。组件成熟度不均衡：Studio 迭代最多（beta.1→beta.4 四轮），Provider/CLI 刚达可用。

## 演化路线图

- Provider：Service 层完善与输入校验（当前 handler 直接操作 Repository）；任务数据从单 JSON 文档演进为更细粒度持久化（清单规模增长后）
- Studio：AI 建议流（优先级「AI 建议 + 人确认」建模已预留，交互未落地）
- CLI：跟随契约演进，保持薄客户端
