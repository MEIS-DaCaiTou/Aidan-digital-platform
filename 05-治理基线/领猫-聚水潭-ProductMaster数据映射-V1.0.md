# 领猫 × 聚水潭 × Product Master 数据映射 V1.0

> 文档级别：核心数据集成与迁移基线
> 上位依据：《服装集团业务事实权威矩阵 TO-BE V1.0》《Product Master 字段级模型 V1.0》
> 下游约束：Product Domain Event Contract、Lingmao Adapter、JST Adapter、Product API、External ID Mapping、数据迁移程序、数据质量中心、OMS/IMS/Channel Product Projection
> 版本：V1.0（已冻结）
> 说明：Product Master 字段和同步规则现已冻结；领猫具体 API JSON Key 因公开资料不足，先以业务字段语义标识，待拿到企业实例 OpenAPI/报文样例后只补技术字段名，不再改集团标准模型。

## 1–2. 文档目标与同步拓扑

解决三套商品模型（领猫 SCM ↕ Product Master ↕ 聚水潭 JST）的：Source、Transform、Target、Sync Direction、Conflict Rule、Event 六个核心问题。冻结后任何 Adapter 不允许自行重新解释 SKU/颜色/尺码/款号/条码含义。

```
研发业务 → 领猫 SCM → Product Request/R&D Ref → Product Master（唯一商品身份）
→ 领猫（R&D 投影）| 聚水潭（执行投影）| Channel（渠道投影）
```

逐步废除"领猫直接定义聚水潭集团 SKU 身份"；保留的只应是 Product Master → JST Adapter → 聚水潭。领猫现有"推送聚水潭"及"自定义款式 SKU 编码"能力，迁移时必须重点识别这条既有链路。

## 3–4. 三套模型身份定义与映射分层

- **Product Master**：spu_id/sku_id = 永久技术主键；spu_code/sku_code = 业务编码。
- **领猫**：围绕款式、款号、款式 SKU、颜色、尺码、吊牌信息、年份、季节、分类、性别及 BOM/样衣/物料/生产；公开页面未给出完整 OpenAPI JSON 字段字典，本文件以业务字段语义表示，技术 Key 待企业实例 API 文档补齐。
- **聚水潭**：官方通用字段已明确 i_id（款式编码）、sku_id（商品编码）、shop_i_id（店铺款式编码）、shop_sku_id（店铺商品编码）、shop_id、wh_id；i_id/sku_id 属普通商品资料体系，shop_i_id/shop_sku_id 属店铺商品资料体系。
- **映射分层**：L1 Product Identity / L2 Classification / L3 Variant / L4 Barcode / L5 Lifecycle / L6 Commercial Base / L7 External Mapping / L8 Technical Metadata。库存、订单、BOM、成本、渠道 Listing 不得塞进 Product Master 商品映射。

## 5–6. 方向代码与冲突规则

- 方向：L→P、P→L、J→P、P→J、L↔P、P→L/J（广播）、MIGRATION、NO_SYNC。
- 冲突规则：CR-01 PM 权威外部覆盖禁止；CR-02 领猫研发事实权威不写入 PM；CR-03 JST 执行事实权威不反写 PM；CR-04 无法唯一映射进人工 Matching Queue；CR-05 已 APPROVED 身份字段冲突禁 UPDATE 创建新 SKU；CR-06 Barcode 全局重复阻断；CR-07 外部枚举先经 Mapping Dictionary；CR-08 Legacy 数据保留 Alias 但统一生成 global ID；CR-09 空值不得覆盖 PM 非空权威值；CR-10 外部停用只产生异常不自动停用集团商品；CR-11 PM APPROVED 前禁止同步正式商品；CR-12 下游不存在则 Create，存在则按 global mapping Upsert。

## 7–12. SPU/Brand/Category/Year/Season/Wave/Gender 映射

- **SPU Identity**：spu_id（PM 生成 UUIDv7，P→L/J Mapping）；spu_code（PM 生成业务编码）；style_no（领猫款号 → 标准化 → PM → JST i_id 候选映射，CR-01/08）；product_name（Unicode Normalize + Trim）；source_type 固定 LINGMAO；source_ref（领猫款式内部 ID → Mapping，CR-08）。**JST i_id ≠ spu_id**：真正系统关联是 spu_id ↔ entity_mapping ↔ JST i_id。
- **款号转换**：trim → 全角转半角 → 去不可见字符 → 保留合法 "-/_/" → normalize case → normalized_style_no；禁止自动删除年份/品牌前缀/重生成。Brand A/2025/ABC001 与 Brand A/2026/ABC001 必须保留两个 spu_id，不能因 style_no 一样合并。
- **Brand**：brand_id 唯一关联键（禁止以品牌名称关联）；brand_code/brand_name 通过 Mapping Dictionary 下发（P→L/J）。
- **Category**：category_id/category_code/category_path 用 Taxonomy Mapping（L→P→J）；JST 类目 ≠ 集团分类树，需 global_category_id ↕ lingmao_category_id ↕ jst_category_id 映射。
- **Year/Season/Wave**：year_id/season_id/wave_id 通过 Dictionary（L→P→J）；JST 自定义属性（其他属性 1-5）只能作为临时 Projection Field，不能成为集团年份/季节/波段权威源。
- **Gender**：领猫性别 → Gender Mapping Dictionary → gender_code → JST Extension Attribute；未知值 UNKNOWN，不得自动猜测。

## 13–18. SKU/Color/Size 映射

- **SKU Identity**：sku_id（PM UUIDv7，P→L/J mapping）；sku_code（PM 标准编码或迁移 Alias，映射 JST sku_id）；spu_id（通过 i_id mapping）；color_id/size_id（Dictionary，L↔P→J）；variant_key（PM only 计算）；sku_status（Status Mapping，P→L/J）。**聚水潭把 sku_id 定义为商品编码而非集团技术 UUID**，代码中统一命名 global_sku_id / jst_sku_code 避免误解。
- **历史 SKU 迁移**：历史 SKU 尽量保留现有稳定业务编码作为 sku_code（降低切换风险）；新 SKU 用 PM 统一生成规则；无论新旧 sku_id(UUID) 必须重新生成；legacy_alias 保留（如 ABC001-BLK-M）。
- **Color**：color_id 统一；color_code/color_name 标准化后 P→L/J；color_family_id 由色系映射；brand_color_name 保留展示属性（CR-09）。禁止 "黑/黑色/经典黑/BLACK" 直接作为同一业务键。
- **SKC 处理**：增加 product_spu_color 逻辑关系（不一定生成全集团永久 SKC 主键），建议生成 spu_color_id，便于与领猫 SKC 图/颜色资料建立关系（Lingmao SKC ↔ spu_id + color_id）。
- **Size**：size_id 统一；size_group_id 按品类规则映射；size_code 标准编码下发；normalized_size PM only。例：领猫 165/88A → size_code=165/88A、normalized_size=M，不得为统一丢失原始业务尺码。

## 19–21. Variant Key / Barcode / Product Type

- variant_key 由 PM 自动生成（COLOR=<color_id>|SIZE=<size_id>），外部系统不允许提交；UNIQUE(spu_id, variant_key)；领猫导入两个完全相同"款号 ABC/黑色/M"不得自动生成两个集团 SKU，进入 DUPLICATE_VARIANT 数据质量队列。
- **Barcode**：barcode 经 Trim+Validator（L→P→J，CR-06）；barcode_type 解析（PM）；is_primary 规则下发（P→J）。重复条码直接 BLOCK，不允许"最后同步者覆盖"。
- **Product Type**：Lingmao Category → Product Type Rule → product_type → JST Projection。
- **Variant Schema**：从商品结构自动确定（COLOR_SIZE/COLOR_ONLY/SIZE_ONLY/NONE），最终字段 variant_schema 权威在 PM；APPROVED 后不可随意从 COLOR_SIZE 改为 COLOR_ONLY。

## 22–28. 生命周期/状态/价格/吊牌/内容边界

- **三套状态必须分离**：领猫主要映射 development_status；聚水潭消费 sellable_status/sku_status；领猫"已结案" ≠ PM DISCONTINUED（CR-10：外部停用只生成状态异常/建议）。
- **SPU 状态映射**：领猫研发创建→DRAFT/DEVELOPING（自动）；研发确认→development_status=CONFIRMED（自动）；PM 审核→APPROVED（PM 自身）；上市→ACTIVE（PM 审批）；清货→CLEARANCE（Merchandising 触发审批）；停产→DISCONTINUED（PM 审批）；聚水潭停用→不改变 PM（否）。
- **SKU 状态映射**：PM ACTIVE→JST 启用；PM SUSPENDED→JST 暂停/禁售 Projection；JST 人工禁用只产生 projection.drift.detected，不得反向停用 PM。
- **价格**：MSRP（领猫吊牌价）→PM→JST 商品标准价格；STANDARD_RETAIL（建议零售价）→JST 售价候选；采购价/生产成本/渠道促销价/实际成交价不属于 PM Base Price。
- **吊牌信息边界**：进入 PM：barcode、MSRP、必要身份字段；进入 Product Content：成分、洗涤说明、消费者展示信息；继续留在领猫：生产/吊牌制作相关业务数据；不能把整张吊牌表复制进 PM。
- **Product Content**：R&D Source Content → Content Normalization → Product Content（不进入 product_spu 核心身份表）。

## 29–33. JST 投影 / Entity Mapping 样例

- **普通商品 vs 店铺商品必须分开**：PM → JST 普通商品资料（i_id/sku_id）；Channel Product → JST 店铺商品资料（shop_i_id/shop_sku_id）。禁止 PM 直接管理 shop_i_id/shop_sku_id；shop_sku_id 属于 Channel Listing Mapping（global_sku_id → Channel Product SKU → JST shop_sku_id → Platform SKU ID）。
- **JST Projection 模型**：jst_product_projection（global_spu_id、global_sku_id、jst_i_id、jst_sku_id、last_product_version、sync_status、last_synced_at），全部进入 entity_mapping 或 Adapter Projection。
- **Mapping 样例**：SPU：global_id=spu UUID，entity_type=SPU，source_system=LINGMAO，source_id=LM_STYLE_8831，source_code=AW27JK001；同 global_id 第二条 source_system=JST，source_id=AW27JK001。SKU：同一 global_sku_id 拥有领猫与聚水潭两条 External Mapping。

## 34–39. 同步模式 / 创建条件 / Adapter

- 同步模式：COMMAND（PM 主动要求创建/更新 Projection，如 CreateJstSku）、EVENT（商品完成后广播 product.sku.created）、RECONCILIATION（周期性核对）、MIGRATION（历史一次性迁移）。
- **PM→JST 创建条件**：SPU.lifecycle_status >= APPROVED、SKU.status eligible、Brand/Category/Color/Size mapped、SKU Code exists；否则 SYNC_BLOCKED。
- **PM→JST Upsert**：聚水潭普通商品资料上传接口支持外部系统→JST，单次最多 150 商品；Adapter 批量策略：Event → Task Queue → batch ≤ 150 → Upload，不能同步等待全部 JST 上传完成。
- **JST API 限流隔离**：所有 JST 调用经 JstAdapter（Token/Signature/Rate Limit/Retry/Circuit Breaker/Dead Letter）；聚水潭开放平台使用 partnerid/token/method/ts/sign 等公共认证参数；Product Service 禁止直接请求 open.erp321.com。
- **LingmaoAdapter**：负责款式读取、商品身份申请、PM ID 回写、状态同步、SKU 映射；Product Service 禁止直接理解领猫内部字段。
- **领猫 API 字段确认策略**：V1 冻结业务字段语义（款号/款式名称/年份/季节/分类/性别/颜色/尺码/SKU/吊牌条码），拿到企业实例接口文档后只补 lingmao_api_field，不得修改 PM Target/Transform Rule/Ownership。

## 40–45. 新商品流程 / 历史迁移

- **标准同步流程**：领猫研发款式建立 → product.identity.requested → PM → 生成 spu_id → Classification → SKU Matrix → APPROVED → product.spu.approved/product.sku.created → Integration Platform → Lingmao Adapter + JST Adapter。
- **允许领猫先建立研发款式**：研发款式 ≠ 正式集团商品（可能被取消/打样失败/未上市），但一旦需要正式集团 SKU 必须进入 PM。
- 增加集成状态：NOT_REGISTERED / REQUESTED / REGISTERED / REJECTED / SYNCED。
- **历史匹配 Key 优先级**：Level1 barcode 唯一匹配 → Level2 brand+style_no+color_code+size_code → Level3 existing sku_code → Level4 manual review。禁止只凭商品名称自动合并。
- **匹配置信度**：MATCHED_EXACT（才允许完全自动迁移）/ MATCHED_HIGH / MATCHED_REVIEW / UNMATCHED / CONFLICT。

## 46–52. 冲突场景

- **历史数据冲突**：领猫 AW001-BLK-M 与 JST AW001-BLACK-M 但 Barcode 相同 → 一个 global_sku_id + 两个 external mappings，不因编码不同创建两个集团 SKU。
- **Barcode Conflict**：领猫 SKU-A 与 JST SKU-B 同一 barcode 但 Style/Color/Size 完全不同 → IDENTITY_CONFLICT 人工处理，禁止自动合并。
- **Color Conflict**：黑/黑色/BLACK 经 Mapping Dictionary 到同一 color_id；但曜石黑/炭黑/午夜黑默认不能自动合并为同一业务颜色 SKU（color_family 可相同，color_id 仍不同）。
- **Size Conflict**：165/88A 与 M 可 normalized_size=M，但 SKU 身份仍根据原业务 size_id，不因 normalized_size 一样自动合并。
- **Drift**：JST 中商品名称/SKU 编码/颜色/尺码被人工修改 → product.projection.drift_detected → 自动修复或人工队列，绝不 JST→PM 覆盖。
- **Drift 类型**：MISSING_PROJECTION / VALUE_MISMATCH / ORPHAN_EXTERNAL_PRODUCT / IMMUTABLE_FIELD_CHANGED / STATUS_MISMATCH。
- **Orphan**：JST 存在 PM 没有的 SKU（ORPHAN_EXTERNAL_PRODUCT）：一期迁移期允许存在+必须治理，正式切换后禁止新产生；领猫正式商品 SKU 在 PM 不存在（切换后 ILLEGAL_PRODUCT_IDENTITY，研发草稿除外）。

## 53–60. 删除/时间/幂等/事件

- 外部删除（领猫/JST 删除）不触发 PM DELETE，只产生 external_projection.deleted；PM 商品 ARCHIVE 而非物理删除。
- 更新时间：source_updated_at 与 PM updated_at 分别保存；Mapping 增加 last_source_updated_at、last_synced_at；Adapter 保存 source_revision 避免旧消息覆盖新数据。
- **幂等规则**：任何商品同步使用 idempotency_key（<event_id>:<target_system>，如 01999...:JST），确保 Event Retry 不重复创建商品。
- **Event 主键标准**：SPU：aggregate_type=SPU、aggregate_id=spu_id；SKU：aggregate_type=SKU、aggregate_id=sku_id；禁止 aggregate_id=style_no 或 jst_sku_id。
- **第一批事件候选**：product.identity.requested、product.spu.created/updated/approved/status_changed、product.sku.created/updated/status_changed、product.barcode.assigned/changed、product.mapping.created/updated、product.projection.synced/sync_failed/drift_detected。
- product.sku.created 消费方：Lingmao Adapter、JST Adapter、OMS Projection、IMS Projection、Channel Platform、Data Platform（各取所需字段）；product.mapping.created Payload 至少 {entity_type, global_id, source_system, source_id}。

## 61–64. Event 传播边界 / 最小投影

- **禁止传播字段**：BOM、采购价、实际成本、库存、生产订单、销售订单、会员、平台消费者数据（属于其他 Domain）。
- **PM→JST 最小投影**：style_no/JST i_id、sku_code/JST sku_id、product_name、brand、category、color、size、barcode、status、MSRP/标准零售价（若业务需要）。不把 Product Content 全塞进 JST。
- **PM→领猫最小回写**：global_spu_id、spu_code、global_sku_id、sku_code、registration status；领猫保留 BOM/设计/生产/成本/供应商完整所有权。
- **领猫→PM 最小输入**：source_style_id、style_no、product_name、brand、category、year、season、wave、gender、colors[]、sizes[]、existing_barcode[]——是 Identity Request Input，不是"领猫直接写 Product Master"。

## 65–71. API / Resolver / 审计

- **POST /api/v1/product-identity-requests**：请求 {source_system, source_id, style_no, brand_code, category_code, year, season, colors:[{source_code, source_name}], sizes:[]}；响应 {request_id, spu_id, spu_code, status: REGISTERED, skus:[{sku_id, sku_code}]}。领猫必须保存这些 Group ID。
- **统一 Resolver**：支持 style_no / sku_code / barcode / lingmao_id / jst_i_id / jst_sku_id → 统一返回 spu_id/sku_id；输入命中多个（如 AW001 在 Brand A 与 Brand B）返回 AMBIGUOUS，禁止"选第一条"。
- **集成同步审计**：integration_sync_log（sync_id、source_system、target_system、entity_type、global_id、event_id、operation、request_hash、response_code、status、attempt、started_at、completed_at、error_code、error_message）。
- 同步状态：PENDING / PROCESSING / SUCCESS / FAILED / RETRYING / DLQ / SKIPPED。
- **不能把 HTTP 200 当同步成功**：Adapter 必须判断 Vendor Business Response；同步成功后才建立 entity_mapping。

## 72–76. Reconciliation / 切换阶段 / 来源优先级

- **Reconciliation**：每天至少执行 PM↔Lingmao、PM↔JST；检查 Missing/Orphan/Code mismatch/Status mismatch/Barcode mismatch/Mapping mismatch；输出 data_quality_issue，由规则决定 AUTO_FIX / MANUAL_FIX / IGNORE_WITH_REASON；不负责自动改主数据。
- **切换阶段**：Stage 0 Discover（读取不写）→ Stage 1 Mapping（生成 global ID + external mapping，不改原业务）→ Stage 2 Dual Projection → Stage 3 Controlled Creation（新商品只能通过 PM 注册）→ Stage 4 Enforcement（关闭 JST 人工新建集团 SKU、领猫未经 PM 注册直接推 JST 商品）。
- **迁移期来源优先级**：历史商品身份候选领猫优先（更靠近服装研发源头），但 Barcode 需两边交叉验证；JST 已有实际交易 SKU 不能因与领猫不同直接删除；最终匹配→合并 Mapping，不暴力覆盖。
- **历史交易保护**：JST SKU-A 已有历史订单和库存，即使最终应对应 PM SKU-B，只能建立 Mapping，不能修改历史交易主键。

## 77–81. 字段权威总表 / Adapter 职责 / Event 输入

- **字段权威**：spu_id/sku_id/spu_code/sku_code/款号/Brand/Category/Year/Season/Wave/Gender/Color/Size/Barcode/Lifecycle → PM；R&D 状态/BOM/样衣/采购/生产/成本源数据 → 领猫；JST 普通商品投影/店铺商品/shop_i_id/shop_sku_id → JST/Channel；平台商品 ID → 平台；库存 → IMS/JST（另域）；订单 → OMS/JST（另域）。
- **Lingmao Adapter**：read_rnd_style、request_product_identity、sync_product_identity_back、read_product_changes、reconcile_product_mapping。
- **JST Adapter**：create/update/read_product_projection、map_jst_i_id、map_jst_sku_id、reconcile_product_projection。
- **Adapter 禁止**：创建 global SKU、修改 PM DB、自己创建颜色/尺码 Master、自行判断 SKU 合并、自行覆盖 Barcode 冲突、自行修改生命周期。
- **Event Contract 直接输入**：spu_id、spu_code、style_no、sku_id、sku_code、brand_id、category_id、year_id、season_id、wave_id、color_id、color_code、size_id、size_code、barcode、lifecycle_status、sku_status、version——不再讨论字段命名。
- **Event 不引用第三方主键作为 Aggregate**：第三方 ID 统一放 external_refs（{LINGMAO: ..., JST: ...}），aggregate_id 永远是集团 ID。

## 82–88. 第一批表 / Mapping Dictionary / 数据质量 / Agent

- **第一批表**：mapping.entity_mapping、mapping.mapping_dictionary、mapping.mapping_rule、integration.sync_task、integration.sync_log、integration.sync_error、quality.data_quality_issue。
- **Mapping Dictionary**：dictionary_type、source_system、source_value、global_id、global_code、transform_type、status，覆盖 Brand/Category/Season/Gender/Color/Size/UOM/Status。
- **映射变更必须版本化**：valid_from/valid_to/version，不能直接覆盖历史。
- **Data Quality 第一批规则**：DQ-PROD-001 领猫正式 SKU 未映射 PM；002 JST SKU 未映射 PM；003 重复 Barcode；004 同 SPU 重复 Color+Size；005 JST SKU 身份字段与 PM 不一致；006 领猫款号与 PM 不一致；007 External Mapping 一对多冲突；008 APPROVED SKU 缺少 JST Projection。
- **Agent 读取规则**：Agent 不能直接查询领猫/JST 判断"商品是谁"；必须 Agent → Product API → Product Master；需要供应链资料时经 global_spu_id → Lingmao Adapter。推荐 Agent Resolver Tool：resolve_product（输入款号/SKU/Barcode/领猫 ID/JST 商品编码，返回 global_spu_id/global_sku_id/confidence/mapping_status）。

## 89–96. 验收场景与冻结决策

**六个验收场景**：
1. 领猫建立 AW27JK001 黑/白 × S/M/L → 1 SPU + 6 SKU，PM 分配 spu_id + 6 sku_id，领猫成功回写 + JST 成功生成 6 个普通商品 SKU；
2. JST 人工创建 AW27JK001-RED-M 而 PM 不存在 → 识别 ORPHAN_EXTERNAL_PRODUCT，不自动创建 PM SKU；
3. PM 黑色 M 已 APPROVED，领猫改为黑色 L → REJECT IDENTITY UPDATE，必须新建 SKU 或人工处理；
4. 同一 Barcode 领猫→SKU-A、JST→SKU-B → IDENTITY_CONFLICT，不得自动合并；
5. product.sku.created 发布后 JST 首次超时，Retry 时 JST 实际已创建 → 通过 Idempotency + Reconciliation 不产生重复商品；
6. 历史 30 万 SKU 迁移完成后：Mapped Rate ≥ 99.x%、Duplicate Barcode 明确清单、Unmapped/Orphan JST/Orphan Lingmao 明确清单、Manual Review Queue 可追踪（上线门槛由迁移计划单独冻结）。

**冻结的架构决策**：①领猫不得再作为集团 SKU 身份权威；②聚水潭不得作为集团 SKU 身份权威；③Product Master 是 SPU/SKU/Color/Size/Barcode 唯一权威；④领猫研发款式允许早于正式集团商品存在；⑤正式 SKU 产生必须经过 Product Master；⑥JST i_id/sku_id 只是 External Identity；⑦shop_i_id/shop_sku_id 属于 Channel 域；⑧所有外部 ID 通过 entity_mapping 维护；⑨历史交易不因主数据治理重写外部交易主键；⑩任何身份字段冲突不得静默覆盖。

**最终数据链**：领猫 SCM（R&D Source）→ Identity Request → Product Master（Global Truth）→ 领猫（R&D Projection）/ 聚水潭（ERP Projection）/ Channel（Listing）→ Product Event → OMS / IMS / Merchandising → Data / Agent。

以后无论更换领猫、聚水潭、渠道平台，集团核心 spu_id/sku_id 及商品身份模型都不发生变化——这就是本次数据治理真正需要达到的目标。
