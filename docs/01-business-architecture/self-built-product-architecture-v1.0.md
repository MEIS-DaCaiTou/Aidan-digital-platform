# 服装集团自研系统产品架构 V1.0

> 文档定位：集团自研数字业务平台顶层产品与研发架构（可直接交给产品经理、UI/UX、架构师和 Codex）
> 目标使用者：产品经理、业务负责人、UI/UX、技术架构师、后端、前端、数据团队、AI 团队、Codex
> 覆盖范围：商品中心、PIM、DAM、渠道商品发布中心、OMS、IMS、配货/补货/调拨平台、数据平台/BI/AI
> 冻结日期：2026-09

## 1. 总体产品定位

不重新开发 ERP、PLM、WMS、MES。自研平台核心职责：将集团的商品、素材、渠道、订单、库存、门店和经营数据统一起来，形成跨品牌、跨渠道、跨仓库、跨系统的数字业务操作系统。

```
集团经营/BI/AI → 数据平台（Lakehouse/Metrics）→ 集团数字业务平台
（商品中心 PIM DAM 渠道发布 OMS IMS 商品运营平台）
→ API / Event Bus / CDC / MQ
→ PLM 研发设计 | ERP 财务采购 | WMS 仓储 | POS 门店 | 电商平台（抖音/天猫等）
```

## 2. 统一产品原则

**一个业务对象只能有一个权威源：**

| 数据对象 | 权威源 |
|---|---|
| 款式研发数据 | PLM |
| 正式商品 SPU/SKU | 商品中心 |
| 商品销售信息 | PIM |
| 图片/视频/PSD | DAM |
| 渠道商品 | 渠道商品中心 |
| 销售订单 | OMS |
| 实物库存 | WMS |
| 可售库存 ATP | IMS |
| 成本/财务 | ERP |
| 经营指标 | 数据平台 |

## 3. 核心角色体系

集团管理员、品牌负责人、商品企划、商品经理、设计师、商品资料员、摄影/视觉、电商运营、渠道运营、商品运营、门店运营、仓储人员、客服、售后人员、财务人员、数据分析师、AI 运营人员、IT 管理员、审计人员。

权限维度必须支持：集团 → 品牌 → 法人 → 事业部 → 渠道 → 区域 → 门店 → 仓库。

## 4. 产品一：商品中心 Product Master

- **定位**：所有销售系统的商品唯一权威源。负责 Identity（身份）、Classification（分类）、Variant（颜色尺码变体）、Lifecycle（生命周期）。不负责设计研发、营销标题、库存、订单。
- **页面**：商品工作台、SPU 列表/详情、SKU 列表/详情、品类中心、属性中心、颜色中心、尺码中心、条码中心、品牌中心、商品审核中心、商品变更记录、数据质量中心。
- **数据库对象**：product_spu、product_sku、product_category、product_category_attribute、product_attribute、product_attribute_value、product_color、product_color_group、product_size、product_size_group、product_barcode、product_brand、product_season、product_wave、product_year、product_lifecycle、product_relation、product_version、product_audit_log。
- **生命周期**：DRAFT → REVIEWING → APPROVED → ACTIVE → CLEARANCE → DISCONTINUED；不可物理删除，只能停用/冻结/归档。
- **核心 API**：GET/POST /api/products/spu、GET/PATCH /api/products/spu/{id}、GET/POST /api/products/spu/{id}/skus、GET /api/products/skus/{id}、GET /api/product-categories|colors|sizes、POST /api/products/import、POST /api/products/{id}/approve|activate。
- **领域事件**：product.spu.created/updated/approved/activated、product.sku.created/updated/disabled、product.lifecycle.changed。

## 5. 产品二：PIM 商品信息中心

- **定位**：解决“这个商品怎么卖”（商品名称、卖点、参数、详情、成分、洗护、模特信息、尺码说明、渠道商品描述）。
- **页面**：商品资料工作台、商品资料列表、商品编辑器、参数模板、卖点管理、尺码说明、洗护信息、商品详情搭建、多语言中心、商品资料审核、数据完整度、AI 文案中心。
- **数据库对象**：pim_product、pim_content、pim_content_version、pim_attribute_value、pim_description、pim_selling_point、pim_size_guide、pim_care_instruction、pim_composition、pim_channel_content、pim_translation、pim_content_template。
- **API**：GET/PUT /api/pim/products/{spuId}、PUT .../description|attributes|selling-points、POST .../generate-copy、GET .../completeness。
- **事件**：pim.product.created/updated、pim.content.updated/approved、pim.product.completed。
- **核心流**：商品中心 product.spu.activated → PIM 自动创建资料 → 资料员完善 → 审批 → pim.content.approved → 渠道发布中心。

## 6. 产品三：DAM 数字资产中心

- **定位**：集团所有商品素材（主图、SKU 颜色图、模特图、平铺图、细节图、场景图、详情页图片、视频、PSD、AI、PDF、源设计稿）统一进入 DAM，散落于员工电脑/微信群/NAS/设计师文件夹/电商运营电脑的情况不允许。
- **页面**：素材工作台、素材库、商品素材、SKU 素材、文件夹视图、标签视图、素材详情、批量上传、素材版本、图片处理中心（裁剪/尺寸/水印）、素材审核、素材使用追踪、重复素材检测、AI 素材中心。
- **数据库对象**：dam_asset、dam_asset_file、dam_asset_version、dam_asset_relation、dam_asset_tag、dam_asset_metadata、dam_asset_usage、dam_asset_folder、dam_upload_task、dam_transform_task。文件进对象存储，数据库只存 asset_id/storage_key/hash/mime_type/width/height/duration/metadata。
- **API**：POST /api/dam/assets/upload、GET /api/dam/assets、GET /api/dam/assets/{id}、POST .../relations|transform、POST /api/dam/assets/batch-transform、GET /api/dam/products/{spuId}/assets。
- **事件**：dam.asset.uploaded/processed/approved/updated/archived。

## 7. 产品四：渠道商品发布中心（重点自研）

- **定位**：集团只维护一次商品，通过渠道商品发布中心转换成抖音/天猫/京东/拼多多/小红书/视频号/官网商品。
- **产品模型（必须区分三个对象）**：集团商品 Product → 渠道商品 Channel Product → 渠道平台商品 Platform Listing。不能把抖音商品 ID 直接写进集团商品表。
- **页面**：渠道商品工作台、店铺管理、渠道授权、渠道模板、商品转换、SKU 映射、素材编排、图片替换（单张/多张批量）、商品裂变（一个母商品生成多个商品）、批量编辑、发布任务、发布队列、发布错误中心、渠道商品列表、商品更新中心、商品下架中心。
- **数据库对象**：channel、channel_store、channel_authorization、channel_product、channel_product_sku、channel_listing、channel_listing_sku、channel_attribute_mapping、channel_category_mapping、channel_publish_template、channel_publish_task、channel_publish_task_item、channel_publish_error、channel_product_variant。
- **API**：GET /api/channels、GET /api/channel-stores、POST /api/channel-products、POST .../generate|batch-update、POST .../{id}/publish|unpublish、POST /api/channel-products/batch-publish、GET /api/publish-tasks/{id}。
- **平台 Adapter**：/channel-adapter/douyin、/tmall、/jd、/pdd —— 平台差异封装在 Adapter 层。
- **事件流**：pim.content.approved + dam.asset.approved → channel.product.ready → 发布任务 → 平台 API → channel.listing.published；失败 channel.publish.failed → 错误中心 → 人工修复/自动重试。

## 8. 产品五：OMS 全渠道订单中心

- **定位**：统一接入和履约编排；WMS 负责怎么拣/怎么装/怎么出库，OMS 负责谁来发/从哪里发/什么时候发/订单状态。
- **页面**：订单工作台、全部订单、订单详情、待审核订单、待分仓订单、异常订单、缺货订单、拆单管理、合单管理、发货管理、取消管理、售后单、退款单、换货单、履约规则。
- **数据库对象**：order、order_item、order_address、order_payment、order_discount、order_source、order_fulfillment、order_fulfillment_item、order_shipment、order_package、order_status_history、after_sale_order、refund_order、return_order、exchange_order。
- **状态机**：CREATED → PAID → ALLOCATING → ALLOCATED → FULFILLING → SHIPPED → DELIVERED → COMPLETED；异常 CANCELLED/CLOSED/ON_HOLD。
- **核心 API**：POST /api/orders、GET /api/orders、GET /api/orders/{id}、POST .../allocate|cancel|hold|release|split|ship、POST /api/after-sales、POST /api/refunds。
- **事件**：order.created/paid/allocated/fulfillment.created/shipped/delivered/completed/cancelled/return.requested、order.inventory.reservation.requested。

## 9. 产品六：IMS 库存中心

- **定位**：IMS 不是 WMS。WMS 管仓库里的货放在哪里；IMS 管整个集团还有多少货能够卖。
- **库存模型**：ON_HAND 实物、AVAILABLE 可售、RESERVED 预占、LOCKED 冻结、IN_TRANSIT 在途、QUALITY_HOLD 质检、DAMAGED 残次、SAFETY 安全。维度 = SKU + Location（仓库/门店/虚拟仓/供应商仓）+ Inventory Type。
- **页面**：库存总览、SKU 库存、仓库库存、门店库存、ATP 查询、库存占用、库存锁定、在途库存、渠道库存、库存流水（Ledger）、库存差异、库存预警。
- **数据库对象**：inventory_location、inventory_balance、inventory_available、inventory_reservation、inventory_lock、inventory_in_transit、inventory_ledger、inventory_channel_allocation、inventory_snapshot。
- **Ledger 必建**：+100 PO_RECEIPT / -2 ORDER_RESERVE / +2 ORDER_RELEASE / -1 SHIPMENT / +10 TRANSFER_IN，保证可审计。
- **API**：GET /api/inventory、GET /api/inventory/availability、POST /api/inventory/reservations、DELETE /api/inventory/reservations/{id}、POST /api/inventory/locks|releases、GET /api/inventory/ledger。
- **事件**：inventory.received/changed/reserved/released/shipped/transferred/adjusted/availability.changed。
- **订单库存流**：order.paid → OMS → inventory.reserve → IMS → reservation.success → OMS 履约；失败 → 缺货处理。

## 10. 产品七：商品运营平台（配货/补货/调拨）

- **配货 Allocation**：配货工作台、首铺计划、门店等级（ABC）、门店商品画像、尺码模型、配货规则、配货模拟（What-if）、配货建议、配货审核、配货执行、配货效果。对象：allocation_plan/rule/store_profile/size_curve/suggestion/result。
- **补货 Replenishment**：补货工作台、缺货预警、安全库存、销售速度、补货建议、补货审批、补货结果。对象：replenishment_rule/policy/forecast/suggestion/order。
- **调拨 Transfer**：调拨工作台、智能调拨、门店调拨、仓间调拨、调拨审批、调拨在途、调拨收货、调拨分析。对象：transfer_request/order/order_item/shipment/receipt。
- **算法输入**：销售速度、库存、在途、门店等级、店铺面积、城市、气候、品类、价格带、颜色、尺码、历史售罄、促销计划、生命周期 → 首铺/补货/调拨/返单/降价建议。
- **API**：POST /api/allocation/plans、POST /api/allocation/plans/{id}/calculate、GET /api/allocation/suggestions、POST /api/replenishment/calculate、GET /api/replenishment/suggestions、POST /api/transfers、POST /api/transfers/{id}/approve。
- **事件**：allocation.calculated/approved/executed、replenishment.suggested/approved、transfer.created/shipped/received。

## 11. 产品八：数据平台 / BI / AI

- **架构**：业务系统 → CDC/MQ/API → ODS → Lakehouse → DWD → DWS → ADS → Metric Store → BI/AI。
- **数据域**：商品、销售、订单、库存、门店、会员、供应链、财务、渠道、营销。
- **指标中心**：所有 BI 不允许自行重新定义指标；每个指标有编码、中文名、英文名、计算公式、时间口径、组织口径、数据源、负责人、版本（GMV、Net Sales、Sell Through、Stock Turnover、Markdown Rate、Gross Margin、Return Rate、ATP、Weeks of Supply 等）。
- **BI 页面**：集团/品牌/商品/库存/渠道/门店/供应链/会员驾驶舱。
- **AI 平台**：AI Agent → AI Tool/API Layer → 业务服务（不直接连接生产库）。第一批 Agent：商品运营 Agent、商品资料 Agent、素材 Agent、渠道运营 Agent、经营分析 Agent。

## 12. 八大产品之间的数据主链

- 商品链：PLM → 商品中心 → PIM → DAM → 渠道发布 → 电商平台
- 订单链：渠道 → OMS → IMS → WMS → 物流 → 消费者
- 商品运营链：订单+库存+商品+门店 → 数据平台 → 配货/补货/调拨 → IMS → WMS
- 闭环：商品 → 渠道 → 订单 → 库存 → 销售 → 数据 → 决策 → 重新指导商品

## 13–17. 平台通用规则

- **事件总线**：核心 Topic：product/pim/dam/channel/order/inventory/allocation/member.events。标准 Envelope：event_id、event_type、event_version、occurred_at、source、tenant_id、brand_id、aggregate_id、trace_id、payload。所有事件可追踪、可重试、可幂等、可回放、有版本。
- **API 规则**：统一 /api/v1/；批量任务不得设计成长连接阻塞接口（POST /publish-tasks 返回 task_id，GET 查进度）。
- **任务中心**：async_task、async_task_item、async_task_error；状态 PENDING/RUNNING/PARTIAL_SUCCESS/SUCCESS/FAILED/CANCELLED。覆盖商品导入、素材上传、图片转换、渠道商品生成、批量发布、库存同步、数据导出、AI 批处理。
- **审批中心**：统一 Workflow Center（workflow_definition/instance/task/action），不每个系统重做审批；支持商品、素材、渠道发布、调拨、配货、价格审批。
- **操作审计**：audit_log 记录谁/什么时候/改了什么/改前/改后/哪个IP/哪个系统/为什么。库存调整、订单取消、商品价格、商品上下架必须 100% 可审计。

## 18–20. 前端导航 / 数据库分域 / 系统间禁止直接改库

- **前端导航**：首页 / 商品（商品中心、PIM、DAM、商品审核）/ 渠道（渠道商品、发布任务、店铺、渠道模板）/ 订单（全部订单、履约、异常、售后）/ 库存（库存总览、ATP、在途、库存流水、库存异常）/ 商品运营（配货、补货、调拨、商品生命周期）/ 数据（驾驶舱、商品分析、销售分析、库存分析、AI 助手）/ 系统（组织、权限、工作流、API、任务中心、日志）。
- **数据库分域**：product_db、pim_db、dam_db、channel_db、order_db、inventory_db、merchandising_db、platform_db、analytics（同一集群也逻辑隔离）。
- **禁止直接改库**：OMS 不能直接 UPDATE inventory_balance；渠道中心不能直接修改 product_sku；所有数据只能由权威服务写入。

## 21–23. 技术架构 / DAM 基础设施 / 搜索

- **技术架构形态**：第一阶段模块化领域架构 + 独立数据库边界 + 事件驱动；约 8～12 个业务服务：product-service、pim-service、dam-service、channel-service、order-service、inventory-service、merchandising-service、analytics-service、identity-service、workflow-service、task-service、integration-service。
- **DAM 基础设施**：Object Storage + CDN + Image/Video Processing + Hash + Metadata；源文件不存数据库（MinIO/S3 兼容/企业对象存储）。
- **搜索**：数据库 + Search Index（商品/素材/订单/渠道商品），数据库始终是权威数据源。

## 24–26. Codex 仓库结构与任务模板

- **仓库结构**：/apps（admin-web、product-service、pim-service、dam-service、channel-service、order-service、inventory-service、merchandising-service）；/packages（api-contracts、domain-types、event-contracts、ui-components、auth、logging、testing）；/docs（architecture、prd、database、api、events、ui）。API Contract 与 Event Contract 独立管理。
- **每个二级功能交给 Codex 必须包含**：功能目标、用户角色、页面、页面字段、操作、权限、状态机、数据实体、数据关系、API、输入输出、业务校验规则、领域事件、异常场景、审计要求、测试标准。
- **Codex 标准任务模板**：Epic（如 CHANNEL-003 渠道商品批量发布）：Business Goal、Roles、Pages、APIs、Entities、Events、Acceptance Criteria（500 商品批量提交、单商品失败不影响其他、失败重试、行为可审计、实时任务进度）。

## 27–29. 研发优先级 / 最终形态 / 研发文档树

- **优先级**：P0 核心底座（组织权限、商品中心、PIM、DAM、任务中心、审批中心、事件总线）；P1 渠道商品数字化（渠道中心、渠道商品、商品模板、素材编排、批量发布、抖音 Adapter）；P2 全渠道交易（OMS、IMS、售后、履约、预占、渠道库存同步）；P3 智能商品运营（配货、补货、调拨、生命周期、数据平台、BI、AI Agent）。
- **最终形态**：服装集团数字业务操作系统 = DATA/AI + Merchandising Intelligence + Commerce Core（OMS+IMS）+ Product Commerce（商品中心→PIM→DAM→渠道商品→渠道发布）+ Enterprise Platform（MDM/IAM/Workflow/Task/Audit/API/Event）+ External Enterprise Systems（PLM/ERP/WMS/POS/SRM/MES/CRM）。
- **核心闭环**：设计商品 → 建立商品 → 整理资料 → 整理素材 → 生成渠道商品 → 发布 → 产生订单 → 占用库存 → 履约 → 销售 → 数据分析 → AI 判断 → 补货/调拨/配货 → 重新指导商品经营。
- **研发文档树**：00 产品架构 / 01 商品中心 PRD / 02 PIM PRD / 03 DAM PRD / 04 渠道商品中心 PRD / 05 OMS PRD / 06 IMS PRD / 07 商品运营平台 PRD / 08 数据平台 PRD / 10 UI 设计规范 / 11 权限设计 / 12 数据模型设计 / 13 API 规范 / 14 Event 规范 / 15 工作流规范 / 16 异步任务规范 / 20 Codex 开发规范 / 21 前端开发规范 / 22 后端开发规范 / 23 数据库规范 / 24 测试规范。

**新增功能硬性要求**：任何功能进入开发前必须能回答——属于哪个 Domain？权威数据在哪里？谁可以操作？通过哪个 API？会产生什么 Event？影响哪些下游系统？回答不了就不允许进入开发。
