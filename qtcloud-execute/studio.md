# qtcloud-execute Studio 领域模型

qtcloud-execute Studio（`qtcloud_execute_studio`，Flutter Web，v0.1.0-beta.4）是量潮执行云的看板客户端：清单（业务）× 任务的泳道看板。源码位于 quanttide-execute `apps/qtcloud-execute/src/studio`。

## 核心模型

领域模型只有两个实体，全部定义在 `lib/models/`：

### Task（任务）

执行云的最小领域单元——一个事项。

| 字段 | 类型 | 说明 |
|:--|:--|:--|
| `id` | String | 唯一标识 |
| `title` | String | 标题，一句话概括 |
| `description` | String | 描述，展开细节 |
| `status` | TaskStatus | 状态（看板泳道列） |
| `priority` | TaskPriority | 优先级 |
| `category` | String? | 业务自定义分类（自由字符串，不枚举约束） |

**TaskStatus**（泳道列）：`notStarted` 未开始 / `inProgress` 进行中 / `reviewing` 评审中 / `done` 已完成。四态在任意方向自由流转（`moveTo`），推进与回退均合法——真看板语义，任务可能被退回重做。早期版本（beta.1）的 `advanceTo`「只前进」约束已移除。

**TaskPriority**（四档）：`urgent` 紧急 / `high` 高 / `medium` 中 / `low` 低。建模原则是「AI 建议 + 人确认」——AI 不直接修改（判断不是事实）。

### TaskList（清单）

一个业务一个清单。字段：`id`（业务标识，如 qtdata / qtclass / qtcloud）、`name`（业务名称）、`tasks`（任务数组）。任务直接属于清单，无分组层级；业务自定义分类走 `Task.category` 作为「看分组」的次级视角，与清单切换分离。

## 读写边界

`lib/repositories/task_repository.dart` 定义 `TaskRepository` 抽象（DDD——内存 / 服务端 API，可注入）：

- `InMemoryTaskRepository`：测试注入
- `ApiTaskRepository`：运行时注入，接入 provider API，服务端持久化（服务端从 OSS `data/tasks.json` 读取）

接口方法：`loadLists()`（动态加载全部清单，不静态假设）、`loadTasks(listId)`（看板投影数据源）、`updateTask(listId, task)`（同 id 替换，不存在则新增）、`save()`（API 实现随 updateTask 落盘，无操作）。错误模型为 `ApiException`（非 2xx，携带状态码与服务端信息）。

## 状态与交互层

状态管理用 flutter_bloc，两级 Cubit，均构造注入仓储、页面只消费不创建：

- `TaskListCubit`：清单列表 + 当前清单 id；`loadLists()` 动态加载（不清零仍存在的 currentListId），`switchList(id)` 切换看板跟随
- `BoardCubit`：当前清单的任务与更新

交互模型是纯展示层的：左侧清单导航（名称 + 任务数徽章）、类别过滤器（「全部」+ 各 category）、WIP 上限徽章（进行中/评审中超限标红）、任务详情走弹窗（`TaskDetailDialog` 就地操作，不跳页）。路由仅 `/` 一个页面（go_router），无 `/tasks/:id`。

## 服务端契约

- API 基地址经 `--dart-define=QTCLOUD_EXECUTE_API_BASE_URL` 注入（部署流水线 repo 变量），生产走系统级 API 网关 `https://api.quanttide.com/qtcloud-execute`
- 应用层零 CORS——跨域由系统级网关统一处理（对齐 qtcloud-delib 规范）
- 客户端域名 `studio.execute.cloud.quanttide.com`（原 `execute.cloud.quanttide.com` 移交站点）

## 建模要点

- **状态语义即业务语义**：`moveTo` 取代 `advanceTo` 不是实现细节，而是对「执行」的定义——执行允许回退重做，任务状态不是单向流水线
- **清单与分类分离**：清单是业务隔离单元（一次一个清单），category 是清单内的次级分组视角，两者不混用
- **镜像边界清晰**：客户端模型与 provider API 的 JSON 形状一一对应（`fromJson`/`toJson` 全字段），无客户端派生逻辑
