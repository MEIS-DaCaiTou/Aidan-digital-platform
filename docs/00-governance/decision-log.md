# 架构与业务决策日志

本文档记录已经冻结、后续不得在局部 PRD 中自行推翻的关键决策。

| ID | 决策 | 状态 |
|---|---|---|
| ADR-001 | 一个业务事实只能存在一个最终写入权威 | Frozen |
| ADR-002 | AS-IS 只作为迁移调查存档；正式 PRD 唯一基准为 TO-BE V1.0 | Frozen |
| ADR-003 | Product Master 是 SPU/SKU/Color/Size/Barcode 的集团身份权威 | Frozen |
| ADR-004 | 领猫保留研发、BOM、样衣、采购、生产、质检等供应链专业事实 | Frozen |
| ADR-005 | 聚水潭保留电商订单、仓储、发货、售后等执行能力；不成为集团商品身份权威 | Frozen |
| ADR-006 | OMS / IMS 逐步自研，第一阶段可先建设 Order Hub / Inventory Hub | Frozen |
| ADR-007 | 所有第三方商品 ID 进入 entity_mapping，不进入集团主键 | Frozen |
| ADR-008 | `spu_id / sku_id` 使用集团永久技术主键；外部编码仅作为映射/查询条件 | Frozen |
| ADR-009 | 已 APPROVED 的 SKU 身份字段不可直接修改；错误身份通过停用旧 SKU + 新建 SKU 处理 | Frozen |
| ADR-010 | Agent 不得直连数据库；写动作必须通过 Domain API + Policy + Approval | Frozen |
| ADR-011 | Akeneo、Saleor 不进入 V1 核心；不重建已有 PLM/SCM/WMS/ERP 通用能力 | Frozen |
| ADR-012 | 后续 Event Contract 必须继承 Product Master 与 Mapping V1.0 已冻结主键和字段语义 | Frozen |

后续新增重大决策使用 ADR-013 起连续编号。
