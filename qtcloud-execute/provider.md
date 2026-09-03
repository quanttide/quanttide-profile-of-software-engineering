# qtcloud-execute Provider 领域模型

qtcloud-execute Provider（Go + 阿里云 FC 3.0，v0.1.0-alpha.2）是量潮执行云的服务端：任务清单的读写 API。源码位于 quanttide-execute `apps/qtcloud-execute/src/provider`。

## 核心模型

领域模型与 Studio 完全同构——`internal/task/repository.go` 是 JSON 契约的事实源：

### Task（任务）

| 字段 | 类型 | 说明 |
|:--|:--|:--|
| `id` | string | 唯一标识 |
| `title` | string | 标题，一句话概括 |
| `description` | string | 描述，展开细节 |
| `status` | string | `notStarted` / `inProgress` / `reviewing` / `done` |
| `priority` | string | `urgent` / `high` / `medium` / `low` |
| `category` | *string | 业务自定义分类（可选，自由字符串） |

状态与优先级用常量定义（非枚举类型），注释标明「对齐 tasks.json 取值」。

### List / TaskList（清单与集合）

- `List`：一个业务一个清单（`id` / `name` / `tasks`），任务直接归属，无分组层级
- `TaskList`：数据文件顶层结构——`{"lists": [...]}`，文件不存在时返回空集合（不报错）

## 存储抽象

`internal/store` 定义 `Store` 接口（get / put / listDir / isDir，按相对路径操作），双实现：

- `LocalStore`：本地文件系统，默认数据源（数据文件直读，docker-compose 挂载 `data/tasks.json`）
- `OSSStore`：阿里云 OSS，生产数据源——运行时数据桶 `qtcloud-execute-provider`，配置经 FC 环境变量注入（`NewOSSStoreFromEnv`）

`Repository`（`internal/task`）组合 Store 与数据文件路径（`data/tasks.json`），load/save 全量读写——任务数据整体作为一份 JSON 文档，无逐条持久化。

## HTTP API

`internal/task/handler.go` 三个端点（经系统级 API 网关 `https://api.quanttide.com/qtcloud-execute` 暴露）：

| 端点 | 方法 | 说明 |
|:--|:--|:--|
| `/api/lists` | GET | 列出全部清单 |
| `/api/lists/{id}/tasks` | GET | 某清单任务（支持按状态过滤） |
| `/api/lists/{id}/tasks/{task_id}` | PUT | upsert 任务（同 id 替换，不存在则新增） |

另有 `/health` 健康检查。错误语义：清单不存在返回 `ErrListNotFound`（映射 404）。

## 部署与运维

- 多阶段 Dockerfile（静态构建 + alpine 运行），镜像推 ACR 后由 Terraform 部署到 FC 3.0（`custom-container` runtime）
- CI（`deploy-provider.yml`）：`provider/*` tag → 构建镜像 → Terraform apply；Terraform state 迁至 OSS 远程（`quanttide-terraform-state/qtcloud-execute/terraform.tfstate`）
- 数据文件随 updateTask 落盘到 OSS 桶——服务端无数据库，存储即一份 JSON 文档

## 建模要点

- **契约单一事实源**：Task/List 的 JSON 形状在 provider 定义，Studio 与 CLI 各自镜像（客户端 `fromJson` / Rust struct 一一对应），无派生逻辑
- **存储与领域分离**：`Store` 接口只管字节（get/put），`Repository` 负责 JSON 契约解析，领域层不感知数据源是本地文件还是 OSS
- **无任务状态机**：服务端不校验状态流转合法性（Studio 的 `moveTo` 自由流转语义），四态只是字符串取值——执行语义由客户端承载
