# qtcloud-pay 代金券接入指南

代金券系统是账本核心 API 的现成能力（M2 落地），接入方无需 qtcloud-pay 侧任何改动，按发放 → 查询 → 结算抵现三个动作集成。本文描述调用方链路；券的领域语义见 [models.md](./models.md)。

## 接入链路

### 1. 开户（一次性）

```http
POST /accounts
{"customer_id": "..."}
```

### 2. 发放代金券

```http
POST /accounts/{id}/vouchers
{
  "amount": {"amount": "50.00", "currency": "CNY"},
  "scope": "course",
  "product_id": "...",
  "expires_at": "2026-12-31T00:00:00Z",
  "count": 10,
  "batch_no": "batch-20260810-001",
  "note": "新客立减活动"
}
```

服务端在单个数据库事务内完成：创建一批券并写入一笔 `issue` 交易（信息性记录，不影响余额）。`batch_no` 是幂等键，重复提交同一批次不会重复发放，调用方可放心重试。单批发放上限 1000 张；`scope` 取 all/cloud/course/data/product，`product_id` 仅在 `scope=product` 时生效。

### 3. 查询

```http
GET /accounts/{id}/vouchers
```

返回 id 倒序列表，读取时惰性流转过期状态（`issued` 到 `expired`），无后台任务。

### 4. 结算抵现

代金券没有独立的核销端点——抵现发生在订单结算中，自动参与：

```http
POST /orders
{
  "order_id": "T20260810-001",
  "customer_id": "...",
  "account_id": "...",
  "scope": "course",
  "amount": 100.00
}
```

结算编排（单事务）自动完成：拉取可用券（已发放、未过期、scope 匹配）→ 计费计算得出逐项抵扣计划 → 券抵现部分逐张核销并关联订单号 → 余额缺口部分扣减账户余额 → 每张券写一条 `redeem` 交易、余额部分写一条 `consume` 交易。抵扣明细固化在 `settle_detail`，经 `GET /orders/{id}` 可查可解释。`order_id` 是幂等键，并发重复提交返回已有订单。

## 接入约束

- 券不能单独核销：`Use` 只供结算编排内部调用，券面值必须融入订单结算才有意义
- scope 匹配规则：`all` 全场通用；`product` 精确匹配 `product_id`；其余按业务类型字符串精确匹配（`course` 与 `cloud` 互不通用）。结算时传入的 scope 决定哪些券参与抵现
- 金额整数分：存储与计算全部使用 int64 分，元/分转换集中在 transport 层，请求与响应均以元传输
- 状态契约（`issued/used/expired`）权威在 quanttide-pay-toolkit `tests/fixtures/status.json`，接入方只引用不定义；改动抵扣语义需先动工具库 fixtures，再回 qtcloud-pay 更新引用
- 批次号建议含业务前缀与日期（如 `batch-20260810-001`），一经发放不可复用，规范需与运营提前约定
