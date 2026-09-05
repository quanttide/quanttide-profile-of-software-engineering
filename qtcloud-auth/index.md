# 量潮身份云软件工程档案

qtcloud-auth 是量潮身份认证与权限管理云服务，提供 OAuth 2.0 标准的统一认证入口——用户名密码登录、手机验证码登录（含自动注册）、令牌刷新与用户信息查询。源码位于 quanttide-auth `apps/qtcloud-auth`。

## 系统架构

系统当前只有单一服务端组件 Provider（Go），面向各业务应用提供认证 API：

```
业务应用（Studio/CLI/第三方）──HTTPS──▶ Provider (Go，阿里云 FC 3.0)
                                          │ GORM
                                          ▼
                              共享 RDS PostgreSQL（quanttide-platform 管理）
```

Provider 是契约事实源：OAuth 2.0 标准端点形状（`/oauth/token`、`/userinfo`），内部为典型的 cmd/internal 分层（`cmd/server` 入口组装，`internal/api` 处理器、`internal/model` 数据模型、`internal/store` GORM 持久化、`internal/app` 数据库方言切换）。

## 组件与技术栈

| 组件 | 语言与框架 | 版本 | 职责 |
|:--|:--|:--|:--|
| Provider | Go 1.23+，net/http，go-oauth2/oauth2，GORM | 0.1.0-alpha.9 | 认证 API：令牌签发、验证码、注册、用户信息、运维端点 |

## 对外接口

| 方法 | 路径 | 说明 |
|:--|:--|:--|
| POST | `/oauth/token` | 统一认证入口，按 `grant_type` 分发：`password`（密码）/ `sms_code`（验证码，含自动注册）/ `refresh_token`（刷新） |
| POST | `/oauth/sms/send` | 发送手机验证码（频率限制共享存储） |
| POST | `/oauth/register` | 用户名密码/邮箱注册 |
| GET | `/userinfo` | 当前用户信息（OIDC 标准，Bearer JWT） |
| DELETE | `/admin/users/{login}` | 运维端点：按用户名/邮箱/手机号删除用户，`X-Admin-Token` 保护（fail-closed，未配置一律 403） |

安全基线：密码哈希 Argon2id（PHC 格式，OWASP 首选参数），JWT HS256 无状态校验（天然适配 FaaS 多实例），`JWT_SECRET` / `ADMIN_PASSWORD` / `ADMIN_TOKEN` / 短信密钥全部环境变量注入。

## 工程实践

存储双引擎对齐 qtcloud-pay 选型：开发 SQLite（`DB_SQLITE_DSN`），生产 PostgreSQL（`DB_DRIVER=postgres` + `DATABASE_URL`），GORM AutoMigrate。验证码、频率限制与 OAuth 令牌（含 refresh_token 轮换）均走共享存储，保证 FC 多实例可用。

质量门禁：`go test ./...`（单元）+ pytest 集成测试（`tests/`，httpx 对接运行中服务，`SMS_TEST_CODE=123456` 固定验证码）。

部署遵循平台统一模式：Docker Hub + 阿里云 ACR 双通道镜像发布（FC 中国区拉不到 Docker Hub），Terraform IaC（`manifests/terraform`，系统级共享 VPC/RDS + 应用级数据库与 FC 函数），`provider/*` tag 触发 deploy workflow。部署踩坑沉淀在 `docs/dev-guide/ci.md`。

## 当前状态

2026-08 一周内从 0.0.1（OAuth 令牌端点 + 短信验证码）推进到 0.1.0-alpha.9：存储从 BuntDB 迁 GORM、密码哈希 SHA256→bcrypt→Argon2id 两轮升级、阿里云 SMS 发送器上线、注册端点与运维端点补齐、真实密钥注入替换默认值。核心认证链路可用，尚未大规模接入业务。

## 演化路线图

按 provider ROADMAP 的 FC 适配规划推进：

- 过期验证码清理任务（当前每次发送落一条记录，无清理）
- `/healthz` 健康检查端点（FC 探活与 API 网关接入的前置）
- 日志结构化（JSON），便于 FC 日志服务检索
- 密钥管理升级：FC 环境变量 → KMS 注入（去除 tfstate 明文），长期 CI 升级 OIDC + RAM 角色
