# qtcloud-work Studio 软件工程档案

qtcloud-work Studio 是量潮知识工作云的工作台，包名 `qtcloud_work_studio`，Flutter Web 编写，当前版本 0.1.0，是知识工作云的图形界面入口。源码位于 quanttide-work 仓库 `apps/qtcloud-work/src/studio`。一条底线：界面上出现的东西，命令行里都做得到；命令行里没有的动作，这里也不出现。

## 界面与状态

三个页面：任务（一次执行实例，左边对话、右边状态面板）、流程（一条定义，左边对话、右边步骤与定义两态）、设置（三处位置加 provider 探活）。页面里可复用的块住 `lib/views/`：侧栏、顶栏、对话、状态面板、步骤链、判据面板、定义态、执行者标签。

状态交给 Bloc：`states/workbench_bloc.dart` 一个文件装状态、事件与 Bloc。界面件只认传进来的领域对象与回调，拿数据、改状态都在 Bloc。

## 仓储

界面与 Bloc 只依赖 `repositories/studio_repository.dart` 接口，实现只有一套 `repositories/local/`：不起子进程、不依赖命令行，直接在工作区上算，结果与命令行一致。命令面收在 `dispatch.dart` 一处，`bin/qtcloud.dart`（Dart 命令行入口）与界面都走它。

三处位置不进模型，由 `repositories/local/run_context.dart` 从环境读：`QTCLOUD_WORK_ROOT`、`QTCLOUD_WORK_DATA`、`QTCLOUD_WORK_WORKFLOWS`。平台相关读写（文件、宿主、环境）按 io 与 web 双实现隔离。

## 领域模型

领域模型收回本仓自持，不再依赖 pub.dev 的 `quanttide_work` 包：`lib/quanttide_work.dart` 为出口，聚合 artifact、criterion、workflow、task、workspace 与横切件 outcome、error、executor、paths、fields，与 CLI 的 Rust 模型同构。「走过几步、下一步、进度、判据条数」由领域对象自己算，界面不留第二份模型；落点、流水判定与定义核对走工作区聚合的同一套算法。

## 对表

`scripts/parity.sh` 是与命令行的同构验收：同一处工作区、同一条命令，Rust CLI 与 Dart 各跑一次，比结果信封的 `ok`、`columns`、`rows` 与 `data`；给人看的 `lines` 不比，界面与命令行各写各的。

## 工程实践

质量门禁 `flutter analyze` 与 `flutter test`，CI 跑同样的两条；`test/` 与 `lib/` 同构分层，夹具取自命令行的真实输出。六个平台目录已初始化，当前以 Web 与桌面为主。

发布：推 `studio/vX.Y.Z` 标签触发 `release-studio.yml`，门禁过了再构建 Web 版（`--dart-define=APP_VERSION=<版本>`），上传 OSS 桶 `qtcloud-work-studio`（哈希资源长缓存、入口文件 no-cache）并刷新 CDN `work.cloud.quanttide.com`；改 studio 只跑门禁、不部署。发布线尚未开过第一版。

## 演化

对照软件工程契约与 Flutter 手册的待办：

- `lib/` 三个文件超 250 行（最长 `tasks.dart` 412 行），按契约触发转聚合；
- 路由未用 `go_router`，现由 `app.dart` 自切屏；
- 构建未自托管 CanvasKit（`FLUTTER_WEB_CANVASKIT_URL`），国内网络有白屏风险；
- 依赖许可清单未列（`flutter_bloc`、`yaml` 等）；
- `parity.sh` 的命令名还是 beta.2 之前的旧命令面，待更新后恢复对表。
