# 开源底座调研与选型结论 V1.0

> 目的：调研可二次开发改造的成熟开源项目，用于搭建服装集团业务系统，降低开发成本；并考虑系统需要可供 agent 调用的接口设计。
> 结论日期：2026-09
> 核心结论：采用 **Composable Architecture**——成熟开源项目负责通用能力，服装行业核心能力保持自研，通过统一 API/Event/MCP 拼成集团业务平台。

## 一、推荐开源组合（V1 基线）

| 领域 | 建议底座 | 建设策略 |
|---|---|---|
| ERP/财务/采购/基础生产 | ERPNext + Frappe | 直接复用 + 接口集成 |
| 集团组织/基础工作流/部分后台 | Frappe Framework | 二次开发 |
| 商品主数据 Product Master | 自研 | 集团核心 |
| PIM | Akeneo CE 或自研 Frappe PIM（PoC 后定） | 二选一 |
| DAM | ResourceSpace | 高度建议直接复用 |
| 渠道商品发布 | 自研 | 抖音/天猫/京东/PDD Adapter |
| Commerce Core | Saleor | 强烈值得 PoC（后文已降级为可选前台组件） |
| OMS | Saleor + 自研编排层（后文已修订为自研 OMS） | 不建议完全重写 |
| IMS | Saleor 库存能力 + 独立 IMS 扩展（后文已修订为自研 IMS） | 核心库存逻辑自研 |
| 配货/补货/调拨 | 自研 | 企业核心竞争能力 |
| IAM/SSO | Keycloak | 直接复用 |
| BI | Apache Superset | 直接复用 |
| 数据转换 | dbt Core | 直接复用 |
| 分析型数据仓库 | ClickHouse | 推荐 |
| Agent 接口 | 自研 MCP Gateway | OpenAPI + MCP |
| AI Agent | 自研 | 只调用业务 API/MCP |

## 二、ERPNext / Frappe

- ERPNext GPLv3，底层 Frappe Framework MIT；官方支持开发商业应用与集成应用；v16 活跃维护中，规划支持到 2029 年底。
- 原生支持 Item Template → Color/Size → Item Variant（官方示例即 T-shirt 颜色/尺码），包含 BOM、Work Order、Production Planning、Quality、Inventory 等制造能力。
- 接口友好：DocType 可通过 REST API 访问，最新提供 /api/v2/document/...；自定义 Python 方法显式 whitelist 即可暴露为 API；有 Hooks、Document Events、Webhook、后台任务。
- 建议定位：**ERPNext 负责财务/采购/应收/应付/基础制造，不深改 Core**；集团业务平台（商品/PIM/OMS/IMS/配补调/Channel）通过 REST/Event 对接。

## 三、Odoo Community（第二候选）

- Odoo 19 CE（LGPLv3）原生支持颜色/尺码 Product Variant、单 SKU 独立条码/库存/价格、Variant BOM。
- 风险点：Odoo 19 官方 JSON-2 External API 文档注明官方外部 API 涉及 Custom pricing plan；XML-RPC/JSON-RPC 计划在 Odoo 22 移除。选型前必须验证自托管 Community、API 使用方式和许可边界。
- 排序：ERPNext/Frappe 第一候选，Odoo CE 第二候选；团队如有大量 Odoo/Python 经验可互换。

## 四、Saleor（本次调研最值得关注）

- Python + Django + GraphQL + Headless Commerce，核心 BSD-3-Clause，代码活跃。原生 API-only 架构，支持商品、Variant、Channel、多仓、订单、退货、促销、客户、多币种、多语言；官方强调多渠道架构及 Webhook/App/GraphQL 扩展。
- **定位修正（后续拍板）**：Saleor ≠ 集团 OMS/Commerce Core，仅作为可选 Storefront Backend（品牌官网/独立站/小程序商城/海外 D2C 时才评估接入），同为 IMS 单向下游。
- saleor-mcp（官方 MCP Server，只读，通过 GraphQL）是集团 Agent 架构的参考实现；但仓库许可证 AGPL-3.0，集团 fashion-group-mcp 建议从零实现。

## 五、Medusa / Vendure（备选）

- **Medusa**：核心 Commerce Modules MIT，Order Module 支持订单/Draft Order/Return/Exchange/Claim，Inventory Module 支持多地点与 Reservation，有 OpenAPI Admin API。TypeScript/Node 技术栈友好。现为 open-core，Enterprise Materials 需商业协议。排序：Saleor > Medusa > Vendure。
- **Vendure**：TypeScript + GraphQL Commerce Framework，Admin/Shop API 完整，Schema 适合 Agent 工具生成；但默认 GPLv3 + 商业双授权，深改 Core 需评估 GPL 或购买商业许可。技术备选。

## 六、PIM：Akeneo CE（PoC 候选）

- Akeneo CE（OSL 3.0）覆盖 Family/Category/Attribute/Variant/Color/Size/Channel/Locale/Completeness/Product Model。
- 风险：Calendar Versioning 不保证向后兼容，二开过深升级成本高；完整 Asset Manager 是商业版能力。
- 建议：Akeneo = PIM，ResourceSpace = DAM，不让 Akeneo 同时承担 PIM+DAM。OSL 3.0 的衍生作品/部署义务需法务审查。

## 七、DAM：ResourceSpace（直接减少开发量）

- 专业 DAM：图片/视频/设计文件、Metadata、Tag、Collection、搜索、权限、版本；免费开源、自托管、支持 Docker；BSD-style 许可证；完整 RESTful JSON API（创建资源、上传/替换文件、修改 Metadata、搜索、Collection）。
- 二开方式：Fashion DAM Adapter 建立 ResourceSpace Resource ID ↔ SPU/SKU/Color 关联；dam_asset 系列表大部分不用重新实现。

## 八、Pimcore / Directus（不建议作为免费底座）

- **Pimcore**：PIM/MDM/DAM/Datahub/GraphQL 很强，但当前采用 Pimcore Open Core License，商业组织免费 Community 适用条件含年营收 < 500 万美元/欧元等限制。可研究产品设计，不建议默认选为低成本底座。
- **Directus**：采用 Monospace Sustainable Core License，对组织规模/收入/高级功能实施许可控制，Directus 12 引入许可证 enforcement。不作为集团核心底座。

## 九、数据平台（直接用成熟开源）

| 能力 | 项目 | 许可证 |
|---|---|---|
| Analytics DB | ClickHouse | Apache 2.0 |
| Data Transformation | dbt Core | Apache 2.0 |
| BI/Dashboard | Apache Superset | Apache 2.0 |
| IAM | Keycloak | Apache 2.0 |
| Agent Protocol | MCP | — |

## 十、自研 vs 开源重划分（调研后）

| 系统 | 原计划 | 调研后建议 |
|---|---|---|
| ERP | 采购 | ERPNext/Odoo PoC |
| 商品中心 | 自研 | 继续自研 |
| PIM | 自研 | Akeneo PoC / 自研二选一 |
| DAM | 自研 | ResourceSpace 二开 |
| Channel | 自研 | 继续自研 |
| OMS | 自研 | Saleor Core + 自研（后续修订为自研） |
| IMS | 自研 | 复用库存基础模型 + 核心自研 |
| 配货/补货/调拨 | 自研 | 继续自研 |
| IAM | 自研/平台 | Keycloak |
| BI | 自研 | Superset |
| 数仓分析 | 自研 | ClickHouse + dbt |
| Agent Gateway | 新增 | 自研 MCP Gateway |

真正需要从零开发的核心收缩为：**Product Master + Channel Publishing + OMS Orchestration + Group IMS + Merchandising + Agent Business Tools**。

## 十一、Agent 接口设计原则

- 架构：Human/Frontend → REST/GraphQL API → Domain Service；AI Agent → MCP Gateway → Domain API。**禁止 Agent 直接查 SQL / ERP arbitrary RPC。**
- Agent API 分三类：
  - **Read**（product.search/get、inventory.get_availability、order.get/search、sales.get_metrics）：Agent 自动调用。
  - **Recommend/Draft**（replenishment.calculate、allocation.calculate、transfer.create_draft、channel_product.generate_draft）：Agent 可产生结果，不自动执行关键业务动作。
  - **Execute**（transfer.approve、channel_product.publish、inventory.adjust、order.cancel、refund.execute）：必须 Agent → Policy → Workflow → Human Approval/Rule Approval → Execute。
- Agent 写接口公共字段：actor_type、actor_id、agent_id、conversation_id、trace_id、idempotency_key、reason、dry_run、approval_id。
- Agent Tool 必须是**业务动作 API**（reserve_inventory、create_transfer_draft、publish_channel_product），而不是数据库 CRUD（update_inventory_balance、delete_order）。
- MCP：TypeScript SDK v2 已进入稳定线（2026-07-28 MCP Spec），Server 可标准化暴露 tools/resources/prompts，支持远程 HTTP 传输。

## 十二、正式编写研发文档前的 PoC Gate（6 个业务场景）

1. 一款服装 5 色 × 6 码 = 30 SKU 从商品中心流入 ERP/PIM/Saleor；
2. DAM 一次导入 500–1000 张图片按款号/颜色自动关联；
3. 一个集团商品生成 20 个抖店裂变商品，单张/批量替换主图并生成发布任务；
4. 抖音订单进入 OMS → IMS 预占 → 模拟 WMS 发货 → OMS 完成；
5. 100 家门店 × 数千 SKU 的库存查询、ATP、补货/调拨计算；
6. Agent 执行“查某款库存 → 找缺码门店 → 生成调拨草稿”，但不能未经审批直接发出调拨单。
