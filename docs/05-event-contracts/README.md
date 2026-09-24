# Event Contracts

本目录存放领域事件契约。

## 当前契约

- [Product Domain Event Contract V1.0](./product-domain/product-domain-event-contract-v1.0.md)（PR 审核中）
- [Product Domain Event JSON Schema V1](./product-domain/schemas/product-domain-event-v1.schema.json)

## 后续计划

- Order Domain Event Contract
- Inventory Domain Event Contract

所有事件必须定义：Producer、Consumer、Trigger、Schema、aggregate_id、Partition Key、Ordering、Idempotency、Retry、DLQ、Replay、Schema Evolution。
