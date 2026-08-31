# 量潮课堂软件工程档案

qtclass 是量潮课堂应用，面向学员提供课程展示与课堂学习能力，由展示站（Site）、课堂管理与互动播放器（Studio）、集成层（Provider）三个组件构成，课程内容与学习记录分别由课程云（qtcloud-course）与学习云（qtcloud-learn）承载。

## 系统架构

```
Site (React SPA)              Studio (Flutter Web)
 课程体系 / 教案 / 学习档案      课堂管理 + 互动播放器
 （静态 markdown 副本）           │
                                 ▼
                         Provider (Go，集成层，规划中)
                            │              │
                            ▼              ▼
                    qtcloud-course    qtcloud-learn
                  （内容与标准事实源）    （完成记录）
```

Site 是纯静态展示站：课程体系、生产实习教案与学习档案（训练营/任务）以仓库内 markdown 呈现，不依赖后端；Studio 通过 Provider 聚合课程云内容并回写学习云完成记录，接入 qtcloud-auth 登录。Provider 不设独立存储，课程云定义内容与标准、学习云只记账，应用层零关联表（ID 同源直连，见 `docs/dev-guide/provider.md`）。

## 组件与技术栈

| 组件 | 语言与框架 | 职责 |
|:--|:--|:--|
| Site | React 19，TypeScript，Vite，React Router | 课程体系展示、生产实习教案、学习档案（训练营/任务）浏览 |
| Studio | Flutter | 课堂管理 + 互动播放器，学员端登录与进度/立项 |
| Provider | Go（规划中） | 学员端集成层：播放器数据组装、完成记录回写 |

## 课程体系

- **理论课**：微专业（如量潮大数据微专业）、班级（浙理班级、社会招生班级）
- **实践课**：实训基地（如量潮实训基地）、训练营（大数据训练营、CEO 助理训练营、VIP 训练营）
- **学生类型**：付费学员（免审核购买课程和实训）、VIP 学员（免审核任何课程）、免费学员（需通过选拔和阶段性考核）

## 工程实践

质量门禁按组件各自执行：Site 用 ESLint 与 `tsc -b`（构建前类型检查），Studio 用 `flutter analyze` 与 `dart format`，Provider 用 `go vet` 与 `go test`（质量门禁 job 前置，不过不构建镜像）。

部署与运维遵循平台统一命名：Site 部署 `class.quanttide.com`，Studio 部署 `studio.class.quanttide.com`；Provider 为阿里云函数计算（FC），Terraform 管理基础设施（`manifests/terraform/`，状态键 `qtclass/terraform.tfstate`）。发布管道按 scope 分离：`deploy-site.yml` / `deploy-studio.yml` / `deploy-provider.yml`，标签命名 `studio/vX.Y.Z`、`site/vX.Y.Z` 子包级与仓库级 `vX.Y.Z` 并存。

## 权限与安全模型

Site 为公开展示站，无登录；学员端登录在 Studio 侧：接入 qtcloud-auth password token 接口，token 持久化到 `shared_preferences`，请求自动携带 `Authorization: Bearer <token>`，缺 token 时引导登录。

## 当前状态

组件成熟度不均衡：Studio 最完整（v0.1.9，正式版，课堂管理与播放器已接入生产实习内容）；Site 处于预发布（v0.1.2-beta.3，课程体系 + 教案 + 学习档案）；Provider 规划中（Phase 1 服务骨架与 FC 部署管道完成，Phase 2 播放器数据组装完成，完成回写待落地）。Site 文档体系 v0.1.x 对齐阶段，BRD/PRD 待补齐。

## 演化路线图

Site 侧（`src/site/ROADMAP.md`）：v0.2.0 补课程内容展示、搜索、进度跟踪与登录；v0.3.0 加视频播放、作业、讨论区与证书；v1.0.0 成为完整课程学习平台。Provider 侧（`src/provider/ROADMAP.md`）：v0.1 完成骨架、播放器组装与完成回写，验收标准为「同源直连」——不建任何跨域关联表。
