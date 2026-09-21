# qtcloud-work CLI 软件工程档案

qtcloud-work CLI 是量潮知识工作云的命令行工具，crate 名 `qtcloud-work-cli`，二进制名 `qtcloud-work`，Rust 编写，当前版本 0.1.0-beta.2，发布于 crates.io。它把知识工作做成可执行的编排：工作流定义干什么，工单记这一趟怎么走，判据判算不算完。源码位于 quanttide-work 仓库 `apps/qtcloud-work/src/cli`。

## 定位

本地文件驱动：读工作区里的 YAML 与 Markdown、写账本、跑判据，不启服务、不连数据库、没有后台进程。对 provider 只保留探活（`health`，`GET /health`，缺省基地址 `https://api.quanttide.com/qtcloud-work`），正式接口层等就位后再接。

写入型动作都可以先预演（`--dry-run` 只说要写什么、不落盘）；每个动作支持 `--json` 与 `--out <文件>`。`--json` 统一为结果信封：`ok` / `lines` / `columns` / `rows`，原文托在 `data`。AI 与脚本是第一等调用方，信封与给智能体的话术（`prompts`，随带已被裁决的流水）都按这个前提设计。

## 命令面

命令动词式，与规格端点表一一对应：端点表里没有的操作，命令行里也没有。

| 组 | 命令 | 说明 |
|:--|:--|:--|
| 工作区 | `search` `catalog` `audit` `material` | 按名找文档、列目录、审计资产、看材料四字段 |
| 工作流 | `workflow create / show / list / check / export / import` | 定义是 YAML；`check` 核对判据路径与描述点到的小节 |
| 工单 | `order create / show / list / next / done / journal / delete` | 开单、走一步、闸门放行、写日志、销白纸 |
| 导览 | `help [<话题>]` | 按用途列出命令 |
| 探活 | `health` | provider 探活，`--json` 透传服务端响应 |

## 位置与账本

位置不进模型，全部由启动参数装载，优先级为命令行、环境变量、缺省：`--root` 工作区根（缺省向上找第二大脑）、`--data` 账本、`--workflows` 工作流目录、`--artifacts` 产物落点。

账本归 CLI，落 `$XDG_DATA_HOME/qtcloud-work/workspaces/<工作区键>/`，含工作区身份 `workspace.yaml`（首跑生成，`id` 供凭证派生）、领域事件 `events.jsonl` 与工单 `workorders/<工单>.yaml`。产物归工作区，落 `<工作区根>/artifacts/`，内容谁写谁定，程序只管落点。

## 领域模型

`src/` 顶层按聚合、领域服务、适配三类落位。聚合六个：工单（封面加工作记录流水，进度与完结由定义加流水推导，闸门不落字段）、工作流（YAML 定义与核对）、目录、资产表与产物实例、材料、工作区（装载、落点、核对、流水判定）。领域服务两个：`search` 按名找文档，`audit` 跑机械判据。适配是入口与边界：clap 命令树、`help` 导览、`prompts` 话术、`health` 探活。

判据挂在步骤上，按谁判分三类：`rule` 程序按 `path` / `absent` / `file` 加 `contains` / `run` 当场核，`agent` 由智能体照判准审，`human` 进待拍板清单等人 `order done` 放行。一步算过，等于所有 `rule` 通过、所有 `agent` 判过。

凭证分层：定义不写 `id`，工作流凭证按「工作区 id + 名字」派生（uuid5，命名空间钉死，各平台一致）；工单与工作记录的凭证由程序发。

## 工程实践

质量门禁三条：`cargo fmt --check`、`cargo clippy --all-targets --locked -- -D warnings`、`cargo test --locked`；`src/` 下单文件超 250 行即红。测试按用例组织，一个场景一个文件，测试上标 `// 用例：N`，由脚本与使用指南的用例对账。

发布走 `qtcloud-devops`：`release audit` 预检、`release publish` 建标签与 Release；推 `cli/v*` 标签触发 `release-cli` 工作流，校验版本与变更记录后构建三平台二进制，挂 GitHub Release 并发布 crates.io。

## 演化

provider 正式接口层就位后另起一轮接入；命令面跟随规格端点表演进。
