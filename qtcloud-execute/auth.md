# qtcloud-execute 认证接入方案

execute 当前无身份体系：三个入口（Studio/CLI/Provider）均无用户概念与鉴权。本文规划接入 qtcloud-auth（0.1.0-alpha.9，密码/验证码登录、刷新、注册全链路可用，未大规模接入业务）的路径、前置决策与顺序。支付接入另见 [pay.md](./pay.md)。

## 客户端

- Studio 加登录页，调 `POST /oauth/token`：`grant_type=password` 或 `grant_type=sms_code`（未注册手机自动注册）；`access_token` 有效期 3600 秒，过期用 `grant_type=refresh_token` 静默续期
- token 存储：access_token 放内存 + localStorage；`GET /userinfo` 取昵称头像展示
- CLI 加 `login` 子命令，token 存 `~/.config/qtcloud-execute/`（对齐 qtcloud-pay CLI 的 config 模式），同时支持环境变量注入（AI/脚本场景更友好）
- 全部请求带 `Authorization: Bearer <token>`

## 服务端

认证收口在系统级网关（阿里云 API 网关，`api.quanttide.com`），不在 Provider：网关 JWT 鉴权插件在边缘验签，无效令牌到不了后端；execute 五个端点已在网关注册，Authorization 头透传已配置（`scripts/api-gateway/deploy.sh`）。

验签配置优先走 RS256：插件配 auth 的公钥（JWKS），无共享密钥——与平台部署规范已预留的 `JWT_PRIVATE_KEY`（RS256，base64 单行）对齐；HS256 仅作为 auth 未升级前的过渡（网关配 `JWT_SECRET`，密钥只存网关一份，不分发给各业务后端）。

Provider 侧不引入 JWT 库，只读网关透传的 claims 头（如 `X-JWT-Claim-sub`）取用户标识，用于授权判定。

网关层方案的两个前提：

1. **封禁 FC 直连**：execute 的 FC 触发器是公网 URL，网关层鉴权只保护网关入口——必须将 FC 限制为仅允许网关调用（IP 白名单或签名校验），否则拿到 FC 地址即可绕过鉴权，方案形同虚设
2. **网关只做认证不做授权**：「谁能编辑哪个清单」的 owner 语义仍在 Provider（见前置决策 2），claims 头只是省去解析 JWT

## 前置决策

1. **验签位置**：认证统一收口在网关（JWT 鉴权插件 + FC 直连封禁），Provider 不自行验签。auth 当前是 HS256 对称签名，过渡期网关配共享密钥；auth 升级 RS256 + JWKS 公钥发布后，网关改配公钥即彻底去除密钥分发——auth 未大规模上线仍是推动升级的最低成本窗口期
2. **授权语义**：auth 的 Role 只到「角色持权限串」一层，execute 需自定义「谁能编辑哪个清单」。最小方案：Provider 记录清单 owner（首次创建者的 user.id，取自网关透传 claims），编辑权限跟 owner 走；细粒度 RBAC 后置
3. **跨域**：对齐平台规范，auth 请求经系统级网关处理，应用层零 CORS

## 建议顺序

Studio/CLI 接登录先行（模式全部现成：网关暴露、config 目录）；服务端同步推进网关 JWT 插件配置与 FC 直连封禁；auth 升级 RS256 后网关改配公钥。
