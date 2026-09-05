# qtcloud-pay 领域模型

qtcloud-pay 账本核心（Go，provider v0.1.0-alpha.8）的领域模型：账户 × 交易 × 双券 × 计费 × 订单 × 对账八个模块。JSON 形状的契约权威在 quanttide-pay-toolkit `tests/fixtures/`，本文描述的是 provider 内部结构与建模语义。

## 账本核心

### Account（账户）

客户的虚拟钱包，定义在 `internal/account/model.go`：`id`、`customer_id`（客户标识，账号中心前台按此查询）、`balance`（余额，分）、时间戳。Balance 是交易的投影——不是独立事实，与交易在同一事务内维护，可由交易流水推导（对账的一致性基准）。

### Transaction（交易）

不可变的账本记录，定义在 `internal/transaction/model.go`，是全系统的写入唯一入口：

| 字段 | 类型 | 说明 |
|:--|:--|:--|
| `id` | int64 | 主键（自增） |
| `account_id` | string | 所属账户 |
| `type` | ledger.Type | 交易类型（工具库契约） |
| `amount` | int64 | 金额（分） |
| `balance_after` | int64 | 交易后余额快照（仅充值/消费有效） |
| `order_id` | string | 关联订单 |
| `idempotency_key` | string | 幂等键（唯一索引，不对外） |
| `note` | string | 备注 |
| `created_at` | time | 创建时间 |

五类交易类型（契约来自工具库 `pkg/ledger`）：`recharge` 充值（对公打款入账）、`refund` 退款（对公退款出账）、`consume` 消费（余额支付部分）、`issue` 发券、`redeem` 核销（券抵扣部分）。后两类是信息性记录，不影响余额（`AffectsBalance`）。只插入、不更新、不删除，幂等由 IdempotencyKey 唯一约束保证。

## 优惠手段

### Coupon（优惠券）与 Voucher（代金券）

两券分立建模，语义不同：Coupon 是一条规则（本身不代表钱），按比例（`rate` 整数百分比，90 = 9 折）或满减（`threshold` 门槛 + `amount` 减额）抵扣；Voucher 直接抵现（本身就是钱，`amount` 即面值）。

共同结构：`account_id`、`batch_no`（批次，内部字段）、`scope`（适用范围：all/cloud/course/data/product，product 另需 `product_id`）、`expires_at`、`status`（issued/used/expired，契约来自工具库 `pkg/status`）、`used_at`、`order_id`。过期采用惰性流转——读取时判定并更新状态，无后台任务。

## 计费与结算

### BillingRule（计费规则）

抵扣顺序配置：`priority`（执行顺序）+ `kind`（coupon/voucher/balance）+ `condition`（JSON，预留）。v0.1.0 只提供默认抵扣顺序，规则引擎后置。抵扣计算本身（`CouponInput`/`VoucherInput`/`Deduction` 类型别名）全部引用工具库 `pkg/billing`——纯计算、无存储依赖，给定订单金额与可用券/余额输出逐项抵扣明细。

### Order（订单）

客户购买付费服务的交易请求，结算是唯一的跨模块写入口：`id`、`customer_id`、`account_id`、`product_id`、`scope`（业务类型）、`amount`（分）、`status`（created/settled）、`settle_detail`（结算计划快照，json.RawMessage——逐项抵扣明细的不可变留痕）、时间戳。`POST /orders`（`order.Settle`）在单个数据库事务内完成：应用计费规则 → 写消费/核销交易 → 更新余额与券状态。v0.1.0 的「支付」语义即余额/券抵扣结算，外部支付渠道不入订单。

## 对账

`internal/reconciliation/model.go` 三组模型：

- `Discrepancy`：一致性校验结果——账户当前余额与交易推导余额的偏差
- `BankRow`/`BankMatch`/`BankUnmatch`/`BankReport`：对公打款核对——银行流水 CSV（凭证号 + 金额分 + 日期）与充值交易匹配，产出匹配/未匹配报告
- `Statement`/`StatementEntry`：账户账单导出

对账全部按需调用，无调度器。

## 支付渠道

`internal/channel` 定义 `Provider` 接口（`Name/Pay/Query/Refund`），微信 JSAPI（公众号/小程序卖课）与支付宝网页支付（PC 卖课）两个实现，各自暴露原生类型方法并提供接口适配器。渠道原始状态码解析遇未知值必须报错（不用 UNKNOWN 兜底，新渠道状态暴露而非掩盖）；渠道与账本刻意平行，v0.2.0 接入时只新增「回调 → 自动入账」适配。

## 建模要点

- **余额是投影不是事实**：Account.Balance 可由 Transaction 流水推导，一致性靠「同事务更新 + 对账校验」维持，不引入独立余额账
- **账本不可变 + 幂等键唯一约束**：不丢、不重、可查，是全部写路径的共同前提
- **双券分立**：规则型优惠与面值抵现语义不同，不抽象成统一「优惠」实体
- **结算快照留痕**：Order.settle_detail 固化逐项抵扣明细，结算后规则或券状态变化不影响已成交订单的可解释性
- **契约单向依赖**：端侧（provider/cli/studio）引用工具库契约，绝不反向定义；契约演进由 fixtures 驱动
