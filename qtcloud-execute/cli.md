# qtcloud-execute CLI 领域模型

qtcloud-execute CLI（`qtcloud-execute-cli`，Rust，v0.1.0-alpha.3，发布于 crates.io）是量潮执行云的命令行客户端——已部署 provider（FC）的辅助入口，主要供 **AI/脚本** 调用。源码位于 quanttide-execute `apps/qtcloud-execute/src/cli`。

## 定位：纯服务端客户端

不读写本地文件，只通过 HTTP 对接已部署的 provider API。alpha.1 之前曾内置种子数据与本地文件持久化，重写后全部移除——运行数据完全依赖服务端（OSS `data/tasks.json`）。

服务端地址优先级：`--server` > 环境变量 `QTCLOUD_EXECUTE_API_BASE_URL` > 默认系统级网关 `https://api.quanttide.com/qtcloud-execute`。

## 数据模型

`src/main.rs` 三个 struct 镜像 provider 的 JSON 契约（serde 反序列化）：

- `Task`：id / title / description / status / priority / category
- `List`：id / name / tasks
- `ListsResp` / `TasksResp`：响应包装

枚举取值与 Studio/Provider 一致：状态 `notStarted` / `inProgress` / `reviewing` / `done`，优先级 `urgent` / `high` / `medium` / `low`。CLI 不建模状态机，只透传字符串。

## 子命令

| 命令 | 说明 | 对应 API |
|:--|:--|:--|
| `lists` | 列出全部任务清单 | `GET /api/lists` |
| `tasks <list_id> [--status <s>]` | 列出某清单任务，可按状态过滤 | `GET /api/lists/{id}/tasks` |
| `add <list_id> <title> [--description] [--priority] [--category]` | 新增任务（ID 由 CLI 生成） | `PUT` upsert |
| `update <list_id> <task_id> [--status] [--priority] [--description] [--category]` | 更新任务（先 GET 合并再 PUT 全量） | `PUT` upsert |

所有子命令支持 `--json`：直接透传服务端原样 JSON，供 AI/脚本程序解析——这是「主要供 AI 调用」定位的直接体现。

## 发布与 CI

- 发布管道 `release-cli`：`cli/*` tag → `check`（validate-version / validate-changelog 脚本）→ `build-binaries`（多平台矩阵）→ `attach-release`（GitHub Release 二进制）+ `publish-crate`（crates.io，`CRATES_API_TOKEN`）
- 契约登记 `cli` scope：`registry: crates`、`framework`、`release.changelog`、`test_threshold`

## 建模要点

- **零领域逻辑**：CLI 是 provider 契约的薄客户端，没有本地状态、没有派生模型——连任务 ID 生成都只是 uuid，语义（状态流转、分类）全部由服务端契约与客户端约定承载
- **AI 友好即产品需求**：`--json` 透传与环境变量配置是为一等调用方（AI/脚本）设计的接口形态
- **update 即合并**：`update` 先 GET 再 PUT 全量，遵循 provider 的 upsert 语义，不做局部补丁
