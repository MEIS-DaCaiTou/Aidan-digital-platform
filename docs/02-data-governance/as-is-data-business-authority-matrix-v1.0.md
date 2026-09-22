# 现有系统数据与业务权威矩阵 AS-IS V1.0

> 文档性质：**迁移调查与历史存档**  
> 非正式 PRD 设计依据。所有新系统设计、评审和研发以《业务事实权威矩阵 TO-BE V1.0》为唯一基准。

## 1. 当前已确认生产系统

| 系统 | 当前定位 |
|---|---|
| 领猫 | 服装研发、商品、供应链、采购、生产、质检等业务系统 |
| 聚水潭 JST | 电商 ERP、店铺商品、订单、库存、仓储、发货、售后等执行系统 |
| Excel / 线下表 | 需作为隐形数据源专项调查 |
| POS / 财务 / WMS / CRM 等 | 具体使用情况在迁移 Discovery 阶段继续盘点 |

## 2. AS-IS 调查目的

需要确认每个业务事实当前在哪里创建、由谁修改、被哪些系统消费，以及领猫与聚水潭之间已经存在的商品、SKU、库存、采购、订单同步关系。

本文件不尝试用现状定义未来系统边界。

## 3. 当前商品域矩阵

| 业务事实 | 领猫 | 聚水潭 | 当前风险 | TO-BE 去向 |
|---|---|---|---|---|
| 品牌 | 有 | 有 | 多源 | Product Master |
| 年份/季节/波段 | 有 | 部分 | 编码不统一 | Product Master |
| 品类 | 有 | 有 | 分类树不同 | Product Master + Mapping |
| 款号/SPU | 有 | 有 | 双方均存在 | Product Master |
| SKU | 有 | 有 | 极高：多源身份 | Product Master |
| 颜色 | 有 | 有 | 自由文本/编码差异 | Product Master |
| 尺码 | 有 | 有 | 尺码体系差异 | Product Master |
| 条码 | 有 | 有 | 重复/错绑风险 | Product Master |
| 商品生命周期 | 有/部分 | 有/部分 | 状态口径不同 | Product Master |
| 店铺商品 ID | - | 有 | 平台相关 | Channel Domain |
| BOM/样衣/工艺 | 有 | - | 无需复制 | 领猫 |
| 成本/采购价 | 有 | 部分 | 多口径 | 领猫/财务 |

## 4. 研发供应链域

| 事实 | 当前主要系统 | 迁移判断 |
|---|---|---|
| 商品企划/研发立项 | 领猫 | 保留 |
| 样衣/打版/评审 | 领猫 | 保留 |
| BOM / 面辅料 / 工艺 | 领猫 | 保留 |
| 供应商能力与生产信息 | 领猫 | 保留 |
| 生产跟单/生产节点 | 领猫 | 保留 |
| 质检 | 领猫 | 保留 |
| 成品交仓前状态 | 领猫 | 保留，交仓后切换到仓储事实 |

## 5. 电商交易与履约域

| 事实 | 当前主要系统 | 迁移判断 |
|---|---|---|
| 平台原始订单 | 渠道平台 | 原始事实保留 |
| 电商订单执行 | 聚水潭 | V1 继续保留 |
| 审单/打单/发货 | 聚水潭 | V1 继续保留 |
| 仓库物理库存 | 聚水潭/WMS | 按实际保管系统确认 |
| 平台库存同步 | 聚水潭 | V1 继续执行，后续由 IMS 提供 ATP |
| 售后执行 | 聚水潭/平台 | V1 保留 |
| 集团标准订单 | 当前缺失/不统一 | 后续 OMS |
| 集团 ATP | 当前由 JST 等规则计算 | 后续 IMS |

## 6. 必须调查的重叠区域

重点核实：

1. 领猫商品如何推送聚水潭，是否仍允许双方人工新建 SKU；
2. `款号 / SKU / 条码 / 颜色 / 尺码` 的真实编码规则；
3. 聚水潭采购与领猫采购分别承担什么场景；
4. 同一仓库在领猫、聚水潭、财务/POS 中的编码；
5. “实际库存 / 可用库存 / 可售库存 / 占用 / 虚拟库存 / 在途”的现有定义；
6. 商品图片和源文件当前存放位置；
7. 当前是否存在 Excel 主表绕开系统；
8. 财务系统、POS、门店库存、WMS 的实际产品和接口情况。

## 7. 历史数据需要建立的统一 ID

迁移期间生成：

- `global_spu_id`
- `global_sku_id`
- `global_location_id`
- 必要时 `global_supplier_id`

并通过 `entity_mapping` 维护：

```text
global SKU
   ↕
Lingmao SKU
   ↕
JST SKU
   ↕
Channel SKU
```

## 8. 数据质量问题分类

统一登记：

- MISSING
- DUPLICATE
- MISMATCH
- ORPHAN
- INVALID_MAPPING
- STALE_DATA
- ILLEGAL_WRITE

重点指标包括 SKU 映射率、条码唯一率、仓库映射率、订单 SKU 识别率、库存对账差异率与同步成功率。

## 9. AS-IS 最终输出物

迁移 Discovery 阶段最终需要形成：

- System Landscape
- AS-IS Process
- Data Ownership Matrix
- Source → Global 字段映射
- ID Mapping
- Data Quality Report
- API Capability Inventory
- KEEP / INTEGRATE / EXTEND / MIGRATE / RETIRE 决策表

## 10. 与 TO-BE 的关系

```text
AS-IS
现状事实 / 调查 / 差异
        ↓
Migration Plan
Mapping / Cleanup / Reconciliation
        ↓
TO-BE
未来唯一设计基准
```

如果当前领猫、聚水潭或 Excel 的做法与 TO-BE 冲突，默认作为迁移和治理问题处理，而不是反向修改新系统的权威边界。
