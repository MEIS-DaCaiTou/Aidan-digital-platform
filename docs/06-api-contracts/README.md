# API Contracts

本目录存放集团 Domain API 契约与接口规范。

原则：
- 业务动作 API 优先于数据库 CRUD。
- 外部系统通过 Adapter 隔离。
- Agent/MCP 不直接访问数据库或第三方通用 CRUD。
- Product / Order / Inventory 等核心资源统一使用集团 global ID。
