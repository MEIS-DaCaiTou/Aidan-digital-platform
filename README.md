# Aidan Digital Platform

服装集团数字业务平台（Aidan Digital Platform）。

本仓库用于沉淀集团数字化业务平台的业务架构、数据治理、Product Master、系统集成、事件契约、API 契约、PRD、数据模型、Agent/MCP、迁移与 PoC 文档，并逐步承载后续研发代码。

## 文档入口

正式文档统一进入：

- [docs/README.md](./docs/README.md)

## 当前治理基线

1. **AS-IS**：仅用于现状调查、迁移分析、历史留档和审计存档。
2. **TO-BE V1.0**：所有正式 PRD、数据模型、API、事件、权限、研发与验收的唯一基准。
3. 若 AS-IS 与 TO-BE 冲突，默认按迁移/治理问题处理，不反向污染目标架构。
4. 一个业务事实，只允许一个系统拥有最终写入权。
5. 系统可以复制数据，但不能复制数据所有权。

## 当前核心冻结文档

- 业务事实权威矩阵 TO-BE V1.0
- Product Master 字段级模型 V1.0
- 领猫 × 聚水潭 × Product Master 数据映射 V1.0

下一阶段将基于以上文档冻结 Product Domain Event Contract V1.0。
