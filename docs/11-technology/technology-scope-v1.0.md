# 技术路线收敛与 V1 范围冻结

> 状态：已拍板通过，作为《技术架构 V1.0》冻结范围，后续进入 PoC Gate 验证。
> 冻结日期：2026-09

## 一、不能妥协的规矩：商品/订单/库存不许有第二个"真身"

- **商品主数据**：自研很薄的 Product Master（SPU/SKU/颜色/尺码/生命周期），其余系统（ERP、商城、PIM）一律只读这些字段，写入接口只暴露给 Product Master 自己。ERPNext 的 Item、Saleor 的 Product 通过权限/校验堵死"手工新建 SKU"，只允许通过 API 从 Product Master 同步。
- **订单和库存（OMS/IMS）**：进入自研范围，不用 Saleor 当系统记录本体。Saleor 最多作为官网/小程序等纯 D2C 前台的可选组件，其库存和订单状态单向听 IMS 的，不能反向写。
- **一句话：开源项目只允许当"执行引擎"，不允许当"第二权威源"。**

## 二、技术栈削减

| 保留 | 砍掉/降级 | 理由 |
|---|---|---|
| Keycloak（IAM） | — | 边界干净，与商品/订单无一致性纠缠 |
| ResourceSpace（DAM） | — | 素材靠 ID 引用的松耦合关系，替换代价小 |
| ERPNext（财务/采购/基础生产） | — | 但财务/税务单独验证（见四） |
| — | Saleor 作为独立 Commerce Core | 降级为可选前台组件 |
| — | Akeneo（PIM）直接砍掉 | 在 Product Master 上加内容模块，工作量不比维护 PHP/Symfony 更大；省一整个技术栈更值 |

自研核心保留差异化价值与一致性风险最高的四块：**Product Master、OMS/IMS、配货补货调拨、渠道发布中心**。

## 三、交付节奏（先跑通最不确定的，不先搭架子）

1. **第一步（4–8 周，只产风险答案，不产业务功能）**：
   - 按真实量级（十几万～几十万 SKU、上百门店）压测候选 ERP 的 Item Variant/库存模型；
   - 敲定财务/税务用哪个系统（见四）；
   - 写一页纸的 Product Master 字段权威表和事件契约，钉死"谁能写什么"再写代码。
2. **第二步**：商品+供应链地基（Product Master、ERPNext、DAM、集成总线）。
3. **第三步**：全渠道。先接一个平台（如抖音），把 OMS→IMS→渠道适配器→回传链路跑稳，再复制到天猫、京东。
4. **第四步**：商品运营智能（配货/补货/调拨）——唯一长期竞争壁垒，宁可用简单规则跑几个月攒数据再迭代模型。
5. **第五步**：数据平台 + Agent 接口。Product Master 和基础 OMS 就绪后，只读 Agent 工具先上线；推荐类第二批；执行类最后且必须挂审批流。

## 四、财务/税务选型 Gate 0（硬性条件）

- ERPNext 需同时通过：①商品 SKU 规模 ②采购业务 ③基础制造 ④成本核算 ⑤中国会计 ⑥发票/税务 ⑦金税相关实际落地 ⑧月结/年结 ⑨集团多法人。任何关键项不通过 → 直接进入金蝶/用友/Odoo 中国实施方案或其他成熟国内 ERP。
- ERPNext 中国本地化当前主要为社区项目（如 saoxia/erpnext_china，README 说明 v15 通过兼容测试），与成熟国家级财税本地化产品不是一个风险等级；官网中国本地化页面仍标 [Draft]。需找国内生产环境、对接过金税的实施商验证，拿不到可验证案例就换方案。
- Odoo 19 官方财政本地化国家列表含中国，但"有 China Localization"≠"满足中国服装集团金税/开票/税务申报/集团财务要求"；部分高级报表属 Enterprise 范围。Gate 0：真实中国生产环境案例 + 实施商 + 财务负责人验证。

## 五、组织保障

- 指定小型平台团队，专责 Product Master、事件总线、API/MCP 网关的代码评审权；任何业务团队"顺手加商品字段"的 PR 必须过该团队。
- 平台团队还拥有三个"机器权力"：CI 强制 Architecture Test、Contract Test、Schema Compatibility Test（如 order-service 直接连接 product_db → FAIL；删除事件必需字段 → FAIL）。

## 六、预算/团队有限的收缩方案

第一期只做 Product Master + ERPNext + 一个自研轻量配货补货模块；渠道发布、CDP、AI Agent 全部后置，避免八个子系统同时开工导致每块都是半成品。

## 七、事实级权威源（对"单一权威源"的修正）

**一个业务事实，只允许一个写入权威**，而非"一个名词只能属于一个系统"：

| 事实类型 | 唯一权威系统 | 其他系统定位 |
|---|---|---|
| SPU/SKU 身份 | Product Master | ERP/WMS/商城只保存投影 |
| 颜色/尺码/商品生命周期 | Product Master | 下游只读 |
| 商品销售内容 | Product Master Content 模块 | 渠道获得投影 |
| 素材文件/版本 | ResourceSpace | Product Master 只保存 Asset ID |
| 原始平台订单事实 | 抖音/天猫等来源平台 | OMS 幂等接入 |
| 集团标准订单 | OMS | ERP/WMS/渠道读取 |
| 履约状态 | OMS | WMS 提交履约事件 |
| 支付/平台退款事实 | 来源平台/支付机构 | OMS 标准化保存 |
| 仓库实物库存 | WMS | IMS 消费库存事件 |
| 门店实物库存 | POS/门店库存服务 | IMS 消费库存事件 |
| ATP 可售库存 | IMS | 所有销售渠道只读 IMS |
| 库存预占/释放 | IMS | OMS 请求 IMS |
| 渠道库存池 | IMS | 抖音/天猫只是投影 |
| 财务库存价值 | ERP/财务系统 | 来源于业务凭证 |
| 配货/补货/调拨建议 | Merchandising | 批准后形成正式业务单据 |

示例：WMS 实物 92（WMS 是真相）→ IMS 收到事件后计算 ATP=75（IMS 是真相）——两个不同业务事实，不是两个库存真相。

## 八、Product Master"禁止第二真身"的四层工程保护

1. UI / Role Permission（Item: Read=Yes, Create=No, Write 核心字段=No, Import=No）；
2. before_insert / validate（Frappe App 在写入发生前拒绝非 PRODUCT_MASTER 来源写入）；
3. 专用 Integration Service Account；
4. 持续一致性审计：每天/实时核对 Product Master SKU ↔ ERPNext Item ↔ WMS SKU，发现下游存在但 PM 不存在的 SKU 立即告警。

注意：Frappe 存在 ignore_permissions=True 及 db_insert/db_update 绕过路径，仅靠页面/角色权限不是绝对安全边界，必须 Prevent + Detect 并用。

## 九、第一阶段 PoC 必产出的三样资产

- **Product Master Kernel**：至少跑通 POST /products、POST /products/{id}/skus、GET /products/{id}、product.created、sku.created、sku.updated；
- **Event Contract**；
- **Integration Test Harness**：验证 Product Master → ERPNext → WMS Mock → Channel Mock 保持单一权威。

## 十、第一阶段六个"生死 Gate"

- **Gate 1 Product Master**：不能从 ERP/WMS/渠道产生未经 PM 授权的 SKU；
- **Gate 2 大数据量**：20–30 万 SKU、数百 Location、千万级库存流水、真实颜色/尺码组合，测 SKU Search / Inventory Query / ERP Sync / Batch Sync / Event Replay；
- **Gate 3 一致性**：重复消息、乱序、丢失、ERP 宕机、接口超时、重复回调 → 幂等、Retry、DLQ、Replay、Reconciliation 全部跑通；
- **Gate 4 ERP 财税**：财务人员实际确认总账/应收/应付/采购/成本/税务/开票/月结/审计能否落地；
- **Gate 5 OMS/IMS 模型**：抖音下单 → OMS → IMS Reserve → WMS Mock → Shipment → IMS Deduct → OMS Shipped；再制造取消（Reserve → 取消 → Release），验证 Ledger 能完整重建状态；
- **Gate 6 Agent**：Agent 只能查询，绝不能直接 UPDATE/INSERT/访问 DB；证明 MCP → Domain API 路线跑得通。

## 十一、Agent 权限三级标准（平台标准）

- **LEVEL 1 READ**：Agent 自动执行（search_products、get_product、get_sku、get_inventory、get_inventory_availability、search_orders、get_order、get_store_inventory、get_stock_ledger、get_product_sales_summary，全部只读）。
- **LEVEL 2 DRAFT/RECOMMEND**：可产生草稿，不能生效（create_transfer_draft、create_replenishment_draft、generate_channel_product_draft）。
- **LEVEL 3 EXECUTE**：必须 Agent → Policy Engine → Approval → Business API → Execute（approve_transfer、publish_product、adjust_inventory、cancel_order、refund_order）。
- 不存在 Agent → DB；不存在 Agent → ERPNext 通用 CRUD 接口。

## 十二、V1 正式冻结的六大平台

```
01 Product Platform      Product Master / Product Content(PIM) / External ID Mapping
02 Commerce Platform     OMS / IMS
03 Channel Platform      Channel Publishing / Douyin Adapter（后续 Tmall/JD/PDD）
04 Merchandising Platform Allocation / Replenishment / Transfer
05 Integration Platform  ERP Adapter / WMS Adapter / POS Adapter / Event Bus
06 Agent Platform        Domain APIs / MCP Gateway / Policy / Approval
```

现有/开源：ERP → ERPNext（最终财务选型待定）、DAM → ResourceSpace、IAM → Keycloak、Data/BI 后续建设。
一期退出/后置：Saleor（退出一期）、Akeneo（删除）、CDP（后置）、大数据平台（后置）、复杂 AI 预测（后置）、新 ERP/自研 WMS/PLM/SCM/财务/生产管理（不做）。

## 十三、最关键的项目原则

**系统可以复制数据，但不能复制数据所有权。**

- Product Master 才有资格决定"这个 SKU 是否存在"；
- WMS 决定实物库存事实；IMS 决定这件货还能不能卖；OMS 决定集团订单处于什么履约状态。
- 把这件事做死后，ERPNext 换金蝶、ResourceSpace 换别的 DAM、抖音换接口版本，集团核心架构都不用重做。
