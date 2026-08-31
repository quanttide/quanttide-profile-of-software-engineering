# 学习管理领域模型

quanttide-learn-toolkit 承载的学习管理领域模型，JSON 契约定义。分三层：**标准实体**（Learner × Completion，已由学习云 provider 实现）、**核心领域模型**（Schedule + Task，设计完成未落码）、**档案视图**（Organization → Member 四区，设计中）。

## Learner（学习者）

学习管理领域的学习者主体。

### 字段

- `id`: 必选，本领域学习者 ID（uuid）。
- `user_id`: 可选，预留，关联 auth 领域用户 ID。

### JSON 示例

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "6f9619ff-8b86-d011-b42d-00cf4fc964ff"
}
```

## Completion（完成记录）

Learner 与课程域验收标准的交叉记录，记录通过状态。

### 字段

- `id`: 必选，主键（uuid）。
- `learner_id`: 必选，→ Learner.id。
- `criterion_id`: 必选，→ 课程域 Criterion.id（同源直连，本领域不设验收标准实体）。
- `status`: 必选，`completed` / `not_completed`，二元枚举。
- `created_at`: 可选，创建时间。
- `updated_at`: 可选，更新时间。

### JSON 示例

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "learner_id": "550e8400-e29b-41d4-a716-446655440000",
  "criterion_id": "cri-vibe-lesson1-zed",
  "status": "completed",
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-01-01T00:00:00Z"
}
```

## Schedule（学习路径）

学习管理体系的核心领域模型——Task 的有序集合，探路者—跟随者结构的枢纽。

### 字段

- `id`: 必选，唯一标识。
- `title`: 必选，名称。
- `description`: 可选，描述。
- `tasks`: 必选，Task 有序数组；顺序、依赖、分组经数组次序与 Task 间关系表达，不引入额外层级实体。

### JSON 示例

```json
{
  "id": "schedule-agent-engineer",
  "title": "智能体工程师训练营",
  "description": "按学习路径推进的训练计划，主线是建设数据工程第二大脑。",
  "tasks": [
    {
      "id": "task-data-second-brain",
      "title": "熟悉数据工程第二大脑",
      "description": "以新人视角通读领域第二大脑，记录困惑并提出改进建议。达成：至少一条建议经 Issue 讨论达成共识后进入 PR。"
    },
    {
      "id": "task-data-intention",
      "title": "整理数据工程意图",
      "description": "从数据工程日志考古组织意图，增加最新想法，描绘量潮为什么想要建设数据工程第二大脑。"
    }
  ]
}
```

## Task（任务）

路径上的一个节点——「学什么、做什么、做到什么程度算过」的最小单位，自足、自描述。

### 字段

- `id`: 必选，唯一标识。
- `title`: 必选，标题。
- `description`: 必选，验收判定写进描述（做到什么程度算过），不引用外部概念。
- `source`: 可选，来源标注（从哪位探路者的哪段实践提炼）。

### JSON 示例

```json
{
  "id": "task-data-second-brain",
  "title": "熟悉数据工程第二大脑",
  "description": "以新人视角通读量潮数据工程领域第二大脑（quanttide-data 主仓库、资产图式章程、发布管理章程），记录「知识诅咒」视角下的困惑，并提出改进建议。达成：至少一条建议经 Issue 讨论达成共识后进入 PR。",
  "source": "源自学习日志提炼"
}
```

## Organization（组织）

任意粒度的群体抽象（学校、班级、校企合作项目、训练营），是数据边界与权限边界。

### 字段

- `id`: 必选，唯一标识。
- `name`: 必选，名称。
- `type`: 必选，`school` / `class` / `enterprise` / `camp`。
- `courses`: 可选，课程引用数组。

### JSON 示例

```json
{
  "id": "org-camp-bigdata",
  "name": "大数据训练营",
  "type": "camp",
  "courses": ["course-production-internship"]
}
```

## Member（成员）

学员，档案挂在所属组织下。`id` 与 Learner.id 同源。

### 字段

- `id`: 必选，→ Learner.id。
- `org_id`: 必选，→ Organization.id。
- `name`: 必选，姓名。
- `role`: 必选，`student`。

### JSON 示例

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "org_id": "org-camp-bigdata",
  "name": "iGuo",
  "role": "student"
}
```

## Record（学习轨迹）

课时进度记录，档案含金量最低的一区。

### 字段

- `lesson_id`: 必选，课时标识。
- `status`: 必选，`in_progress` / `passed` / `failed`。
- `acceptance`: 可选，验收方式（自检类 / 真实反馈类）。
- `branch_path`: 可选，场景分支路径数组。
- `duration_minutes`: 可选，耗时（分钟）。

### JSON 示例

```json
{
  "lesson_id": "lesson-qtdata-intro",
  "status": "passed",
  "acceptance": "self-check",
  "branch_path": ["a1", "b2"],
  "duration_minutes": 45
}
```

## Assessment（考核记录）

作业、测验、考试的评分与评级。

### 字段

- `type`: 必选，`assignment` / `quiz` / `exam`。
- `score`: 必选，分数。
- `grade`: 可选，评级。
- `passed`: 必选，是否通过。

### JSON 示例

```json
{
  "type": "assignment",
  "score": 92,
  "grade": "A",
  "passed": true
}
```

## Artifact（产出物）

真实产出（PR、Release、作品仓库、证书），含链接与状态，档案含金量最高的一区。

### 字段

- `type`: 必选，`pr` / `release` / `repo` / `certificate`。
- `url`: 必选，产出物链接。
- `status`: 必选，`merged` / `approved` / `rejected`。

### JSON 示例

```json
{
  "type": "pr",
  "url": "https://github.com/quanttide/quanttide-data/pull/1",
  "status": "merged"
}
```

## 关系与边界

```
Learner ──1:N──▶ Completion ◀──N:1── Criterion（课程域，同源直连不建实体）
Organization ──1:N──▶ Member ──1:N──▶ 档案四区（Record / Assessment / Artifact / 画像）
Schedule ──1:N──▶ Task
```

- **同源直连**：Criterion 由课程域一等实体承载，本领域只保存 `criterion_id` 跨域引用。
- **状态二元**：Completion.status 仅 `completed` / `not_completed`，不提前引入三态。
- **核心模型只建两个概念**：Schedule（聚合根）+ Task（节点）；Task 无外键、无跨域引用，跨域对齐是应用层职责。
- **Journal / Profile 不是领域模型**：是数据视图（沉淀层 / 快照层），不进入 JSON 契约。
- **档案四区含金量从低到高**：学习轨迹 → 考核记录 → 产出物 → 能力画像（由前三者推导，未单列实体）。
