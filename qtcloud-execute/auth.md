# qtcloud-execute 认证接入方案

execute 当前无身份体系：三个入口（Studio/CLI/Provider）均无用户概念与鉴权。本文规划接入 qtcloud-auth（0.1.0-alpha.9，密码/验证码登录、刷新、注册全链路可用，未大规模接入业务）的路径、前置决策与顺序。支付接入另见 [pay.md](./pay.md)。

## 客户端

- Studio 加登录页，调 `POST /oauth/token`：`grant_type=password` 或 `grant_type=sms_code`（未注册手机自动注册）；`access_token` 有效期 3600 秒，过期用 `grant_type=refresh_token` 静默续期
- token 存储：access_token 放内存 + localStorage；`GET /userinfo` 取昵称头像展示
- CLI 加 `login` 子命令，token 存 `~/.config/qtcloud-execute/`（对齐 qtcloud-pay CLI 的 config 模式），同时支持环境变量注入（AI/脚本场景更友好）
- 全部请求带 `Authorization: Bearer <token>`

## 服务端

Provider 加 JWT 验签中间件（验签名 + 查 exp，auth 侧逻辑可直接参照），受保护端点校验 Bearer。

## 前置决策

1. **验签密钥分发**：auth 当前是 HS256 对称签名，execute Provider 必须与 auth 共享 `JWT_SECRET`（环境变量注入，但密钥多一份暴露面）。auth 未大规模上线是改 RS256 + JWKS 公钥端点的最低成本窗口期——若预期多个业务接入，先向 auth 提需求，一次解决所有资源服务器的密钥分发
2. **授权语义**：auth 的 Role 只到「角色持权限串」一层，execute 需自定义「谁能编辑哪个清单」。最小方案：Provider 记录清单 owner（首次创建者的 user.id），编辑权限跟 owner 走；细粒度 RBAC 后置
3. **跨域**：对齐平台规范，auth 请求经系统级网关处理，应用层零 CORS

## 建议顺序

Studio/CLI 接登录 + Provider 验签中间件先行（模式全部现成：环境变量注入、网关暴露、config 目录），同步向 auth 提出 RS256/JWKS 需求。
