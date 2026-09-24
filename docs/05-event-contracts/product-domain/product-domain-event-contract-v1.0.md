# Product Domain Event Contract V1.0

> 文档级别：核心集成契约
> 状态：V1.0 待 PR 冻结
> 领域：Product Master / Product Integration
> 上位依据：[业务事实权威矩阵 TO-BE V1.0](../../02-data-governance/to-be-business-fact-authority-matrix-v1.0.md)、[Product Master 字段级模型 V1.0](../../03-product-master/product-master-field-model-v1.0.md)、[领猫 × 聚水潭 × Product Master 数据映射 V1.0](../../04-integration/lingmao-jst-product-master-mapping-v1.0.md)
> 机器契约：[product-domain-event-v1.schema.json](./schemas/product-domain-event-v1.schema.json)

## 1. 目标与边界

本契约冻结商品领域事件的名称、生产者、触发条件、Envelope、Payload、顺序、幂等、重试、死信、重放和版本演进规则，供 Product Master、领猫 Adapter、聚水潭 Adapter、OMS、IMS、Channel、Merchandising、Data Platform 与 Agent/MCP 共同实现。

本契约只传播商品身份、分类、变体、生命周期、外部映射及投影同步结果，不传播 BOM、采购价、实际成本、库存、订单、会员、生产订单或平台消费者数据。外部系统 ID 只能出现在 `external_ref` 或 Mapping Payload 中，不能成为 `aggregate_id`。

## 2. 关键术语

| 术语 | 定义 |
|---|---|
| Aggregate | 事件对应的业务聚合，V1 为 SPU、SKU、BARCODE、MAPPING、PROJECTION 或 IDENTITY_REQUEST |
| Aggregate ID | 集团永久技术主键；SPU 使用 `spu_id`，SKU 使用 `sku_id` |
| Aggregate Version | 业务实体变更版本，用于识别乱序或缺失事件 |
| Event Version | 事件 Payload Schema 版本，与 Aggregate Version 无关 |
| Root Aggregate ID | 分区根标识；SPU/SKU/Barcode 使用 `spu_id`；Mapping 使用被映射实体所属 `spu_id`，无 SPU 归属的 Reference Master 使用 `global_id`；孤儿 Projection 使用自身 `projection_id` |
| Projection | Product Master 在领猫、聚水潭或其他下游的受控副本 |
| Producer | 对业务事实负责并发布事件的服务，不等于原始数据录入系统 |
| Consumer | 订阅事件并创建本地投影、同步任务或分析事实的服务 |

## 3. 事件传输与主题

V1 使用逻辑主题 `aidan.product.events.v1`。具体消息中间件可以在技术实现阶段选择，但不得改变本契约语义。

- 交付语义：At-least-once。
- 持久化：Producer 必须在同一数据库事务内写业务数据与 Outbox。
- 发布：Outbox Relay 负责发布，业务请求不得同步等待所有下游完成。
- 消费：Consumer 必须保存 Inbox/消费记录，并以 `event_id` 做唯一幂等键。
- 分区：`partition_key = root_aggregate_id`。商品族内事件进入同一分区。
- 顺序：只保证同一 `partition_key` 内的发布顺序；跨 SPU 不保证全局顺序。
- 保留：在线主题建议不少于 30 天；长期审计由对象存储或数据平台保留，具体期限由运维策略冻结。
- 死信主题：`aidan.product.events.v1.dlq`。

SKU 事件的 `aggregate_id` 仍为 `sku_id`，但 `root_aggregate_id` 与 `partition_key` 使用 `spu_id`。这样既保持正确的聚合身份，又能保证同一 SPU 下 SPU、SKU、Barcode 事件的有序处理。SPU/SKU Mapping 沿用同一商品族分区；Brand、Category、Color、Size 等无单一 SPU 归属的 Reference Master Mapping 使用自身 `global_id` 分区。

## 4. 标准 Envelope

所有事件必须使用以下 Envelope。业务 Payload 放入 `payload`，不得在顶层增加业务字段。

| 字段 | 类型 | 必填 | 规则 |
|---|---|---:|---|
| `spec_version` | string | 是 | 固定 `1.0` |
| `event_id` | UUIDv7 | 是 | 全局唯一；重试与重放保持不变 |
| `event_type` | string | 是 | 小写点分命名，如 `product.sku.created` |
| `event_version` | integer | 是 | Payload Schema 主版本，V1 为 `1` |
| `schema_ref` | string | 是 | `urn:aidan:event:<event_type>:v<event_version>` |
| `occurred_at` | RFC 3339 UTC | 是 | 业务事实发生时间 |
| `published_at` | RFC 3339 UTC | 是 | 消息发布或再次发布的时间 |
| `source_system` | enum | 是 | 见 4.1 |
| `producer` | string | 是 | 发布服务稳定名称 |
| `tenant_id` | UUIDv7 | 是 | 集团租户；不得用品牌代替 |
| `aggregate_type` | enum | 是 | SPU/SKU/BARCODE/MAPPING/PROJECTION/IDENTITY_REQUEST |
| `aggregate_id` | UUIDv7 | 是 | 对应聚合主键，禁止业务编码或第三方 ID |
| `aggregate_version` | integer | 是 | 变更后的实体版本，从 1 单调递增 |
| `root_aggregate_id` | UUIDv7 | 是 | 商品事件为 `spu_id`；申请事件为 `request_id` |
| `partition_key` | UUIDv7 | 是 | 必须等于 `root_aggregate_id` |
| `trace_id` | string | 是 | 跨服务链路追踪标识 |
| `correlation_id` | string | 是 | 同一业务流程标识 |
| `causation_id` | UUIDv7/null | 是 | 触发本事件的上游 event_id；无则 null |
| `actor` | object | 是 | actor_type、actor_id、source_system |
| `data_classification` | enum | 是 | V1 固定 `INTERNAL` |
| `payload` | object | 是 | 与 event_type/event_version 匹配 |

### 4.1 `source_system`

沿用 Product Master 的正式枚举：`PRODUCT_MASTER`、`LINGMAO`、`JST`、`POS`、`ERP`、`DAM`、`CHANNEL_DOUYIN`、`CHANNEL_TMALL`、`CHANNEL_JD`、`CHANNEL_PDD`、`CHANNEL_XHS`、`CHANNEL_WECHAT`，并补充事件基础设施使用的 `INTEGRATION_PLATFORM`、`CHANNEL_PLATFORM`、`SYSTEM`。V1 的 Product Master 事实事件只能由 `PRODUCT_MASTER` 发布；Adapter 只能发布自身投影结果事件。

### 4.2 `actor`

```json
{
  "actor_type": "USER",
  "actor_id": "user-2481",
  "source_system": "PRODUCT_MASTER"
}
```

`actor_type` 允许 `USER`、`SERVICE`、`AGENT`、`SYSTEM`。Agent 发起的写动作必须记录真实 Agent ID，且已在 Domain API 层通过 Policy/Approval；消费者不得把事件本身视为越权写入许可。

## 5. 事件命名与语义

命名格式为 `product.<entity>.<past_tense>`。事件表达已经发生的业务事实，不使用 `create_product`、`sync_now` 等命令式名称。

- `created`：新身份已经持久化。
- `updated`：可变字段已经更新，不用于生命周期状态变化。
- `approved`：审核通过且身份冻结规则开始生效。
- `status_changed`：受控状态机转换已经完成。
- `assigned`：关系已经建立，如 Barcode 与 SKU 绑定。
- `synced`：目标系统业务响应已确认成功，不等于 HTTP 200。
- `sync_failed`：本轮目标投影同步已终止或进入 DLQ。
- `drift_detected`：Reconciliation 发现权威数据与外部投影不一致。

V1 不定义 `product.*.deleted`。有历史引用的商品只能通过状态事件停用、终止或归档。

## 6. 事件目录

| Event Type | Producer | Trigger | Aggregate | 主要消费者 |
|---|---|---|---|---|
| `product.identity.requested` | Integration Platform | 合法身份申请已接收并持久化 | IDENTITY_REQUEST | Product Master |
| `product.identity.request_rejected` | Product Master | 身份申请因校验或冲突被拒绝 | IDENTITY_REQUEST | Lingmao Adapter、Data Platform |
| `product.spu.created` | Product Master | DRAFT SPU 已创建 | SPU | Lingmao Adapter、Data Platform |
| `product.spu.updated` | Product Master | SPU 可变字段已更新 | SPU | Lingmao/JST Adapter、OMS、IMS、Channel、Data Platform |
| `product.spu.approved` | Product Master | SPU 已通过审核并冻结身份字段 | SPU | Lingmao/JST Adapter、OMS、IMS、Channel、Data Platform |
| `product.spu.status_changed` | Product Master | SPU 生命周期/可售/研发状态已合法转换 | SPU | 全部商品投影消费者 |
| `product.sku.created` | Product Master | SKU 身份已创建 | SKU | Lingmao/JST Adapter、OMS、IMS、Channel、Data Platform |
| `product.sku.updated` | Product Master | SKU 可变字段已更新 | SKU | Lingmao/JST Adapter、OMS、IMS、Channel、Data Platform |
| `product.sku.status_changed` | Product Master | SKU 状态已合法转换 | SKU | JST Adapter、OMS、IMS、Channel、Data Platform |
| `product.barcode.assigned` | Product Master | Barcode 已校验并绑定 SKU | BARCODE | Lingmao/JST Adapter、OMS、IMS、Channel |
| `product.barcode.changed` | Product Master | 主条码或条码状态已受控变更 | BARCODE | Lingmao/JST Adapter、OMS、IMS、Channel |
| `product.mapping.created` | Product Master | 外部 ID 与集团 ID 映射建立 | MAPPING | Resolver、对应 Adapter、Data Platform |
| `product.mapping.updated` | Product Master | 映射状态、主映射或有效期变更 | MAPPING | Resolver、对应 Adapter、Data Platform |
| `product.projection.synced` | 对应 Adapter | Vendor 业务响应确认投影成功 | PROJECTION | Product Master、Integration Monitor、Data Platform |
| `product.projection.sync_failed` | 对应 Adapter | 重试耗尽或不可重试错误 | PROJECTION | Integration Monitor、Data Platform |
| `product.projection.drift_detected` | 对应 Adapter | 对账发现投影缺失、孤儿或字段偏移 | PROJECTION | Product Master、Data Quality、Integration Monitor |

## 7. Payload 通用类型

### 7.1 Product Reference

所有事件中的 `spu_id`、`sku_id` 为 UUIDv7；`spu_code`、`sku_code`、`style_no`、`barcode` 只用于业务识别和下游投影。

### 7.2 Change Set

`updated` 与 `status_changed` 事件必须提供：

```json
{
  "changed_fields": ["product_name", "wave_id"],
  "change_reason": "商品资料复核",
  "previous": {
    "product_name": "旧名称"
  },
  "current": {
    "product_name": "新名称"
  }
}
```

`previous/current` 只包含本次改变且允许进入事件的字段。完整前后快照保存在 `product_change_log`，不广播整个数据库记录。

### 7.3 External Reference

```json
{
  "source_system": "JST",
  "entity_type": "SKU",
  "source_id": "JST-881029",
  "source_code": "AW27JK001-BLK-M"
}
```

## 8. 身份申请事件

### 8.1 `product.identity.requested`

表示 Integration Platform 已接收一笔合法格式的商品身份申请。它不表示 SPU/SKU 已创建，也不授予领猫直接写 Product Master 的权限。

Payload 必填：

```json
{
  "request_id": "019d0000-0000-7000-8000-000000000001",
  "request_source": "LINGMAO",
  "source_ref": {
    "source_system": "LINGMAO",
    "entity_type": "SPU",
    "source_id": "LM_STYLE_8831",
    "source_code": "AW27JK001"
  },
  "style_no": "AW27JK001",
  "product_name": "女式短款羽绒服",
  "brand_code": "BR-A",
  "category_code": "APPAREL/WOMEN/OUTERWEAR/DOWN",
  "year_code": "2027",
  "season_code": "AUTUMN_WINTER",
  "wave_code": "AW27-W01",
  "gender_code": "WOMEN",
  "variants": [
    {
      "source_sku_id": "LM_SKU_9911",
      "color_code": "BLK01",
      "size_code": "M",
      "barcode": "6901234567892"
    }
  ]
}
```

### 8.2 `product.identity.request_rejected`

Payload：`request_id`、`request_source`、`source_ref`、`reason_code`、`reason_message`、`quality_issue_id`（可空）、`retryable`。禁止在拒绝事件中回传内部堆栈、Token 或 Vendor 密钥。

## 9. SPU 事件

### 9.1 SPU Snapshot

SPU 事件使用最小稳定 Snapshot：

```json
{
  "spu_id": "019d0000-0000-7000-8000-000000000010",
  "spu_code": "SPU-2027-00001823",
  "style_no": "AW27JK001",
  "product_name": "女式短款羽绒服",
  "brand_id": "019d0000-0000-7000-8000-000000000020",
  "category_id": "019d0000-0000-7000-8000-000000000021",
  "year_id": "019d0000-0000-7000-8000-000000000022",
  "season_id": "019d0000-0000-7000-8000-000000000023",
  "wave_id": "019d0000-0000-7000-8000-000000000024",
  "gender_code": "WOMEN",
  "product_type": "APPAREL",
  "variant_schema": "COLOR_SIZE",
  "lifecycle_status": "APPROVED",
  "sellable_status": "NOT_SELLABLE",
  "development_status": "CONFIRMED",
  "version": 7
}
```

- `product.spu.created`：Payload 为 `spu` + `source_ref`（可空）。
- `product.spu.updated`：Payload 为 `spu` + Change Set。
- `product.spu.approved`：Payload 为 `spu` + `approved_at` + `approved_by`；此时 `lifecycle_status` 必须为 `APPROVED`。
- `product.spu.status_changed`：Payload 为 `spu_id`、`spu_code`、`status_dimension`、`from_status`、`to_status`、`effective_at`、`change_reason`、`version`。`status_dimension` 只允许 `LIFECYCLE`、`SELLABLE`、`DEVELOPMENT`。

## 10. SKU 事件

### 10.1 SKU Snapshot

```json
{
  "sku_id": "019d0000-0000-7000-8000-000000000030",
  "sku_code": "SKU-2027-00018231",
  "spu_id": "019d0000-0000-7000-8000-000000000010",
  "spu_code": "SPU-2027-00001823",
  "style_no": "AW27JK001",
  "brand_id": "019d0000-0000-7000-8000-000000000020",
  "category_id": "019d0000-0000-7000-8000-000000000021",
  "year_id": "019d0000-0000-7000-8000-000000000022",
  "season_id": "019d0000-0000-7000-8000-000000000023",
  "wave_id": "019d0000-0000-7000-8000-000000000024",
  "color_id": "019d0000-0000-7000-8000-000000000040",
  "color_code": "BLK01",
  "size_id": "019d0000-0000-7000-8000-000000000041",
  "size_code": "M",
  "primary_barcode": "6901234567892",
  "sku_status": "ACTIVE",
  "version": 3
}
```

- `product.sku.created`：Payload 为 `sku`。SKU 可以在 SPU APPROVED 前创建为 DRAFT；Adapter 只有在下游创建条件满足后才能生成正式投影。
- `product.sku.updated`：Payload 为 `sku` + Change Set；APPROVED 后不得通过该事件修改 `spu_id/color_id/size_id/variant_key`。
- `product.sku.status_changed`：Payload 为 `sku_id`、`sku_code`、`spu_id`、`from_status`、`to_status`、`effective_at`、`change_reason`、`version`。

## 11. Barcode 事件

- `product.barcode.assigned`：Payload 为 `barcode_id`、`sku_id`、`sku_code`、`spu_id`、`barcode`、`barcode_type`、`is_primary`、`status`、`valid_from`、`version`。
- `product.barcode.changed`：Payload 为上述当前值 + `change_type`、`previous_barcode`（可空）、`previous_status`（可空）、`change_reason`。`change_type` 允许 `PRIMARY_CHANGED`、`STATUS_CHANGED`、`VALUE_CORRECTED`。

条码变更不允许消费者重新识别 SKU 身份。消费者必须继续以 `sku_id` 关联历史订单、库存和投影。

## 12. Mapping 事件

### 12.1 `product.mapping.created`

Payload 必填：

```json
{
  "mapping_id": "019d0000-0000-7000-8000-000000000050",
  "entity_type": "SKU",
  "global_id": "019d0000-0000-7000-8000-000000000030",
  "source_system": "JST",
  "source_entity_type": "SKU",
  "source_id": "JST-881029",
  "source_code": "AW27JK001-BLK-M",
  "mapping_status": "ACTIVE",
  "is_primary": true,
  "valid_from": "2026-09-24T06:00:00Z"
}
```

### 12.2 `product.mapping.updated`

Payload 为 `mapping` 当前值 + Change Set。允许变化：`source_code`、`mapping_status`、`is_primary`、`valid_to`、受控 `metadata`；禁止修改既有 Mapping 的 `source_system/entity_type/source_id/global_id`。关系错误时必须关闭旧 Mapping 并创建新 Mapping，保留审计链。

## 13. Projection 结果事件

Projection 事件只能由相应 Adapter 发布。`target_system=JST` 时 Producer 必须是 JST Adapter；`target_system=LINGMAO` 时必须是 Lingmao Adapter。

### 13.1 `product.projection.synced`

Payload：`sync_id`、`target_system`、`entity_type`、`global_id`、`operation`、`source_event_id`、`source_aggregate_version`、`external_ref`、`vendor_response_code`、`synced_at`。

只有 Vendor 业务返回确认成功后才能发布。HTTP 200、请求进入队列或 Adapter 已接收均不等于 `synced`。

### 13.2 `product.projection.sync_failed`

Payload：`sync_id`、`target_system`、`entity_type`、`global_id`、`operation`、`source_event_id`、`attempt`、`retryable`、`error_code`、`error_message`、`failed_at`、`next_action`。`next_action` 允许 `RETRY_SCHEDULED`、`DLQ`、`MANUAL_REVIEW`、`BLOCKED_BY_MAPPING`。

### 13.3 `product.projection.drift_detected`

Payload：`reconciliation_id`、`target_system`、`entity_type`、`global_id`（孤儿外部商品时可空）、`external_ref`、`drift_type`、`field_differences`、`detected_at`、`recommended_action`、`quality_issue_id`。

`drift_type` 允许 `MISSING_PROJECTION`、`VALUE_MISMATCH`、`ORPHAN_EXTERNAL_PRODUCT`、`IMMUTABLE_FIELD_CHANGED`、`STATUS_MISMATCH`。发现 Drift 只登记事实和建议，不允许 Adapter 直接覆盖 Product Master。

## 14. Producer 权限

| Producer | 允许发布 |
|---|---|
| Product Master | SPU、SKU、Barcode、Mapping 事实事件；身份申请拒绝事件 |
| Integration Platform | `product.identity.requested` |
| Lingmao Adapter | 目标为 LINGMAO 的 Projection 结果事件 |
| JST Adapter | 目标为 JST 的 Projection 结果事件 |
| 其他业务系统 | 无权发布 Product Domain V1 事件 |

消息基础设施必须按 Producer Identity 配置 Topic ACL。仅依赖应用代码约定不构成权限控制。

## 15. Consumer 行为矩阵

| Consumer | 订阅范围 | 必须行为 | 禁止行为 |
|---|---|---|---|
| Lingmao Adapter | approved/created/updated/status/barcode/mapping | 创建或更新研发投影、回写集团 ID | 创建 global SKU、修改 PM DB |
| JST Adapter | approved/sku/status/barcode/mapping | 创建或更新 JST 普通商品投影 | 将 JST 人工值反写 PM |
| OMS Projection | SPU/SKU/status/barcode | 维护只读商品投影和订单快照引用 | 本地修改商品身份 |
| IMS Projection | SKU/status/barcode | 维护库存维度，只用 sku_id 作主键 | 用条码或 JST sku_id 作库存主键 |
| Channel Platform | approved/SKU/status/barcode | 建立渠道商品候选和只读商品投影 | 把平台 Listing ID 写入 PM |
| Merchandising | SPU/SKU/status | 维护商品维度用于配补调分析 | 改商品主数据 |
| Data Platform | 全量 | 保存不可变事件和构建维度历史 | 反向成为商品权威源 |
| Agent/MCP | 经 Product API 或授权投影读取 | 使用 Resolver 获取 global ID | 直接消费事件实施未审批写操作 |

## 16. 幂等、乱序与并发

1. Consumer Inbox 对 `(consumer_name, event_id)` 建唯一约束。
2. 同一事件重复到达时返回成功，不重复创建商品、映射或同步任务。
3. Consumer 保存每个 Aggregate 的 `last_aggregate_version`。
4. 收到 `aggregate_version <= last_aggregate_version` 时按重复或旧事件处理，不覆盖新数据。
5. 收到 `aggregate_version > last_aggregate_version + 1` 时标记 `VERSION_GAP`，暂停该 Aggregate 的破坏性动作，并通过 Product API 补快照或触发 Replay。
6. Adapter 的外部同步幂等键为 `<event_id>:<target_system>`；Vendor 已成功但响应丢失时，先查询或对账，禁止直接重复创建。
7. Product Master 写 API 使用 `expected_version` 乐观锁；Event 不绕过该规则。

## 17. 重试与 DLQ

### 17.1 Consumer 技术失败

默认退避：第 1、5、30 秒，第 5、30 分钟，共 5 次。网络超时、Broker 暂时不可用、Vendor 限流可重试；Schema 不兼容、权限拒绝、必填映射缺失不可盲目重试。

### 17.2 Adapter 业务失败

Adapter 消费商品事件后，应先持久化 `sync_task` 再 ACK。Vendor 同步由任务 Worker 执行，失败结果通过 Projection 结果事件反馈。业务重试次数与间隔由 Adapter 策略管理，但必须保留相同 `source_event_id` 和幂等键。

### 17.3 DLQ 记录

DLQ 必须保留原始 Event、Consumer、失败阶段、错误分类、最后错误、attempt、first_failed_at、last_failed_at、可否重放。修复后由授权人员或自动化任务重放，不能通过手工复制 Payload 生成伪造事件。

## 18. Replay

- Replay 保留原 `event_id`、`event_type`、`event_version`、`occurred_at` 与 Payload。
- `published_at` 更新为重放时间，并通过传输 Header 增加 `replay_id`、`replayed_at`、`replay_reason`。
- Consumer 只有在明确选择 Replay 模式时才可清除指定消费记录；默认幂等机制会忽略已成功处理的事件。
- 大范围 Replay 必须限定 event_type、时间区间、Aggregate 范围与目标 Consumer，并先在隔离环境验证。
- Replay 不得触发通知、重复创建 Vendor 商品或重做已完成的人工审批。

## 19. Schema 演进

### 19.1 兼容变更

允许：新增可选字段、放宽非身份字段长度、增加消费者可忽略的 Metadata。Producer 必须先确认消费者忽略未知字段。

### 19.2 破坏性变更

以下必须提升 `event_version`，并在迁移期双发或提供转换层：删除/重命名必填字段、改变字段类型或语义、改变 Aggregate 规则、把可空改为必填、改变枚举含义。

新增枚举值也可能破坏严格消费者。消费者必须实现 `UNKNOWN` 处理或将未知值送入人工/技术异常队列，不得静默映射成现有值。

V1 事件 Schema 的 `$id` 和 `schema_ref` 一旦冻结不可复用。修订文档描述但不改 Schema 语义时只更新文档小版本；Schema 变更必须更新机器契约。

## 20. 可观测性与审计

最少指标：Outbox 未发布数量与最老等待时间、Producer 发布失败率、Consumer Lag、消费成功/失败/重复数、VERSION_GAP 数、Adapter Sync Success/Failure/Latency、DLQ 数量、Drift 类型分布、Replay 数量。

日志必须包含 `event_id`、`trace_id`、`correlation_id`、`aggregate_id`、`aggregate_version`、`consumer`、`attempt`，不得记录 Token、签名、完整 Vendor 凭证或含个人信息的原始响应。

## 21. 安全与数据分级

- V1 商品事件统一为 `INTERNAL`。
- Topic 使用传输加密和服务身份认证。
- Producer/Consumer 遵循最小权限和 ACL 白名单。
- Payload 禁止包含会员、地址、手机号、身份证、支付信息、平台消费者数据、Vendor 密钥。
- Agent 只能通过已授权 Domain API/MCP Tool 触发写入；事件总线不是 Agent 写入口。

## 22. 验收标准

V1 至少通过以下场景：

1. Product Master 创建 1 SPU + 30 SKU，所有事件的 aggregate/root/partition 规则正确。
2. 同一 `product.sku.created` 重复投递 10 次，JST 仅产生一个商品 SKU 与一个有效 Mapping。
3. 先收到 Aggregate Version 5、后收到 Version 4，消费者不回滚本地投影。
4. 缺少 Version 4 而直接收到 Version 5，消费者标记 VERSION_GAP 并能补快照。
5. Vendor 已创建成功但响应超时，重试与对账不产生第二个外部商品。
6. JST 人工修改身份字段，对账发布 `drift_detected`，不会覆盖 Product Master。
7. Barcode 冲突阻断在 Product Master 内，不能发布一个误导下游的 assigned 事件。
8. 不可重试的 Mapping 缺失进入 MANUAL_REVIEW 或 DLQ，错误可定位到 source event。
9. Replay 保留原 event_id，已成功 Consumer 不重复执行，指定 Consumer 可受控重放。
10. V1 Consumer 能忽略新增可选字段；不支持的主版本明确拒绝并告警。
11. Agent 发起的受控变更可追踪 actor_type=AGENT、actor_id、审批记录与 trace_id。
12. 任意事件均不包含库存、订单、成本、会员或 Vendor 密钥。

## 23. V1 冻结决策

1. Product Master 是 SPU/SKU/Color/Size/Barcode 与 Mapping 事实事件的唯一生产者。
2. `aggregate_id` 永远使用集团技术主键；第三方 ID 不得作为 Aggregate。
3. 同一商品族按 `spu_id` 分区，SKU 事件仍保留自身 `sku_id` Aggregate。
4. 事件交付采用 At-least-once；Outbox、Inbox 和消费者幂等是必选能力。
5. Adapter 同步是异步流程；Vendor 业务确认后才发布 `projection.synced`。
6. Drift 只产生治理事实，不自动改变 Product Master。
7. Event 只携带下游正确处理所需的稳定最小数据；完整详情通过 Product API 查询。
8. 删除不进入 V1 商品事件；停用、终止和归档通过状态事件表达。
9. Replay 保留原事件身份，必须可审计、可限定范围、可避免外部副作用。
10. 破坏性 Schema 变更必须升主版本并提供迁移期。
