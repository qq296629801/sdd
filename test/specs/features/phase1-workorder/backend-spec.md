# 工单信息管理 IIDP 后端规格

## 1. 命名

| 项 | 值 | 说明 |
|---|---|---|
| appName | `demo-mes` | 小写中划线 |
| appPkg | `mes` | Java 包段，全小写 |
| moduleName | `workorder` | 业务模块目录 |
| model_name | `mes_work_order` | 小写下划线，全局唯一 |
| Java 类名 | `MesWorkOrder` | UpperCamelCase |
| 菜单 key | `mes_work_order_menu` | 带业务前缀 |

## 2. 工程文件

| 文件 | 新增/修改 | 说明 |
|---|---|---|
| `sie-iidp-demo-apps/pom.xml` | 修改 | 登记 `sie-iidp-demo-mes` 子模块 |
| `sie-iidp-demo-apps/sie-iidp-demo-mes/pom.xml` | 新增 | 工单模块 POM，依赖 `sie-iidp-demo-common` |
| `src/main/java/com/sie/iidp/mes/app.json` | 新增 | App 描述，登记视图和菜单路径 |
| `model/MesWorkOrder.java` | 新增 | 工单元模型 |
| `views/mes_work_order_view.json` | 新增 | grid / search / form 三视图 |
| `data/menus.json` | 新增 | 工单管理菜单入口 |
| `data/mes_work_order_status.json` | 新增 | 工单状态字典种子数据 |
| `data/mes_work_order_priority.json` | 新增 | 优先级字典种子数据 |
| `apps/apps.json` | 修改 | 登记 `sie-iidp-demo-mes.jar` |

## 3. 模型设计

| 字段 | 类型 | IIDP 注解 | 必填 | 校验 | 索引 | 说明 |
|---|---|---|---|---|---|---|
| `workOrderNo` | `String` | `@Property(displayName="工单号")` | 系统生成 | — | 唯一索引 | 格式 WO+YYYYMMDD+4位流水，create 服务内自动生成 |
| `productId` | `Long` | `@Property(displayName="产品ID")` | 是 | `@NotNull` | 普通索引 | 冗余产品信息，跨 App 弱引用 |
| `productCode` | `String` | `@Property(displayName="产品编码")` | 是 | `@NotBlank` | 否 | 创建时从产品表冗余 |
| `productName` | `String` | `@Property(displayName="产品名称")` | 是 | `@NotBlank` | 否 | 创建时从产品表冗余 |
| `productSpec` | `String` | `@Property(displayName="产品规格")` | 否 | — | 否 | 可选冗余 |
| `uom` | `String` | `@Property(displayName="计量单位")` | 是 | `@NotBlank` | 否 | 创建时从产品表冗余 |
| `plannedQuantity` | `BigDecimal` | `@Property(displayName="计划数量")` | 是 | `@NotNull @DecimalMin("0.0001")` | 否 | 精度(18,4)，必须大于0 |
| `goodQuantity` | `BigDecimal` | `@Property(displayName="良品数量")` | 否 | — | 否 | 默认0，执行中状态可录入 |
| `scrapQuantity` | `BigDecimal` | `@Property(displayName="废品数量")` | 否 | — | 否 | 默认0，执行中状态可录入 |
| `status` | `String` | `@Property(displayName="状态") @Selection(name="mes_work_order_status")` | 是 | — | 是（查询过滤） | 枚举见字典 |
| `priority` | `String` | `@Property(displayName="优先级") @Selection(name="mes_work_order_priority")` | 否 | — | 否 | HIGH/NORMAL/LOW，默认NORMAL |
| `plannedStartDate` | `Date` | `@Property(displayName="计划开始日期")` | 否 | — | 复合索引 | 与 status 组合索引 |
| `plannedEndDate` | `Date` | `@Property(displayName="计划结束日期")` | 否 | — | 否 | 须 ≥ plannedStartDate |
| `actualStartTime` | `Date` | `@Property(displayName="实际开始时间")` | 否 | — | 否 | startProgress 时系统写入 |
| `actualEndTime` | `Date` | `@Property(displayName="实际结束时间")` | 否 | — | 否 | complete 时系统写入 |
| `workCenterId` | `Long` | `@Property(displayName="工作中心ID")` | 否 | — | 普通索引 | 跨 App 弱引用 |
| `workCenterName` | `String` | `@Property(displayName="工作中心名称")` | 否 | — | 否 | 创建/编辑时冗余 |
| `bomVersion` | `String` | `@Property(displayName="BOM版本")` | 否 | — | 否 | |
| `routingVersion` | `String` | `@Property(displayName="工艺路线版本")` | 否 | — | 否 | |
| `remarks` | `String` | `@Property(displayName="备注")` | 否 | `@Size(max=500)` | 否 | |
| `deleted` | `Boolean` | `@Property @Deleted` | 系统 | — | 是 | 软删除标记 |

模型规则：
- 类注解：`@StaticVar @Getter @Setter @Model(name="mes_work_order", displayName="生产工单")`
- 继承 `BaseModel<MesWorkOrder>`
- `status` 绑定字典 `mes_work_order_status`，`priority` 绑定字典 `mes_work_order_priority`
- 索引：`workOrderNo` 唯一索引；`productId`、`workCenterId`、`status`+`plannedStartDate` 复合索引

## 4. 服务设计

| 服务 | 类型 | Java 方法 | 入参 | 出参 | 权限 | 说明 |
|---|---|---|---|---|---|---|
| `search` | 内置 | `search` | `Filter/properties/limit/offset/order` | 列表 | `mes_work_order:read` | 保留平台契约，不重写 |
| `create` | 内置重写 | `create` | `productId/plannedQuantity/priority/plannedStartDate/plannedEndDate/workCenterId/bomVersion/routingVersion/remarks` | `MesWorkOrder` | `mes_work_order:create` | 自动生成 workOrderNo；冗余 productName/productCode/uom；初始 status=DRAFT |
| `update` | 内置重写 | `update` | `id + 可编辑字段（按状态校验）` | `MesWorkOrder` | `mes_work_order:update` | 校验状态允许的可编辑字段范围；COMPLETED/CLOSED 禁止更新 |
| `delete` | 内置重写 | `delete` | `id` | 无 | `mes_work_order:delete` | 仅 DRAFT 状态可删，否则抛 ModelException；软删除 |
| `release` | 自定义 | `release` | `id: Long` | `MesWorkOrder` | `mes_work_order:release` | DRAFT→RELEASED；校验：id 不为空、状态为 DRAFT、权限 |
| `startProgress` | 自定义 | `startProgress` | `id: Long` | `MesWorkOrder` | `mes_work_order:start` | RELEASED→IN_PROGRESS；系统写入 actualStartTime |
| `complete` | 自定义 | `complete` | `id: Long` | `MesWorkOrder` | `mes_work_order:complete` | IN_PROGRESS→COMPLETED；系统写入 actualEndTime |
| `close` | 自定义 | `close` | `id: Long` | `MesWorkOrder` | `mes_work_order:close` | COMPLETED→CLOSED；最终归档 |

服务规则：
- `release`/`startProgress`/`complete`/`close` 均使用 `@MethodService(name="...", displayName="...", auth="mes_work_order:...")`
- 所有状态变更服务：先查库确认当前状态，状态不符合抛 `ModelException`（IIDP 请求级事务自动回滚）
- `create` 重写：读跨 App 产品信息冗余到工单字段（若跨 App 接口不可用则要求调用方传入冗余字段）
- `update` 重写：根据当前状态和调用方角色限制可更新字段范围

## 5. 视图和菜单

| 视图 key | 类型 | 说明 |
|---|---|---|
| `mes_work_order_grid` | grid | 列表：工单号、产品名称/编码、状态（彩色Tag）、计划数量、良品数量、废品数量、计划开始日期、计划结束日期、工作中心、优先级、操作栏 |
| `mes_work_order_search` | search | 搜索：工单号（模糊）、产品名称（模糊）、状态（下拉）、计划开始日期范围 |
| `mes_work_order_form` | form | 表单：产品（关联选择）、计划数量、优先级、计划开始/结束日期、工作中心、BOM 版本、工艺路线版本、备注 |

菜单：
- 菜单文件：`data/menus.json`
- 菜单 key：`mes_work_order_menu`
- `model`：`mes_work_order`
- `view`：`mes_work_order_grid,mes_work_order_search,mes_work_order_form`
- `parent_id`：`待确认`（挂载父菜单 key，由项目菜单结构决定）

操作栏按钮（从 §4 服务设计按钮权限列提取）：

| 按钮 | service | auth | 位置 | 显示条件 |
|---|---|---|---|---|
| 新增 | `create` | `mes_work_order:create` | tbar（工具栏） | 始终显示 |
| 编辑 | `update` | `mes_work_order:update` | 行操作（buttons） | 非 COMPLETED/CLOSED 状态 |
| 删除 | `delete` | `mes_work_order:delete` | 行操作（buttons） | 仅 DRAFT 状态 |
| 下达 | `release` | `mes_work_order:release` | 行操作（buttons） | 仅 DRAFT 状态 |
| 开工 | `startProgress` | `mes_work_order:start` | 行操作（buttons） | 仅 RELEASED 状态 |
| 完工 | `complete` | `mes_work_order:complete` | 行操作（buttons） | 仅 IN_PROGRESS 状态 |
| 关闭 | `close` | `mes_work_order:close` | 行操作（buttons） | 仅 COMPLETED 状态 |

> **规则**：`service` 值必须与 `@MethodService(name=...)` 或内置服务名完全一致；`auth` 在同一 grid 视图内唯一；按钮显示条件通过视图 `condition` 或 `auth` 控制，后端服务仍须校验权限和状态。

## 6. 数据和权限

- 种子数据：
  - 字典 `mes_work_order_status`：`DRAFT`(草稿) / `RELEASED`(已下达) / `IN_PROGRESS`(执行中) / `COMPLETED`(已完工) / `CLOSED`(已关闭)
  - 字典 `mes_work_order_priority`：`HIGH`(高) / `NORMAL`(普通) / `LOW`(低)
- 附件：无
- **权限码汇总**（从 §4 服务设计表的 `权限` 列聚合）：
  - 菜单权限：`mes_work_order:read`
  - 按钮/服务权限：
    - `mes_work_order:create` — 新增工单
    - `mes_work_order:update` — 编辑工单
    - `mes_work_order:delete` — 删除工单（仅草稿）
    - `mes_work_order:release` — 下达工单
    - `mes_work_order:start` — 开工确认
    - `mes_work_order:complete` — 完工报告
    - `mes_work_order:close` — 关闭归档
  - > 完整定义和角色映射放在 `integration-map.md` 权限码总览；本规格只声明本模型用到的权限码，不重复定义角色映射。
- 多租户/作用域：使用平台默认 scope 过滤；状态变更服务内校验 scope
- 多语言：`displayName` 使用中文，如需多语言后续补语言包

## 7. 验收

- [ ] `app.json` 路径与 `resolved` 包路径一致（`com.sie.iidp.mes`）
- [ ] `app.json.view` 登记 `views/mes_work_order_view.json`
- [ ] `app.json.data` 登记 `data/menus.json`、`data/mes_work_order_status.json`、`data/mes_work_order_priority.json`
- [ ] `apps/apps.json` 登记 `sie-iidp-demo-mes.jar`
- [ ] Java `@Model(name="mes_work_order")` 与视图 JSON `"model":"mes_work_order"` 和菜单 `"model":"mes_work_order"` 完全一致
- [ ] `views/mes_work_order_view.json` JSON 可解析，grid/search/form 三视图存在
- [ ] `data/menus.json` JSON 可解析，菜单 key `mes_work_order_menu` 存在
- [ ] 字典种子 `mes_work_order_status` 含 DRAFT/RELEASED/IN_PROGRESS/COMPLETED/CLOSED 五条记录
- [ ] 字典种子 `mes_work_order_priority` 含 HIGH/NORMAL/LOW 三条记录
- [ ] 4 个状态变更服务（release/startProgress/complete/close）的 `@MethodService(name=...)` 与视图按钮 `service` 字段完全对应
- [ ] 能编译：`mvn -s ./settings.xml -DskipTests clean package`

---

## 附：遗留问题（Phase 2）

| 问题 | 说明 |
|---|---|
| 详情页步骤条 + 左右布局 | Phase 2 前端扩展视图实现 |
| 角色级字段可编辑性控制 | Phase 1 由后端 update 服务校验；Phase 2 可在前端 hook 中根据角色做字段 disabled 控制 |
| 工单号流水号并发 | 高并发场景建议 Redis 原子自增；Phase 1 使用数据库序列/时间戳兜底 |
| 跨 App 产品/工作中心数据联动 | Phase 1 调用方传入冗余字段；Phase 2 接入跨 App Lookup 选择器 |
