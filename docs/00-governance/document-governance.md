# 文档治理规范

> 状态：V1.0 冻结  
> 适用范围：Aidan Digital Platform 全部业务、产品、数据、接口、事件、Agent/MCP 与研发文档。

## 1. 文档权威层级

1. **TO-BE 基线文档**：正式 PRD、数据模型、API、Event、权限、研发与验收的唯一设计依据。
2. **AS-IS 文档**：现状调查、迁移分析、历史留档和审计存档，不得作为新系统继续复制旧问题的理由。
3. **PRD / Contract / ADR**：必须引用上位 TO-BE 基线；若冲突，以已冻结的上位版本为准。
4. **聊天记录**：用于讨论和形成决策，但一旦内容冻结，必须沉淀进仓库文档；仓库版本优先于聊天记录。

## 2. 已冻结治理规则

- 一个业务事实，只允许一个系统拥有最终写入权。
- 系统可以复制数据，但不能复制数据所有权。
- Product Master 决定“这个商品是谁”。
- 领猫决定“商品如何研发、采购、生产出来”。
- 聚水潭/WMS 决定“货在物理世界里发生了什么”。
- OMS / IMS / Merchandising 决定集团标准订单、可售库存与货的下一步去向。
- Agent 永远不是 System of Record，只能通过受控 Domain API / MCP Tool 操作。

## 3. AS-IS 与 TO-BE 冲突处理

默认处理顺序：

```text
发现 AS-IS 与 TO-BE 不一致
        ↓
登记 Migration / Data Quality Issue
        ↓
制定映射、清洗、迁移或退役方案
        ↓
不得直接修改 TO-BE 以迁就历史系统
```

只有确认 TO-BE 本身不符合真实业务事实时，才允许走正式版本变更。

## 4. 版本规范

- 冻结版本使用 `V1.0 / V1.1 / V2.0`。
- 小版本：字段补充、澄清、兼容性增强，不改变核心权威边界。
- 大版本：事实所有权、主键、核心状态机、重大系统边界发生变化。
- 已冻结文档不得无版本覆盖关键架构决策。

## 5. PR 评审最低要求

涉及 Product / Order / Inventory / Integration / Agent 的 PR 必须回答：

- Fact Name
- Fact Owner
- Create Authority
- Update Authority
- Read Consumers
- Source Event
- Downstream Event
- Conflict Resolution
- Audit Requirement
- Agent Permission

无法回答的功能不得进入开发。
