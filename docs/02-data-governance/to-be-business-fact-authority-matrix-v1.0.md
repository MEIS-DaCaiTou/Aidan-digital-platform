# 服装集团业务事实权威矩阵 TO-BE V1.0

> 文档级别：正式 PRD 的唯一权威基线（唯一基准）
> 地位说明：《现有系统数据与业务权威矩阵 AS-IS》保留作为迁移调查文件；所有正式 PRD 均以本 TO-BE V1.0 为准。
> 冻结日期：2026-09

## 1. 项目最高原则

**一个业务事实，只允许一个系统拥有最终写入权。**

- 允许：多个系统保存副本、建立搜索索引、数据仓库保存分析快照、下游系统维护自身执行属性。
- 不允许：同一 SKU 领猫可以创建、聚水潭也可以创建、Product Master 也可以创建；同一 ATP 聚水潭算一份、IMS 算一份、渠道再算一份。

## 2. 最终系统职责（冻结）

| 系统 | 正式定位 | 是否业务事实权威 |
|---|---|---|
| Product Platform | 集团商品身份与标准商品内容 | 是 |
| 领猫 | 服装研发、采购、生产、供应链执行 | 是，限定领域 |
| 聚水潭 | 电商订单、仓储、发货、售后执行 | 是，限定执行事实 |
| OMS | 集团标准订单及履约编排 | 是 |
| IMS | 集团库存逻辑、ATP、预占、库存分配 | 是 |
| Merchandising | 配货、补货、调拨决策 | 是 |
| Channel Platform | 渠道商品及发布意图 | 是 |
| DAM | 数字素材原文件与版本 | 是 |
| 财务系统/FMS | 会计、财务、成本、税务 | 是 |
| Data Platform | 分析与指标 | 非交易事实权威 |
| Agent Platform | AI 访问与操作入口 | 永远不是事实权威 |

## 3. 商品领域（Product Master 拥有的事实）

global_spu_id、global_sku_id、品牌、年份、季节、波段、产品线、品类、款号、SPU、SKU、标准颜色、标准尺码、颜色尺码组合、商品条码、商品生命周期、商品启用/停用状态 → 只有 Product Master 有最终写权限。领猫和聚水潭只能保存 global_spu_id/global_sku_id + 自己的业务属性，不能自行产生新的集团 SKU。

## 4. 商品创建流程（正式确定）

```
商品企划/研发立项 → 领猫 → 申请商品身份 → Product Master → 生成 global_spu_id / global_sku_id
→ 领猫 / 聚水潭 / Channel
领猫负责提出"我要开发这样一个款"；Product Master 负责决定"集团正式商品身份是什么"。
```

## 5. 领猫商品领域拥有的事实（Product Master 绝不复制）

商品企划任务、设计任务、设计方案、样衣、打样需求、打版、样衣评审、款式询价、BOM、BOM 版本、面料、辅料、物料用量、工艺、尺寸规格、生产资料包、研发成本、生产节点、生产跟单、生产质检 → 唯一权威：领猫。Product Master 不是 PLM，不是 SCM。

## 6. 商品内容 PIM

不引入 Akeneo，直接作为 Product Platform 的 Product Content 模块。拥有：标准商品名称、商品简称、标准卖点、商品描述、成分描述、洗护说明、尺码说明、标准营销文案、多语言内容、标准销售属性。领猫提供研发源信息（如面料 90% 锦纶+10% 氨纶），Product Content 转换为标准消费者展示（90% 锦纶，10% 氨纶）——两个不同事实。

## 7. 商品价格事实（不塞进 SKU Identity）

| 价格事实 | 权威 |
|---|---|
| 吊牌价/MSRP、标准零售价 | Product Platform |
| 成本价 | 财务/成本系统 |
| 采购价格、供应商报价 | 领猫 |
| 渠道建议售价、渠道活动价 | Channel Platform |
| 平台实际成交价 | 原始订单/OMS |
| 财务确认收入 | 财务系统 |

## 8. 素材 DAM

独立 DAM（建议 ResourceSpace），唯一拥有：原始图片文件、PSD、AI 设计源文件、视频、文件版本、文件 Hash、素材元数据、素材版权/有效期。Product Platform 只保存 asset_id；商品素材关系归 Product Platform；渠道素材排序归 Channel Platform。

## 9. 供应商事实

- 生产供应商身份（supplier_id、名称、工厂、加工能力、供应品类、产能、生产属性、交期/质量表现）：权威 = 领猫。
- 财务属性（结算主体、银行账户、税务信息、付款条件、应付、发票）：权威 = 财务系统。
- 平台内部 global_supplier_id 只用于映射，不成为新的供应商业务系统。

## 10. 采购事实（划死）

商品相关采购统一归领猫：面料/辅料/成品采购需求、外发加工、生产采购订单、供应商交期、生产收货预期。聚水潭不再作为商品采购业务权威；其现有采购单未来定位为 Warehouse Replenishment Execution Projection（仓库收货所需执行数据）。

## 11. 成品库存按"保管阶段"划分

- 生产阶段（生产完工、待质检、质检完成、待交仓，尚未正式移交销售仓）：领猫 = 权威。
- 正式销售仓入库（完成 Warehouse Receipt）后：聚水潭/WMS = 该仓库物理库存事实权威。
- 不存在"领猫 100 vs 聚水潭 92 谁是真的"，而是领猫还有 8 件未交接 + 聚水潭销售仓实物 92 件。

## 12. Location Master

集团平台拥有很薄的 Location Registry（不做仓库业务，只做身份统一）：global_location_id；类型 FACTORY / SUPPLIER / DC / REGIONAL_DC / STORE / RETURN / DEFECT / IN_TRANSIT / VIRTUAL。领猫仓库 WH_038、聚水潭 wh_id=16、POS SH001 全部映射为同一集团 Location。

## 13. 订单三层事实

1. **Source Order**（原始平台事实，如 DOUYIN-20260922-XXX）：权威 = 抖音，聚水潭负责接入；
2. **Execution Order**（聚水潭电商执行订单）：权威 = 聚水潭，负责审单、仓库执行、打单、拣货、发货、物流回传、平台售后协同；
3. **Group Order**（集团标准业务订单）：权威 = OMS，生成 group_order_id，统一渠道/店铺/商品/SKU/金额/状态/履约/售后。

Platform Order → JST Execution Order → Group OMS Order 是三种不同事实。

## 14–15. OMS 拥有/不拥有的事实

- **拥有**：group_order_id、标准订单模型、标准订单状态、标准履约状态、Group SKU 映射、标准渠道、标准店铺、拆单/合单关系、履约路由、Group 售后状态。
- **不拥有（不能自己猜）**：平台是否支付成功（渠道平台）、第三方支付真实结果（支付机构/平台）、仓库是否真正出库（聚水潭/WMS）、物流是否真正签收（物流方）、平台退款是否成功（平台/支付机构）。OMS 通过事件映射为集团标准状态。

## 16. OMS 标准状态

CREATED → PAID → CONFIRMED → ALLOCATING → ALLOCATED → FULFILLING → SHIPPED → DELIVERED → COMPLETED；异常 ON_HOLD / CANCELLED / CLOSED。OMS 内部状态不能直接使用"抖音状态码 2/聚水潭状态 3"，这些属于 Adapter。

## 17. 库存事实正式划分（最重要的权威表）

| 事实 | 唯一权威 |
|---|---|
| 生产完成未交仓数量 | 领猫 |
| 仓库实物库存 | 聚水潭/WMS |
| 门店实物库存 | POS |
| 质检冻结 | 实际保管系统 |
| 残次库存 | WMS/POS |
| 采购在途 | 领猫 |
| 调拨在途 | IMS |
| 订单预占 | IMS |
| 库存业务锁定 | IMS |
| 安全库存 | IMS |
| ATP 可售库存 | IMS |
| 渠道库存池 | IMS |
| 库存分配 | IMS |
| 财务库存金额 | 财务系统 |

## 18. IMS 的真正定义

IMS 不回答"仓库里物理上有几件"（由 WMS/JST 回答）；IMS 回答"集团当前还能承诺卖多少"。概念公式：ATP = Physical On Hand − Reservation − Business Lock − Quality Hold − Safety Stock ± 可承诺在途（具体按库存类型配置）。

## 19–20. IMS Ledger 与库存预占

- IMS 必须采用 Ledger：+100 WAREHOUSE_RECEIPT / -2 ORDER_RESERVE / +1 ORDER_RELEASE / -1 SHIPMENT / -10 TRANSFER_OUT / +10 TRANSFER_IN；当前余额是 Ledger 的结果，不允许只维护 available_qty=95。
- 库存预占只有 IMS 能做：OMS/Channel/聚水潭不能自己维护 reservation_qty。流程：订单进入 OMS → reservation.requested → IMS → 创建 Reservation；失败 → reservation.failed → OMS 缺货处理。

## 21. 聚水潭库存的未来定位

聚水潭继续维护仓库实物、订单执行、出库、入库；但 JST available ≠ 集团 ATP。目标链路：JST/WMS → Physical Inventory → IMS → ATP → Channel Allocation → 聚水潭/渠道。

## 22–24. 配货/补货/调拨事实

- **配货**：唯一权威 Merchandising Platform（Allocation Plan/Rule/Store Profile/Size Curve/Recommendation/Decision）；执行后 → IMS Transfer/Replenishment。
- **补货**：唯一权威 Merchandising；输入 Sales/ATP/Physical/In Transit/Store Grade/Size Curve/Lifecycle/Sales Velocity，输出 Recommendation；批准后 Decision → IMS 执行。
- **调拨三阶段**：调拨建议（Merchandising）→ approve → 正式调拨单（IMS Transfer Order）→ 仓库出入库执行（JST/WMS/POS）→ IN_TRANSIT（IMS）→ 目的地 Receipt → COMPLETED（IMS）。

## 25–26. 渠道商品事实

- 集团商品（Product Master）、标准内容（Product Content）、素材（DAM）、渠道商品草稿/标题/属性/SKU 配置/图片排序/发布意图（Channel Platform）、平台商品 ID（平台+Mapping）、平台审核状态/真实上下架状态（平台）。
- 聚水潭可作 Channel Adapter，但不能拥有集团 Channel Product。最终允许 Channel Platform → JST Adapter → 抖音 或 Channel Platform → Douyin Direct Adapter → 抖音；无论哪条链，Channel Product 不改变。

## 27–28. 售后与财务事实

- **售后**：平台售后申请（平台）、聚水潭售后执行（聚水潭）、仓库退货收货（聚水潭/WMS）、实物质检（实际执行系统）、Group After Sale（OMS）、退款支付结果（平台/支付机构）、财务退款凭证（财务系统）。
- **财务**：会计科目、凭证、应收、应付、总账、税务、发票、银行、实际成本、库存价值、财务收入、利润 → FMS/ERP 财务模块。OMS 只告诉财务卖了什么/多少钱/退款多少，不能告诉财务会计上怎么记账。

## 29. Data Platform 事实

不是交易权威，但拥有**指标定义**（metric_code、name、formula、owner、version）——Metric Registry SOR。实际订单/库存仍来自业务系统。

## 30–31. Agent 身份与权限

- Agent = Actor（与 User/System/Service Account 同级），不是数据库所有者，**永远没有数据所有权**。
- Level 1 Read（自动执行）：search_products、get_product、get_inventory、get_atp、search_orders、get_order、get_sales_metrics、get_stock_risk。
- Level 2 Recommend/Draft（自动产生但不生效）：create_replenishment_draft、create_transfer_draft、create_allocation_draft、generate_channel_product_draft。
- Level 3 Execute：approve_transfer、publish_product、cancel_order、execute_refund → 必须 Agent → Policy → Approval → Domain API。
- 永久禁止：UPDATE DB、DELETE SKU、SET inventory=xxx、修改财务凭证。

## 32–33. 基础 Registry 与 External ID Mapping

- 一期只做四个薄 Registry：Product / Location / Organization / External ID Registry。
- External ID Registry 解决：Product Master SKU ↔ 领猫 SKU ↔ 聚水潭 SKU ↔ 抖音 SKU ↔ POS SKU。
- entity_mapping 字段：global_id、entity_type、source_system、source_id、source_code、status、valid_from、valid_to、created_at。禁止在业务代码里写 if JST: use sku_id；统一走 Mapping Service。

## 34. 系统间数据流（冻结）

商品：领猫研发立项 → Product Master → SPU/SKU → 领猫/JST/Channel。
内容：领猫研发数据 → Product Content → Channel Product。
素材：DAM → Asset ID → Product → Channel。
订单：Platform → JST → OMS。
库存：JST/WMS/POS → Physical Inventory → IMS → ATP。
商品运营：Sales+Inventory+Product → Merchandising → Allocation/Replenishment/Transfer → IMS。

## 35–38. 一期范围与系统边界

**一期正式开发六个平台**：01 Product Platform（Product Master/Content/External ID Mapping）；02 Integration Platform（Lingmao Adapter、JST Adapter、Event Bus、Reconciliation）；03 Order Platform（第一阶段 Order Hub → 演进 OMS）；04 Inventory Platform（第一阶段 Inventory Hub → 演进 IMS）；05 Merchandising Platform；06 Channel/Agent Platform（Channel Publishing、MCP Gateway、Policy、Approval）。

**明确不做**：新 ERP、Saleor Core、Akeneo、自研 WMS/PLM/SCM/财务/生产管理、CDP、复杂 CRM、全功能 AI 平台。

- **领猫边界**：负责"商品怎么被研发和生产出来"（企划、研发、样衣、BOM、供应商、采购、生产、质检）。
- **聚水潭边界**：负责"电商订单怎么被仓库真正履约出去"（平台订单接入、审单、仓储、出库、发货、物流、售后执行）；通过 Adapter 映射到集团统一模型，不使用第三方 ID 作为上层主键。
- **集团自研平台边界**：负责"集团认为什么商品存在、订单处于什么业务状态、还有多少货能卖，以及货下一步应该去哪里"（Product Master / OMS / IMS / Merchandising）——企业真正应掌握的核心数字资产。

## 39–42. 冻结原则与强制问题

**PRD 强制问题（任何功能进入开发前必须写清）**：Fact Name、Fact Owner、Create Authority、Update Authority、Read Consumers、Source Event、Downstream Event、Conflict Resolution、Audit Requirement、Agent Permission。写不清 → 不得进入开发。

**最终业务事实版图**：

```
GROUP CONTROL LAYER：Product Master（商品身份）/ OMS（标准订单）/ IMS（ATP）/ Product Content / Merchandising（配补调决策）
→ Integration Layer → 领猫 SCM（研发供应链事实）| 聚水潭（电商执行事实）
→ Events → Data / BI / Agent
```

**四条冻结原则**：
1. Product Master 决定：这个商品是谁。
2. 领猫决定：这个商品是怎么研发、采购和生产出来的。
3. 聚水潭/WMS 决定：货在物理世界里发生了什么。
4. OMS + IMS + Merchandising 决定：集团如何卖这件商品，以及下一件货应该去哪里。
