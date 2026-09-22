# Product Master 字段级模型 V1.0

> 文档级别：核心数据架构基线
> 上位依据：《服装集团业务事实权威矩阵 TO-BE V1.0》
> 适用范围：Product Platform、领猫 Adapter、聚水潭 Adapter、OMS、IMS、Channel Platform、Merchandising、Data Platform、Agent/MCP
> 数据权威：Product Master
> 版本：V1.0（已冻结）

## 1. 文档目标

解决：SPU/SKU 如何获得永久唯一身份；款号/颜色/尺码/条码由谁定义；领猫、聚水潭、渠道如何映射至集团商品；哪些字段可改、哪些发布后永久冻结；商品生命周期统一；下游用什么主键引用商品；Event Payload 使用什么字段；Agent 查询商品使用什么稳定模型。

## 2. Product Master 边界

只负责四类：**Identity（身份）、Classification（分类）、Variant（颜色尺码变体）、Lifecycle（生命周期）**。不负责：BOM、样衣、工艺、生产跟单、供应商报价、采购订单、库存、订单、营销详情页、图片文件、渠道平台商品、财务成本（分属领猫/DAM/Channel/OMS/IMS/财务）。

## 3. 核心逻辑模型

```
Brand → Product Line → Product SPU（Year/Season/Wave/Category/Gender）→ SKU[]（Color/Size/Barcode[]）
```

核心实体：product_spu、product_sku、product_brand、product_line、product_category、product_year、product_season、product_wave、product_color、product_color_family、product_size、product_size_group、product_barcode、product_lifecycle_history、entity_mapping、product_change_log。

## 4–6. 主键与编码设计

- **技术主键**：UUIDv7（spu_id/sku_id/brand_id/color_id/size_id）。永久不可修改、不携带业务含义、不依赖自增、不因品牌/年份/迁移变化；下游优先保存该 ID。
- **集团业务编码**：spu_code（SPU-2027-00001823）、sku_code（SKU-2027-00018231）。系统自动生成、全局唯一、APPROVED 后不可修改、不编码颜色/尺码含义、不使用领猫/JST 内部 ID。
- **variant_display_code**（如 AW27JK001-BLK-M）：颜色尺码组合展示值，供人阅读，非主键，可重算。
- **款号与 SPU 分离**：style_no（如 AW27JK001）≠ spu_id ≠ spu_code；款号可能历史重复、编码规则调整、品牌重号；集团所有系统关联只使用 spu_id/sku_id。

## 7–12. product_spu 字段模型

- **身份字段**：spu_id（UUIDv7，必填，不可改）、spu_code（varchar40，APPROVED 后不可改）、style_no（varchar80，必填，受控）、product_name（varchar200，可改）、short_name、source_type（LINGMAO/MANUAL/IMPORT/SYSTEM，正常流程 LINGMAO）、source_ref（原始立项标识）。
- **归属字段**：brand_id、product_line_id、category_id、year_id、season_id、wave_id、gender_code（WOMEN/MEN/UNISEX/KIDS_GIRL/KIDS_BOY/KIDS_UNISEX/OTHER）。
- **product_type**：APPAREL / FOOTWEAR / ACCESSORY / BAG / HOME / OTHER（V1 重点 APPAREL，模型不写死）。
- **variant_schema**：COLOR_SIZE / COLOR_ONLY / SIZE_ONLY / NONE。
- **生命周期三状态（禁止合并成一个字段）**：lifecycle_status（DRAFT/REVIEWING/APPROVED/ACTIVE/CLEARANCE/DISCONTINUED/ARCHIVED）、sellable_status（NOT_SELLABLE/SELLABLE/SUSPENDED/STOPPED）、development_status（PLANNED/DEVELOPING/CONFIRMED/CANCELLED）。
- **管理字段**：created_at/by、updated_at/by、approved_at/by、version（乐观锁，修改携带 expected_version）、data_status。

## 13–18. product_sku 字段模型

- 身份字段：sku_id（UUIDv7）、sku_code（APPROVED 后不可改）、spu_id、color_id/size_id（视 schema，APPROVED 后不可改）、variant_key（自动生成，如 COLOR=BLACK|SIZE=M）、variant_display_code（自动计算）。
- **唯一约束 UNIQUE(spu_id, variant_key)**：一个 SPU 不允许产生两个"黑色 M"。
- sku_status：DRAFT / ACTIVE / SUSPENDED / DISCONTINUED / ARCHIVED。SPU ACTIVE 不代表每个 SKU 都 ACTIVE。
- 物流基础字段（商品标准物理属性，非库存）：base_uom、net_weight_g、gross_weight_g、pack_length_mm、pack_width_mm、pack_height_mm（V1 可空，预留给 WMS/JST/TMS）。
- **UOM V1 限制**：PCS / SET / PAIR，服装标准 PCS；禁止各系统出现 件/Piece/pcs/PC/个 等不同编码。

## 19–24. Brand / Product Line / Category / Year / Season / Wave

- **product_brand**：brand_id、brand_code（UNIQUE，如 BR-A）、brand_name、brand_name_en、status。
- **product_line**：product_line_id、brand_id、line_code、line_name、status、sort_order（如 女装/男装/运动/童装）。
- **product_category**：树形结构（category_id、parent_id、category_code、category_name、category_level、category_path、is_leaf、status、sort_order）；**只有 is_leaf=true 的类目允许绑定 SPU**。
- **product_year**：year_id、year_code（2027）、year_name、sort_order、status。
- **product_season**：集团统一编码 SPRING/SUMMER/AUTUMN/WINTER/SPRING_SUMMER/AUTUMN_WINTER/ALL_SEASON；品牌展示名（AW/FW/秋冬）允许不同但映射到集团标准 Season。
- **product_wave**：wave_id、brand_id、year_id、season_id、wave_code（SS27-W01）、wave_name（春一波）、launch_date、end_date、sort_order、status。Wave 不能使用自由文本。

## 25–32. Color / Size

- **Color Family（标准色系）**：BLACK/WHITE/GREY/RED/ORANGE/YELLOW/GREEN/BLUE/PURPLE/PINK/BROWN/BEIGE/MULTI/METALLIC/OTHER。
- **product_color**：color_id、color_code（BLK01）、color_name、color_name_en、color_family_id、hex_reference、status、sort_order。
- **品牌颜色与标准颜色分离**：允许品牌用 午夜黑/曜石黑/经典黑，映射到 BLACK；brand_color_name 存在商品关联层，不污染集团标准色族（后续可扩 product_spu_color）。
- **product_size_group**：WOMEN_TOP_CN、WOMEN_BOTTOM_CN、MEN_TOP_CN、KIDS_HEIGHT_CN、ONE_SIZE 等。
- **product_size**：size_id、size_group_id、size_code、size_name、normalized_size、sort_order、status。
- **尺码禁止自由输入**：SKU 不能直接存 size="M"，必须保存 size_id，展示名从 Size Master 获取（否则出现 M/m/Medium/中码/M码 五种数据）。

## 33–35. Barcode 模型

- 独立建表 **product_barcode**（不直接 product_sku.barcode）：barcode_id、sku_id、barcode、barcode_type（EAN13/EAN8/UPC_A/UPC_E/CODE128/INTERNAL/OTHER）、is_primary、status、valid_from/valid_to、created_at。
- 约束：有效条码 UNIQUE(barcode)；同一 SKU 只能有一个 ACTIVE primary barcode，允许历史旧条码/新条码/供应链内部条码同时存在。

## 36–37. 价格基础模型

- 单独建表 **product_base_price**：price_id、spu_id、sku_id(nullable)、price_type（MSRP/STANDARD_RETAIL）、currency、amount、market、valid_from/valid_to、status。采购价、渠道价、活动价不进入此表。
- **Product Master 不管理成本**：禁止添加 purchase_cost/actual_cost/factory_cost/financial_cost，避免 PM 成本 82 vs 领猫 84 vs 财务 91 的口径冲突。

## 38–42. External ID Mapping

- 所有外部系统关联统一进入 **entity_mapping**，禁止在 product_sku 直接加 lingmao_id/jst_id/douyin_id/tmall_id。
- 字段：mapping_id、entity_type（SPU/SKU/BRAND/CATEGORY/COLOR/SIZE/BARCODE/LOCATION/SUPPLIER，OMS 可扩 ORDER/STORE/WAREHOUSE）、global_id、source_system、source_entity_type、source_id、source_code、mapping_status、is_primary、valid_from/valid_to、metadata_json、created_at/updated_at。
- **source_system 正式枚举**：PRODUCT_MASTER、LINGMAO、JST、POS、ERP、DAM、CHANNEL_DOUYIN、CHANNEL_TMALL、CHANNEL_JD、CHANNEL_PDD、CHANNEL_XHS、CHANNEL_WECHAT。禁止自由字符串。
- **唯一约束 UNIQUE(source_system, entity_type, source_id)**：同一外部 ID 不能映射两个不同集团 SKU。

## 43–46. 商品关系 / 审计 / Actor / 字段级写权限

- **product_relation**：REPLACEMENT / SUCCESSOR / PREDECESSOR / BUNDLE_COMPONENT / RELATED（旧款被新款替代不能改 SKU ID）。
- **product_change_log**：change_id、entity_type、entity_id、field_name、old_value、new_value、change_reason、actor_type、actor_id、source_system、trace_id、occurred_at。
- **Actor 统一**：USER / SERVICE / AGENT / SYSTEM（可查询哪些字段是 AI Agent 改过的）。
- **字段级写权限**：Product Master Service 允许 CREATE/UPDATE/APPROVE/ACTIVATE/DISCONTINUE；领猫 REQUEST_CREATE + READ；聚水潭 READ + 返回自身 JST Mapping（不能创建集团 SKU）；Channel/OMS/IMS READ ONLY；Agent 默认 READ ONLY。

## 47–50. 生命周期与冻结字段

- **SKU 禁止物理删除**：一旦被订单/库存/采购/生产/渠道引用，只能 SUSPEND/DISCONTINUE/ARCHIVE。
- **SPU 状态机**：DRAFT → REVIEWING → APPROVED → ACTIVE → CLEARANCE → DISCONTINUED → ARCHIVED；REVIEWING 可经 REJECT 返回 DRAFT。
- **APPROVED 后冻结字段**：SPU：spu_id、spu_code、brand_id、style_no（默认冻结）；SKU：sku_id、sku_code、spu_id、color_id、size_id、variant_key。颜色或尺码错了 → 停用错误 SKU + 创建新 SKU，不能修改原 SKU（否则历史订单被篡改）。
- **可持续修改字段**：product_name、wave_id、sellable_status、物流尺寸、标准价格、展示顺序；每次修改 version+1 并记录 Audit。

## 51–56. 创建流程与 API

- **标准流程**：领猫创建研发款式 → POST /product-requests → PM 生成 DRAFT SPU → 确定 Brand/Category/Season/Wave → 生成颜色尺码组合 → 生成 SKU → 审核 → APPROVED → product.spu.approved / product.sku.created → Integration Platform → 同步领猫/JST/Channel。
- **URL 主键规则**：永远使用 spu_id/sku_id（GET /api/v1/products/{spu_id}、GET /api/v1/skus/{sku_id}）；style_no/spu_code/barcode 只能作为查询条件。
- **查询 API**：GET /api/v1/products（filter: spu_code/style_no/brand_id/category_id/year_id/season_id/wave_id/lifecycle_status）；GET /api/v1/skus（filter: sku_code/spu_id/barcode/color_id/size_id/sku_status）。
- **Create DTO**：{brand_id, style_no, product_name, category_id, year_id, season_id, wave_id, product_type, variant_schema}。
- **SKU Generate DTO**：{colors:[...], sizes:[...]} 自动按矩阵批量生成（2×3=6 SKU），避免人工逐个创建。
- **标准返回**：SPU（spu_id/spu_code/style_no/product_name/brand/classification/lifecycle_status/version）；SKU（sku_id/sku_code/spu_id/color/size/barcode/sku_status/version）。

## 57–61. Event Contract 基础

- 商品领域事件统一使用 aggregate_id = spu_id / sku_id（不是款号/barcode/JST sku_id/领猫 style_id）。
- **最小 Envelope**：event_id、event_type、event_version、occurred_at、source_system、aggregate_type、aggregate_id、trace_id、payload。
- **product.sku.created 最小 Payload**：sku_id、sku_code、spu_id、spu_code、style_no、brand_id、category_id、year_id、season_id、wave_id、color_id、color_code、size_id、size_code、barcode、status。
- **Payload 原则**：只携带下游正确处理所需的最小稳定数据；其他信息通过 Product API 查询，避免字段增加导致所有消费者一起升级。

## 62–64. Projection / 索引 / 款号唯一规则

- **product_projection**：OMS/IMS/Channel 可建本地投影，只能通过 Product Event 更新，禁止本地用户修改。
- **索引**：product_spu：PK(spu_id)、UNIQUE(spu_code)、INDEX(style_no)、INDEX(brand_id, style_no)、INDEX(category_id)、INDEX(year_id, season_id, wave_id)、INDEX(lifecycle_status)；product_sku：PK(sku_id)、UNIQUE(sku_code)、UNIQUE(spu_id, variant_key)、INDEX(spu_id)、INDEX(color_id)、INDEX(size_id)、INDEX(sku_status)；product_barcode：UNIQUE(barcode)、INDEX(sku_id)。
- **款号唯一规则**：V1 不强制 style_no 全集团唯一，采用 brand_id+style_no 作为业务冲突检测组合（品牌内不重复）；历史复用场景增加 year_id 形成 brand_id+year_id+style_no（迁移兼容模式）。

## 65–70. 数据质量 / 版本 / 删除

- **实时校验（ERROR）**：SPU 无 Brand、SPU 无 Category、APPROVED SPU 无 SKU、COLOR_SIZE 商品 SKU 缺 Color/Size、重复 variant_key、重复 barcode、ACTIVE SKU 所属 SPU 已 ARCHIVED。
- Severity：ERROR 阻止发布；WARNING 允许保存但不能直接 APPROVE。
- 实体 version ≠ event_version（SKU version=18 表示修改 18 次；product.sku.updated event_version=2 表示 Schema 第二版）。
- 所有 Master Data：Soft Delete / Archive（data_status: ACTIVE/INACTIVE/ARCHIVED），不物理删除。

## 71–76. 下游引用规则

- **领猫映射方向**：以本模型为左侧标准（Product Master Field → Lingmao Field → Transform Rule）；重点映射 spu_id/spu_code/style_no/brand/category/year/season/wave/color/size/sku_code/barcode；BOM/样衣等只建立 spu_id↔lingmao_style_id 引用关系。
- **聚水潭映射方向**：Product Master → JST Item/SKU；重点映射 spu_id/sku_id/spu_code/sku_code/style_no/color/size/barcode/standard retail price；聚水潭 i_id/sku_id 只能进 entity_mapping。
- **OMS 引用规则**：Order Item 保存 sku_id + sku_code + 下单时快照（product_name_snapshot/color_name_snapshot/size_name_snapshot）。
- **IMS 引用规则**：库存 Ledger 只允许 sku_id 作为商品主键，不能使用 barcode/style_no/平台 sku_id。
- **Merchandising 引用规则**：配货/补货/调拨以 sku_id/spu_id 为标准商品维度。
- **Channel 引用规则**：Channel Product 保存 spu_id + sku_id[]；platform_product_id/platform_sku_id 通过 Mapping 关联。
- **Agent Tool 引用规则**：对用户展示可用款号/SKU 编码/条码/商品名，但 Tool 内部最终解析为 spu_id/sku_id（Product Resolver）。

## 77–79. 禁止事项 / 数据库边界 / 开发 Epic

- **数据模型禁止**：Product 表同时存库存/销售订单；SKU 直接保存领猫/JST ID；一个 barcode 字段覆盖所有条码；自由文本颜色/尺码；删除有历史订单的 SKU；修改历史 SKU 的颜色尺码；使用 style_no 作为数据库主键。
- **数据库边界**：product_platform_db，逻辑 Schema：master（product_spu/sku/color/size）、mapping（entity_mapping）、audit（product_change_log）；同一 PostgreSQL 也保持逻辑分离。
- **开发 Epic**：PM-001 基础 Reference Master / PM-002 SPU 管理 / PM-003 SKU Matrix（颜色×尺码批量生成）/ PM-004 Barcode / PM-005 External Mapping / PM-006 Product API / PM-007 Product Events / PM-008 Audit & Version / PM-009 Data Quality / PM-010 Product Resolver。

## 80–81. 验收基准与冻结决策

**验收至少证明 10 条**：①1 SPU + 5 Colors + 6 Sizes = 30 SKU；②30 SKU 均唯一 sku_id/sku_code/variant_key；③能同时建立领猫与 JST 映射但不保存第三方主键字段；④领猫/JST 不能制造未经 PM 注册的集团 SKU；⑤重复 Color+Size 无法生成第二 SKU；⑥Barcode 重复被拒绝；⑦SKU APPROVED 后 Color/Size/SPU 不能直接修改；⑧所有变更有 actor/before/after/time/reason/trace_id；⑨Event 可重复消费不产生重复商品；⑩通过 style_no/sku_code/barcode/领猫 ID/JST ID 均能 Resolve 到统一 spu_id/sku_id。

**冻结关键决策**：spu_id/sku_id = 集团商品永久技术身份；spu_code/sku_code = 集团业务编码；style_no = 业务款号 ≠ 主键；外部系统 ID 统一进 entity_mapping；颜色尺码必须用 Master ID 禁自由文本；同一 Variant Key 只能一个 SKU；已使用 SKU 禁改身份字段禁物理删除；PM 不拥有 BOM/库存/订单/采购/成本；所有 Event/OMS/IMS/Channel 统一用 spu_id/sku_id；**Product Master 决定的不是"这个商品有多少库存、卖多少钱、在哪个平台销售"，而是"这个商品是谁"。**

## 82. 对后续文档的直接约束

- 《领猫 × 聚水潭 × Product Master 数据映射 V1.0》：只解决三套字段映射（Source/Transform/Target/Sync Direction/Conflict Rule/Sync Frequency），不再讨论集团标准字段叫什么。
- 《Product Domain Event Contract V1.0》：直接使用 spu_id、sku_id、spu_code、sku_code、brand_id、category_id、year_id、season_id、wave_id、color_id、size_id、barcode、lifecycle_status、version 作为商品事件契约基础词汇。
