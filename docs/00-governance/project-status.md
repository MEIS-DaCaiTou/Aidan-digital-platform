# 项目状态与接手基线

> 更新时间：2026-09-24
> 作用：记录当前已经冻结的结论、仓库落地状态、正在推进的交付物和下一步队列。讨论形成新结论后应先更新对应正式文档，再更新本页；本页不替代 TO-BE、Contract 或 ADR。

## 当前阶段

项目已完成从业务全景讨论到正式文档仓库的迁移，当前进入“契约优先”的产品与研发设计阶段。

已合并的仓库基线：PR #1 `docs: establish project documentation baseline V1`。

当前进行中：

- 《Product Domain Event Contract V1.0》
- Product Event JSON Schema 与事件样例

## 已冻结业务与系统边界

1. 企业继续使用领猫与聚水潭，第一阶段不替换两套生产执行系统。
2. 领猫负责服装研发、样衣、BOM、采购、生产、质检等供应链专业事实。
3. 聚水潭负责电商订单、仓储、发货、售后和平台执行；不承担集团商品身份权威。
4. Product Master 是 SPU、SKU、Color、Size、Barcode 的集团身份权威。
5. 所有领猫、聚水潭和渠道外部 ID 进入 `entity_mapping`，不进入集团主键。
6. OMS、IMS 第一阶段分别以 Group Order Hub、Inventory Hub 方式建立统一只读模型，待数据与事件可靠后再逐步取得执行权。
7. 配货、补货、调拨是集团自研的高价值核心能力。
8. Agent 不是 System of Record，只能经 Domain API/MCP Tool 与 Policy/Approval 操作。

## 正式数据主线

```text
领猫 SCM（研发源）
  → Product Identity Request
  → Product Master（集团商品身份）
  → 领猫 / 聚水潭 / Channel Projection
  → Product Event
  → OMS / IMS / Merchandising / Data / Agent
```

## 文档权威顺序

1. TO-BE 业务事实权威矩阵
2. Product Master 字段级模型
3. 领猫 × 聚水潭 × Product Master 数据映射
4. Event Contract / API Contract / PRD / ADR
5. AS-IS 调查与迁移存档
6. 聊天记录

聊天记录用于讨论；一旦形成稳定结论，必须进入仓库并通过 PR 留痕。

## 后续交付队列

1. 冻结 Product Domain Event Contract V1.0。
2. 编制 Product API Contract V1.0，覆盖 Identity Request、SPU/SKU、Barcode、Mapping、Resolver 与状态转换。
3. 编制 Product Master ERD / PostgreSQL Schema V1.0，落实 Outbox、Inbox、Audit 和 Mapping 表。
4. 编制 Lingmao Adapter 与 JST Adapter Contract，补企业实例 API 字段、认证、限流和错误码。
5. 建立历史商品迁移与 Reconciliation PoC，验证 Barcode、款号、颜色、尺码和 External Mapping。
6. 在商品契约稳定后编制 Order Domain 与 Inventory Domain Event Contract。

## 尚待企业实例确认

- 领猫企业实例 OpenAPI 字段、Webhook/增量机制、认证和限流。
- 聚水潭企业实例可用接口、应用授权范围、速率限制和业务错误码。
- 集团、品牌、法人、仓库、门店的正式编码与 `tenant_id` 规则。
- 历史商品规模、重复条码率、未映射 SKU 比例和上线门槛。
- 消息中间件和 Schema Registry 的具体技术选型。

这些事项不阻塞领域语义、主键、事件名称和最小 Payload 的 V1 冻结；它们在 Adapter Contract、数据模型与 PoC 中补齐。
