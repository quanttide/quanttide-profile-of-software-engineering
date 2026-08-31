# quanttide-learn-toolkit TypeScript 领域模型

## 一句话概括

TypeScript 包承载**学习管理领域模型**，分三层：已落地的标准实体 **Learner × Completion**（对齐《量潮学习管理标准》，与学习云 provider 同一份模型定义）、洞察层核心模型 **Schedule（聚合根）+ Task（节点）**、以及档案视图 **Organization → Member 两级与档案四区**。仓库当前为脚手架（README 占位），本文件是从标准、洞察与档案设计中提炼的模型设计，尚未落码。

## 模型分层

学习管理领域的建模边界（对齐 `data/insight/schedule-core-model.md`）：

| 层 | 概念 | 归属 | 状态 |
|:--|:--|:--|:--|
| 标准实体 | Learner / Completion | 学习云（provider / CLI 已实现） | ✅ 已实现，唯一事实来源 |
| 核心领域模型 | Schedule / Task | 学习管理领域（洞察层） | 🚧 设计完成，未落码 |
| 数据视图 | Organization / Member / 档案四区 | 学习管理档案（profile） | 🚧 设计中 |

## 标准实体：Learner × Completion

TypeScript 类型，对齐《量潮学习管理标准》（`docs/specification/index.md`）与 provider `internal/domain/models.go`：

```ts
export interface Learner {
  id: string          // 本领域的学习者 ID（uuid）
  user_id?: string    // 预留：关联 auth 领域的用户 ID
}

export interface Completion {
  id: string
  learner_id: string    // → Learner.id
  criterion_id: string  // → 课程域 Criterion.id（同源直连）
  status: 'completed' | 'not_completed'
  created_at?: string
  updated_at?: string
}
```

关系：`Learner ──1:N──▶ Completion ◀──N:1── Criterion（课程域）`

关键约束：

- **同源直连，本领域不设 Criterion 实体**：验收标准由课程云一等实体承载，学习云只保存跨域引用——TypeScript 侧同样不定义 Criterion，`criterion_id` 作为字符串透传，不建本地标准实体。
- **状态二元**：`completed` / `not_completed`，与 provider `models.go` 与 CLI 完全一致，不提前引入三态。

## 核心领域模型：Schedule + Task

对齐 `data/insight/schedule-core-model.md`——学习管理体系的核心领域模型是 Schedule（学习路径），探路者—跟随者结构的枢纽：

```ts
export interface Schedule {
  id: string
  title: string
  description?: string
  tasks: Task[]     // 有序集合：顺序、依赖、分组经 Task 间关系表达
}

export interface Task {
  id: string
  title: string
  description: string  // 自足、自描述："学什么、做什么、做到什么程度算过"写进描述
  source?: string      // 来源标注：从哪位探路者的哪段实践提炼
}
```

设计约束：

- **只建两个概念**：Schedule（聚合根）+ Task（节点），不引入额外的层级实体。
- **Task 自足**：无外键、无跨域引用——验收判定写进 Task 自己的描述；学习材料不在模型里挂链接；跨域对齐是应用层职责，不是模型的职责。
- **Journal / Profile 不是领域模型**：是数据视图（沉淀层 / 快照层），承载、观测、投影 Schedule，但本身不被建模。

## 数据视图：学习档案（Organization → Member 四区）

对齐 `docs/dev-guide/learning-profile.md` 与 `data/profile/AGENTS.md`——档案是数据视图，不是被管理的领域对象：

```ts
export interface Organization {   // 任意粒度的群体抽象：学校/班级/校企项目/训练营
  id: string
  name: string
  type: 'school' | 'class' | 'enterprise' | 'camp'
  courses?: string[]
}

export interface Member {
  id: string
  org_id: string
  name: string
  role: 'student'
}

export interface Record {         // 学习轨迹
  lesson_id: string
  status: 'in_progress' | 'passed' | 'failed'
  acceptance: string
  branch_path: string[]
  duration_minutes: number
}

export interface Assessment {     // 考核记录
  type: 'assignment' | 'quiz' | 'exam'
  score: number
  grade: string
  passed: boolean
}

export interface Artifact {       // 产出物（需新增实体）
  type: 'pr' | 'release' | 'repo' | 'certificate'
  url: string
  status: 'merged' | 'approved' | 'rejected'
}
```

档案四区含金量从低到高：**学习轨迹 → 考核记录 → 产出物 → 能力画像**（skill profile，由前三者推导）。排序即价值观：招聘（竞赛模式）看产出物与画像，不看进度。组织是数据边界与权限边界：成员档案挂在自己的组织下。

成员档案另有 markdown 视图两层变体（`data/profile/AGENTS.md`）：**学员档案（验收制）**按 Schedule 的 Task 记完成状态，**自助学习者档案（主题制）**无 Schedule 驱动、按管理领域维护主题表——档案格式是内容视图，不是 API 实体，不进入 TypeScript 类型。

## 与实现对照

| 模型 | 实现状态 | 说明 |
|:--|:--|:--|
| Learner / Completion | ✅ qtcloud-learn provider（Go）与 CLI（Rust）已实现 | TypeScript 包做同一份模型定义（type alias），与学习云 provider 对齐，供 Studio/Site 复用 |
| Schedule / Task | 🚧 洞察层设计完成，未落码 | TypeScript 包可承担首个落地载体（对齐 course-toolkit 先例：内容实体 type alias + 路由常量） |
| Organization / Member / 四区 | 🚧 设计中（artifact / skill profile 需新增） | 当前档案是 markdown 视图（`data/profile/learners/*.md`），实体化待 provider 落地 |

## 结论

TypeScript 包是学习管理领域模型的事实源候选：标准实体（Learner/Completion）已有 Go/Rust 实现可对齐，核心模型（Schedule/Task）与档案视图（Organization/Member）尚未落码——本包是这两块模型的首个代码载体，落地顺序建议先实体（对齐 provider）再核心模型（Schedule/Task），档案视图随 provider artifact 实体化后再引入。命名约束：本领域不设 Criterion 实体，`criterion_id` 只透传；Completion 状态保持二元，不提前引入三态。
