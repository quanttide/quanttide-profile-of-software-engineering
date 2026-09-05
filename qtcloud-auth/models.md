# qtcloud-auth 领域模型

qtcloud-auth Provider（Go，v0.1.0-alpha.9）的领域模型：身份（User）、角色（Role）、验证码（VerificationCode）三个持久化模型，加上无状态 JWT 令牌与可选的 OAuth 令牌存储。全部定义在 `src/provider/internal/`。

## 持久化模型

### User（用户）

统一身份主体，定义在 `internal/model/user.go`：

| 字段 | 类型 | 说明 |
|:--|:--|:--|
| `id` | string | UUID 字符串主键 |
| `username` | string | 用户名 |
| `email` | string | 邮箱 |
| `password_hash` | string | Argon2id 哈希（PHC 格式） |
| `phone` | string | 手机号 |
| `phone_verified` | bool | 手机是否已验证 |
| `nickname` | string | 昵称 |
| `avatar` | string | 头像 |
| `role_id` | string | 关联角色 |
| `created_at` | string | 创建时间 |

登录查找按「用户名/邮箱/手机号任一匹配」统一入口（`findUserByLogin`），密码登录与验证码登录共用。验证码登录未匹配到手机号时自动注册——注册即认证，不设独立激活流程。

### Role（角色）

权限的最小载体：`id`、`name`、`permissions`（字符串数组，GORM serializer:json，pg 存 jsonb / sqlite 存 text）。权限是自由字符串集合，无预定义权限表——RBAC 只到「角色持权限串」一层，ABAC/ReBAC 未建模。

### VerificationCode（验证码）

一次性凭证：`phone`、`code`、`expires_at`、`used`、`created_at`。发送频率限制与验证码同走共享存储（FC 多实例可用）；消费即标记 `used`，过期码不清理（ROADMAP 待办）。

## 令牌模型

访问令牌是 JWT HS256，无状态校验（签名 + exp），不落库；refresh_token 走共享存储。`internal/store/token_store.go` 实现 go-oauth2/oauth2 的 `TokenStore` 接口：access/refresh 双键、过期摘除、Create 幂等 upsert，刷新时轮换并作废旧令牌。

## 持久化抽象

`internal/api/handler.go` 定义 `Storer` 接口（`List/Create/Get/Update/Delete`，按集合名组织），`internal/store` 提供唯一 GORM 实现。数据库方言由 `internal/app.OpenDB()` 按 `DB_DRIVER` 切换（SQLite 开发 / PostgreSQL 生产），启动时 AutoMigrate 全部模型。

## 短信发送抽象

`SMSSender` 接口两个实现：`ConsoleSender`（验证码打日志，开发默认）与 `AliyunSMSSender`（`SMS_DRIVER=aliyun` 启用，密钥环境变量注入）。

## 建模要点

- **无状态优先**：JWT 无状态校验 + 共享存储只承载必须跨实例共享的（refresh_token、验证码、频率限制），对齐 FaaS 多实例形态
- **密码哈希两轮升级**（SHA256 → bcrypt → Argon2id）：趁未大规模上线完成，OWASP 首选参数（m=64MB, t=3, p=2）
- **运维端点 fail-closed**：`ADMIN_TOKEN` 未配置或请求头不匹配一律 403，运维能力不因配置缺失而意外放开
- **自动注册即认证**：验证码登录兼做注册入口，身份模型无「待激活」中间态
