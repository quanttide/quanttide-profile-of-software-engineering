# qtclass Site 领域模型

## 一句话概括

Site 是纯静态展示站：没有后端模型，领域模型由三块静态数据拼成——**课程体系卡片**（Home 硬编码）、**生产实习课程**（`courses.ts` 类型化 Course/Chapter/Lesson + 教案 markdown）、**学习档案**（`learning.ts` 扫描 `schedules/` 与 `tasks/` 目录的 markdown 副本）。三者都是「文件即数据」：数据长在源码目录的 `.ts` 与 `.md` 文件里，运行时用 `import.meta.glob` 按文件路径装载。

## 课程体系：Home 卡片（硬编码）

`pages/Home.tsx` 内联数组，五门课各一张卡片，字段为 `name / level / audience / summary / modules[]`：

| 课程 | 层级 | 模块 |
|:--|:--|:--|
| 知识工作 | 入门 | 文档即资产 / 信息收集与整理 / 知识输出 |
| 氛围编程 | 入门 | 开发环境搭建 / 工具配置 / 任务拆解 |
| 大数据导论 | 进阶 | 数据资产 / 分析流程 / 业务指标 |
| 数据工程 | 高阶 | 数据建模 / 工程管道 / 质量与运维 |
| 生产实习 | 实训 | 理解业务 / 识别机会 / 验证 Demo / 提交立项 |

只有「生产实习」带 `slug`，整卡可点击跳转课程页；其余四门为纯展示卡片。这套数据与 `courses.ts` 的类型化课程模型并存但**未打通**——是同一领域概念的两份手写清单。

## 课程内容模型：Course / Chapter / Lesson

`data/courses.ts` 是唯一类型化模型，三级树：

```ts
interface Lesson { id: string; title: string; slug: string }
interface Chapter { id: string; title: string; lessons: Lesson[] }
interface Course  { id: string; title: string; slug: string; description: string; chapters: Chapter[] }
```

目前只有一门课 `productionInternship`（生产实习），4 章 8 课时：

| 章节 | 课时 |
|:--|:--|
| 量潮数据 | 经营现状、业务模式 |
| 量潮课堂 | 课程简介、销售指南、经营目标 |
| 量潮云 | 产品简介 |
| 量潮招聘 | 招聘流程、发送准入问卷 |

课时正文是 `data/lessons/<slug>.md` 文件，`import.meta.glob('../data/lessons/*.md', { eager, query: '?raw' })` 整包装载，slug 与文件同名即「目录即路由」。课程页按章节渲染课时列表并全局编号；详情页以 slug 查课时信息，正文经 `markdownToHtml` 渲染，缺课时回退「课程内容正在编写中…」。导航模型是扁平化的：`chapters.flatMap(lessons)` 得到全课时序列，上一课/下一课按全局序号前后切换。

## 学习档案模型：LearningItem / schedules / tasks

`data/learning.ts` 承载「学习」页（`/learn`），分两个目录：

- `data/learning/schedules/` —— 训练营（Schedule）：智能体工程师训练营、产品经理训练营
- `data/learning/tasks/` —— 任务（Tasks）：8 个协作任务（熟悉数据工程第二大脑、贡献学习档案、文档工程章程升级等）

```ts
interface LearningItem { slug: string; title: string; description: string }
```

`itemsIn(dir)` 用 `import.meta.glob('../data/learning/**/*.md')` 扫描目录，slug 取文件名；title/description 优先读 markdown 的 YAML frontmatter（`---\ntitle: …\ndescription: …\n---`），无元数据时回退：title 取首个 `# ` 标题，description 取标题后第一段正文（跳过代码块/列表/引用/表格，去内联标记）。按标题做中文 localeCompare 排序。

数据来源是**静态副本**：注释明确「源自 data/profile 子模块（schedules/ 训练营、tasks/ 任务），同步方式为手工复制 `*.md` 到 `src/data/learning/` 对应目录」。训练营详情页有交叉引用富化：正文中 `来源：tasks/<slug>.md` 文本被替换为指向 `/learn/tasks/<slug>` 的可点击链接（`enrichTaskRefs`）。

## 渲染与路由模型

渲染是共享工具：`utils/markdown.ts` 的 `markdownToHtml` 自写转换器，支持标题/段落/有序无序列表/代码块/引用块/表格/分隔线与内联加粗斜体代码链接，课程详情页与学习档案详情页共用同一份实现。

路由（`App.tsx`）两级导航「课程 / 学习」：

| 路由 | 页面 | 数据 |
|:--|:--|:--|
| `/` | Home | 五门课卡片（硬编码） |
| `/learn` | Learn | schedules + tasks 目录扫描 |
| `/learn/:type/:slug` | ItemDetail | `type ∈ {schedules, tasks}`，目录内找 slug |
| `/courses/production-internship` | ProductionInternship | courses.ts 章节列表 |
| `/courses/production-internship/lessons/:lessonSlug` | ProductionInternshipCourse | 教案 markdown + 前后课时 |

详情页找不到内容统一走 404 态（「内容未找到」+ 返回链接）。`/learn/:type/:slug` 对 type 做白名单归一：非 tasks 一律按 schedules 处理。

## 结论

Site 的建模方式是「文件即数据」，三块内容各自为政，尚未收敛：

- **课程体系卡片**与 **courses.ts** 是同一领域概念的两份手写清单（前者无类型、后者单课程），存在合并空间：卡片数据可下沉为 courses.ts 的展示字段。
- **学习档案**依赖手工复制 `data/profile` 的静态同步，副本与事实源（quanttide-learn `data/profile`）之间无版本对齐机制。
- **课程内容**（生产实习教案）是仓库内静态 markdown，与课程云的内容实体（Course/Lesson/Scene，见 `docs/dev-guide/provider.md`）未打通——Site 是纯展示，Studio 已走 API 拉取，二者内容通道尚未统一。
- 无前端依赖的运行时模型（无登录、无状态、无后端调用），v0.2.0 起路线图引入搜索、进度跟踪与登录后，静态文件模型需要让位给真正的数据层。
