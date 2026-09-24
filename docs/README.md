# Aidan Digital Platform 文档中心

本目录是项目正式文档的唯一归档入口。

## 文档结构

| 目录 | 内容 |
|---|---|
| `00-governance/` | 文档治理、ADR 与冻结规则 |
| `01-business-architecture/` | 业务架构、功能全景、自研产品架构 |
| `02-data-governance/` | AS-IS 调查存档与 TO-BE 业务事实权威矩阵 |
| `03-product-master/` | Product Master 字段级模型 |
| `04-integration/` | 领猫/JST 映射、Adapter 与迁移 |
| `05-event-contracts/` | Product / Order / Inventory Event Contract |
| `06-api-contracts/` | REST/GraphQL/Domain API 契约 |
| `07-prd/` | 正式产品 PRD |
| `08-agent-mcp/` | Agent、MCP、Policy 与 Approval |
| `09-data-model/` | ERD、Schema、Ledger 等数据模型 |
| `10-poc/` | PoC Gate、压测、迁移验证 |
| `11-technology/` | 开源选型与技术路线 |
| `99-archive/` | 已废弃/历史版本归档 |

## 当前冻结基线

1. [业务事实权威矩阵 TO-BE V1.0](./02-data-governance/to-be-business-fact-authority-matrix-v1.0.md)
2. [Product Master 字段级模型 V1.0](./03-product-master/product-master-field-model-v1.0.md)
3. [领猫 × 聚水潭 × Product Master 数据映射 V1.0](./04-integration/lingmao-jst-product-master-mapping-v1.0.md)
4. [AS-IS 调查存档 V1.0](./02-data-governance/as-is-data-business-authority-matrix-v1.0.md)

## 当前方案

- [飞书 BaseApp 业务运营平台蓝图 V1.0](./07-prd/feishu-baseapp-operations-platform-blueprint-v1.0.md)（方案已确认，尚未执行飞书结构或数据变更）
## 治理声明

- AS-IS 仅用于迁移调查与历史存档。
- 所有正式 PRD、数据模型、API、事件、权限、研发与验收均以 TO-BE V1.0 及其后续正式版本为唯一基准。
- 已冻结内容必须通过 Git PR 和版本变更更新，不再依赖聊天上下文作为唯一存档。
