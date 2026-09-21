# qtcloud-work CLI 模块分析

对象是 quanttide-work 仓库 `apps/qtcloud-work/src/cli` 的源码结构，基准取 qtcloud-work `852f48f`（工作区干净，版本 0.1.0-beta.2）。静态分析：行数与引用关系取自当前工作区，`src/` 共 48 个文件、5009 行；`tests/` 不在内。定位与命令面见[工程档案](./index.md)，分层规矩见该仓 `CONTRIBUTING.md`。

## 规模与落位

`src/` 顶层按三类落位，外加入口与跨聚合中立件，行数分布如下：

| 落位 | 文件 | 行数 | 占比 |
|:--|--:|--:|--:|
| 聚合（order / workflow / catalog / artifact / material / workspace） | 23 | 2905 | 58% |
| 跨聚合中立件（criterion / outcome / error / executor / paths / fields / ids / clock / sha1 / events） | 13 | 1029 | 21% |
| 适配（`cli.rs`、`cli/`、`help`、`prompts`、`health`） | 9 | 825 | 16% |
| 领域服务（search / audit） | 2 | 222 | 4% |
| 入口（`main.rs`，28 行，只做声明与转交） | 1 | 28 | 1% |

聚合占近六成。聚合内部体量最大的是 order（8 件 1088 行，模型 / 账本 / 动作 / 执行 / 交给 AI 各一件）与 workflow（5 件 680 行，定义 / 读法 / 动作 / YAML 各归其位）。

## 依赖方向

入口层零被引：`src/` 下没有任何模块引用 `crate::cli`，这条由 `tests/contract.rs` 的「动作层不依赖入口层」全文件扫描钉住。服务依赖聚合方向合规：`search → catalog`，`audit → catalog / artifact / workspace`。

越线一处：聚合依赖服务。`order/execute.rs` 四处调用 `crate::audit::items_of / run`（47–48、136–137 行），而 CONTRIBUTING 写明「聚合不得依赖服务」；contract.rs 只钉了入口层方向，这条没有钉子。两条出路：把判据执行从 `audit` 下沉为中立件，或在规矩里为「工单执行要跑判据」开口子。

聚合间有两个环，都经过 workspace：workspace ↔ order（`workspace/locate.rs`、`progress.rs` 引 `crate::order`，order 侧六个文件引 `crate::workspace`）、workspace ↔ workflow（`workspace/model.rs` 等三件引 `crate::workflow`，workflow 侧三件引回）。规矩未禁聚合互引，Rust 同 crate 内也编得过，但这是仅有的两处双向耦合，拆分时优先收窄 workspace 的「场所」职责。

被依赖热度：workspace 最广（15 个文件引用），其后 workflow（9）、artifact 与 order（各 5）、catalog（4）；material 只被一个 handler 引用，是最孤立的聚合。中立件里 outcome 最热（12 个文件），其后 criterion（8）、executor（7）、ids（6）。

## 行数与阈值

单文件 250 行红线全部达标，最大件 `material/mod.rs` 240 行。贴线四件：`material/mod.rs` 240、`workflow/mod.rs` 234、`order/mod.rs` 214、`catalog/mod.rs` 210，其中三件是 `mod.rs`——按规矩「mod.rs 只留类型与出口」，这 200 多行都是类型与出口本身，聚合的模型在变厚，下一轮新增即触发拆分。适配层最大件是 `cli/commands.rs`（205 行，clap 命令树）。

## 规矩对账

| 规矩 | 现状 | 判定 |
|:--|:--|:--|
| 单文件 ≤250 行 | 最大 240 行 | ✓ |
| 聚合与服务不依赖入口层 | 零被引，contract.rs 钉住 | ✓ |
| 聚合不依赖服务 | `order/execute.rs` 调 `crate::audit` 四处 | ✗ |
| 命名用能力名，不造 `-er` | search、audit，无反例 | ✓ |
| 测试按用例组织，用例号对账 | 15 个用例文件，`validate-usecases.sh` 绿 | ✓ |
| 约定文档与代码同调 | CONTRIBUTING 还写 `task/` 聚合与旧测试名（`agent_step`、`task_start`），实际是 `order/` 与 `order_start` 等 | ✗ |

## 小结

结构与自述基本相符：三类落位、250 行红线、入口隔离、用例对账都成立，聚合占六成的重心也符合「领域逻辑在聚合、入口薄」的取向。待办两件：order 对 audit 的服务依赖，改代码或改规矩二选一；CONTRIBUTING 的命名漂移，随 beta.2 的 task → order 改名一并修订。两处聚合环都汇于 workspace，是后续重构的第一个观察点。
