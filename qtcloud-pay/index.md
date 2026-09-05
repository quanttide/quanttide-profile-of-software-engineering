# 量潮支付云软件工程档案

qtcloud-pay 是量潮支付云服务平台，v0.1.0 聚焦卖课场景的账本核心——账户余额、券、计费、订单结算与对账，支付渠道（微信/支付宝）作为旁挂模块独立演进。源码位于 quanttide-pay `apps/qtcloud-pay`。

## 系统架构

系统由三个组件构成，围绕账本核心 API 组织：

```
Studio (Flutter 桌面)   CLI (Rust，命令 qtcloud-pay)
        │                    │ HTTP
        ▼                    ▼
      Provider (Go，阿里云 FC 3.0)
        │ GORM
        ▼
  SQLite（开发）/ PostgreSQL（生产）
```

Provider 是契约事实源，`cmd/server` 组装，`internal/` 按模块分包（transport / service / repository / model / gorm 五件套）：account（账户余额）、transaction（交易账本）、coupon（优惠券）、voucher（代金券）、billing（计费规则）、order（订单结算）、reconciliation（对账）、channel（支付渠道）。模块依赖关系：`transaction` 是最底层（账本写入唯一入口），`order` 是结算编排者（单事务协调计费、写交易、更新余额与券状态），`channel` 刻意不依赖账本模块。

## 组件与技术栈

| 组件 | 语言与框架 | 版本 | 职责 |
|:--|:--|:--|:--|
| Provider | Go，net/http，GORM | 0.1.0-alpha.8 | 账本核心 API（M1–M4）+ 支付渠道（M5） |
| CLI | Rust，clap，reqwest | 0.1.0 | 账本核心命令行工作台：accounts/recharges/coupons/vouchers/orders/reconcile/milestone |
| Studio | Flutter 桌面（Windows/macOS） | 1.0.0+1 | 管理人员图形化工作台：只做展示与表单，不承载账务逻辑 |

## 对外接口

账本核心：`POST /accounts`（开户）、`POST /accounts/{id}/recharges`（充值登记）、`POST /accounts/{id}/refunds`（退款登记）、`GET /accounts/{id}` 与 `GET /customers/{customer_id}/account`（余额查询）、`GET /accounts/{id}/transactions`（流水）、`POST/GET /accounts/{id}/coupons|vouchers`（发放与列表）、`POST /orders`（结算）、`GET /orders/{id}`、`GET /reconcile/consistency`（一致性校验）、`POST /reconcile/bank`（对公打款核对）、`GET /accounts/{id}/statement`（账单）。渠道（默认关闭）：`POST /pay`、`GET /query/{order_id}`、`POST /refund`。

## 工程实践

横切设计约束沉淀在 `src/provider/docs/conventions.md`，全部模块遵守：账本写入唯一入口（经 transaction + 幂等键唯一约束）、余额/券状态与交易同事务更新、金额全链路整数分、存储双引擎（GORM 方言切换）、券过期惰性流转（无后台任务）、配置全走环境变量（零配置文件）。

契约纪律是本仓库的最高优先级约定：账本语义（交易类型、状态、幂等键、计费计算）的契约权威在 quanttide-pay-toolkit 的 `tests/fixtures/`（JSON），本仓库是消费端——禁止端侧发明语义，未知渠道码必须报错而非 UNKNOWN 兜底，纯逻辑（值对象/枚举/纯函数）持续提炼进工具库，契约演进先改工具库再回本仓库更新引用。

质量门禁按组件执行：Provider `make test`（Go 单元 + 集成，itest 起真实服务）、CLI `cargo test`、Studio `flutter test`。缺陷与计划分别登记在 `src/provider/ROADMAP.md` 的 F1–F8（缺陷）与 T1–T5（v0.2.0 计划）。

## 当前状态

账本核心 M1–M4 已落地并通过里程碑验收（CLI 提供 `qtcloud-pay milestone verify M1` 自动验收）：账户/交易/券/计费/订单/对账全链路可用，CLI 与 Studio 均已对接。渠道模块（微信 JSAPI、支付宝网页支付）实现完整但与账本刻意平行——支付成功不入账，「支付 → 自动入账」业务闭环明确推迟到 v0.2.0。生产部署为纯账本 API（FC + Terraform，manifests/terraform）。

## 演化路线图

- v0.2.0 主线：支付回调 → 自动入账，打通「外部支付 → 交易」闭环（T5/F3），模型不变，变的只是交易来源
- 生产引入版本化数据库迁移（当前 GORM AutoMigrate）
- 计费规则引擎（当前 BillingRule 只提供默认抵扣顺序，规则引擎后置）
- 缺陷清单 F1–F8 修复优先于新功能
