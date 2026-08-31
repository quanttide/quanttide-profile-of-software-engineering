# qtcloud-asset Provider 领域模型

## 一句话概括

Provider 的世界里只有几样东西：**桶**、**桶里的文件**、**看文件的人**、**分享链接**。系统只做「看」，不做「改」。

## 实体关系图

```
User（人）
 │
 ├─ 登录后得到 → Session（会话）
 │
 ├─ 查看 → Bucket（桶）── 包含 ──→ Object（文件）
 │
 ├─ 创建 → Share（分享，用 Token 标识）
 │
 └─ 每次操作 → AuditLog（审计记录）
```

## 桶和文件：一切的核心

系统展示的东西只有两级：`Bucket`（桶）和 `Object`（桶里的文件）。

```go
// 一个 OSS 桶，比如 qtcloud-asset-studio
type Bucket struct {
    Name         string // 桶名
    Region       string // 区域
    StorageClass string // 存储类型
    CreatedAt    string // 创建时间
}

// 桶里的一个文件，比如 docs/brd/index.md
type Object struct {
    Key          string // 文件路径
    Size         int64  // 大小
    Type         string // 类型
    StorageClass string
    LastModified string
}
```

查询文件列表由 `ListObjectsParams` 控制：前缀过滤（只看 `docs/` 下的）、排序（按 key/size/date）、分页（游标翻页）。

怎么拿到这些数据？通过 `OssAdapter`——它只实现「读」能力，接阿里云 OSS。系统为将来接入更多资产源（文件系统、GitHub、飞书）预留了 `SourceAdapter` 接口，并按能力拆成五个窄接口：列桶、读桶权限、列文件、读文件、生成链接。想新增一个资产源，实现这几个接口即可。

## 人：谁能看什么

用户模型由角色和状态控制。

```go
type Role string // 角色
const (
    RoleViewer Role = "viewer" // 只能看普通资产
    RoleAdmin  Role = "admin"  // 能管理用户、访问私密资源
)

type User struct {
    ID       string
    Account  string
    Name     string
    Role     Role
    Status   UserStatus // active 正常 / disabled 停用
    PasswordHash string // 只存哈希，不存明文
}
```

登录后服务端保存 `Session`（记录过期时间、IP、浏览器），不再每次校验密码。外部身份（SSO）走 `IdentityProvider` 接口，没配好时用一个占位实现。

存储用接口 `UserStore` 隔离：本地开发用内存实现，生产用 Postgres 实现，API 层不关心数据放哪。

## 分享：把文件打包给人看

「分享」是一个公开只读链接，指向一个桶里的一组文件夹或文件。

```go
type Record struct {
    Token     string   // 链接标识，随机生成
    Title     string
    Bucket    string
    Prefixes  []string // 分享哪些文件夹，如 ["docs/", "examples/"]
    Keys      []string // 或分享哪些具体文件
    CreatedBy string
    CreatedAt time.Time
    RevokedAt *time.Time // 撤销时间，非空表示已作废
}
```

例：把 `qtcloud-asset-studio` 桶里的 `docs/` 和 `examples/` 打包，生成一个只读分享页，对方可单文件下载或 ZIP 下载全部。数量有硬约束：最多 32 个文件夹前缀、128 个文件、标题 120 字。存储同样是接口双实现：测试用内存，生产用 RDS。

## 审计：谁在什么时候做了什么

每次敏感操作记一条 `AuditLog`：谁（UserID）、什么动作（Action）、对哪个对象（Target）、结果（成功/拒绝/失败）、IP、时间。动作枚举覆盖登录、登出、列桶、列文件、生成链接、创建/撤销分享、用户管理等十七类。日志支持多路存储（`MultiAuditLogStore`），生产输出结构化日志经 SLS 查询。

## 两条贯穿全系统的规则

- **只读**：所有模型只有「发现、展示、分享」能力，没有写 OSS 的能力，这是刻意的边界
- **私密桶隔离**：桶名以 `-private` 结尾或叫 `quanttide-terraform-state` 的是私密桶——`viewer` 根本看不见，`admin` 能看见但不能生成访问链接；公开桶的链接永久有效（`ExpiresIn=0`）

## 与数字资产治理标准对照

对照数字资产治理标准（quanttide-asset `docs/specification`）：大部分概念能找到标准对应，少数需要补字段或重构。

### Bucket 与 Object

`Bucket`（name/region/storage_class/created_at）与 `Object`（key/size/type/storage_class/last_modified）都是资产，对应标准的资产模型，只是不同层级、不同类型：Bucket 是容器层资产（`type: OSS`），Object 是内容层资产（key 即其唯一标识）。

| 代码字段 | 标准字段 | 关系 |
|:--|:--|:--|
| Bucket.Name / Object.Key | name | 一致：key 即资产标识 |
| Bucket.CreatedAt / Object.LastModified | created_at / updated_at | 一致 |
| Bucket.Region / StorageClass、Object.Size / Type / StorageClass | — | 标准未定义，保留为连接器私有属性 |
| — | id | 缺口：代码无唯一标识 |
| — | title / description | 缺口：代码无资产标题与描述 |
| — | category | 缺口：代码的用途分类（studio/private/site）是枚举逻辑，应落为 category |
| — | tags | 缺口：无标签 |

层级关系（桶包含对象）标准资产模型未定义 parent 字段，可用 `tags`（如 `parent: <桶名>`）或注册结构表达。

### SourceAdapter 与 OssAdapter

对应标准的「连接器/适配器」概念——`type` 对应不同连接器/适配器（S3、Git 等）。`OssAdapter` 就是 OSS 连接器的实现，GitHub、飞书适配器即标准多源的扩展。五个窄接口（BucketLister/ObjectLister 等）是连接器的实现细节，标准不定义到接口级，不冲突。

### 隐私边界（isPrivateBucket）

硬编码在 `internal/repository/oss.go` 的私密桶判定（`-private` 后缀、`quanttide-terraform-state`），对应标准的 `category` 用途——「用于结构化分类和权限控制」。标准建模是资产属性驱动权限，现状是代码函数驱动，是重构点。

### User / Role / Session 与 Share

标准无对应：资产治理标准不定义身份模型，身份是正交的能力层；分享是资产治理之上的应用能力（把资产打包给人看），两者都保持独立建模。

### AuditLog

对应标准生命周期「维护」——通过自动化审计与人工干预确保元数据与物理实体一致。现状的 AuditAction 枚举覆盖登录、列桶、分享等操作级审计；标准行为级动作（注册、验证、下线）尚未进入枚举，是扩展点。

### 结论

对得上的：SourceAdapter/连接器、审计（维护）。

对不上的：Bucket/Object 缺 id/category/tags、隐私边界硬编码→资产属性。

升级动作的优先级按 index.md 演化路线图展开。
