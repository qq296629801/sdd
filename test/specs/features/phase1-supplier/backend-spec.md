# 供应商管理 IIDP 后端规格

## 1. 命名

| 项 | 值 | 说明 |
|---|---|---|
| appName | `demo-supplier` | 小写中划线 |
| appPkg | `supplier` | Java 包段，全小写 |
| moduleName | `supplier` | 业务模块目录 |
| model_name | `demo_supplier` | 小写下划线，全局唯一 |
| Java 类名 | `DemoSupplier` | UpperCamelCase |
| 菜单 key | `demo_supplier_menu` | 带业务前缀 |

## 2. 工程文件

| 文件 | 新增/修改 | 说明 |
|---|---|---|
| `sie-iidp-demo-apps/pom.xml` | 修改 | 登记 `sie-iidp-demo-supplier` 子模块 |
| `sie-iidp-demo-apps/sie-iidp-demo-supplier/pom.xml` | 新增 | 供应商模块 POM，依赖 `sie-iidp-demo-common` |
| `src/main/java/com/sie/iidp/supplier/app.json` | 新增 | App 描述，登记视图和菜单路径 |
| `model/DemoSupplier.java` | 新增 | 供应商元模型 |
| `views/demo_supplier_view.json` | 新增 | grid / search / form 三视图 |
| `data/menus.json` | 新增 | 供应商管理菜单入口 |
| `apps/apps.json` | 修改 | 登记 `sie-iidp-demo-supplier.jar` |

## 3. 模型设计

| 字段 | 类型 | IIDP 注解 | 必填 | 校验 | 索引 | 说明 |
|---|---|---|---|---|---|---|
| `name` | `String` | `@Property(displayName="供应商名称")` | 是 | `@NotBlank` | 唯一索引 | 供应商全称，全局唯一 |
| `contactPerson` | `String` | `@Property(displayName="联系人")` | 是 | `@NotBlank` | 否 | 主联系人姓名 |
| `contactPhone` | `String` | `@Property(displayName="联系电话")` | 是 | `@Pattern(手机号/固话)` | 否 | 联系电话 |
| `status` | `String` | `@Property(displayName="状态") @Selection(name="supplier_status")` | 是 | — | 是（查询过滤） | 枚举：`ENABLED`(启用) / `DISABLED`(禁用) |
| `remark` | `String` | `@Property(displayName="备注")` | 否 | — | 否 | 可选备注 |
| `deleted` | `Boolean` | `@Property @Deleted` | 系统 | — | 是 | 软删除标记 |

模型规则：
- 类注解：`@StaticVar @Getter @Setter @Model(name="demo_supplier", displayName="供应商")`
- 继承 `BaseModel<DemoSupplier>`
- `status` 字段使用 `@Selection` 绑定字典 `supplier_status`，不硬编码中文

## 4. 服务设计

| 服务 | 类型 | Java 方法 | 入参 | 出参 | 权限 | 说明 |
|---|---|---|---|---|---|---|
| `search` | 内置 | `search` | `Filter/properties/limit/offset/order` | 列表 | `demo_supplier:read` | 保留平台契约，不重写 |
| `create` | 内置 | `create` | `name/contactPerson/contactPhone/status/remark` | `DemoSupplier` | `demo_supplier:create` | 新增前校验 name 唯一 |
| `update` | 内置 | `update` | `id + 可编辑字段` | `DemoSupplier` | `demo_supplier:update` | 编辑前校验 name 唯一（排除自身） |
| `delete` | 内置 | `delete` | `id` | 无 | `demo_supplier:delete` | 软删除，标记 `deleted=true` |
| `toggleStatus` | 自定义 | `toggleStatus` | `id: Long` | `DemoSupplier` | `demo_supplier:toggle` | 切换 ENABLED↔DISABLED；校验：id 不为空、记录存在、有权限；不影响已关联单据 |

服务规则：
- `toggleStatus` 使用 `@MethodService(name="toggleStatus", displayName="禁用/启用", auth="demo_supplier:toggle")`
- 写服务（create/update/toggleStatus）必须校验权限、作用域、必填参数
- `delete` 为软删除，底层通过 `@Deleted` 字段实现，查询时平台自动过滤

## 5. 视图和菜单

| 视图 key | 类型 | 说明 |
|---|---|---|
| `demo_supplier_grid` | grid | 列表：名称、联系人、电话、状态、操作栏（编辑/删除/禁用）|
| `demo_supplier_search` | search | 搜索：供应商名称（模糊）、状态（下拉）|
| `demo_supplier_form` | form | 表单：名称、联系人、电话、状态、备注 |

菜单：
- 菜单文件：`data/menus.json`
- 菜单 key：`demo_supplier_menu`
- `model`：`demo_supplier`
- `view`：`demo_supplier_grid,demo_supplier_search,demo_supplier_form`
- `parent_id`：`待确认`（挂载父菜单 key，由项目菜单结构决定）

操作栏按钮：

| 按钮 | service | auth | 位置 |
|---|---|---|---|
| 新增 | `create` | `demo_supplier:create` | tbar |
| 编辑 | `update` | `demo_supplier:update` | 行操作 |
| 删除 | `delete` | `demo_supplier:delete` | 行操作 |
| 禁用/启用 | `toggleStatus` | `demo_supplier:toggle` | 行操作 |

## 6. 数据和权限

- 种子数据：字典 `supplier_status`，值：`ENABLED`(启用) / `DISABLED`(禁用)
- 字典：`supplier_status`（见上）
- 附件：无
- 菜单权限：`demo_supplier:read`
- 按钮权限：`demo_supplier:create` / `demo_supplier:update` / `demo_supplier:delete` / `demo_supplier:toggle`
- 服务权限：同按钮权限（后端 `@MethodService` auth 与按钮 auth 对齐）
- 多租户/作用域：使用平台默认 scope 过滤，`toggleStatus` 服务内校验 scope
- 多语言：`displayName` 使用中文，若需多语言后续补语言包

> 权限码完整定义见 `integration-map.md` 权限码总览，本规格仅声明用到的权限码，不重复定义角色映射。

## 7. 验收

- [ ] `app.json` 路径与 `resolved` 包路径一致（`com.sie.iidp.supplier`）
- [ ] `app.json.view` 登记 `views/demo_supplier_view.json`
- [ ] `app.json.data` 登记 `data/menus.json` 和字典种子文件
- [ ] `apps/apps.json` 登记 `sie-iidp-demo-supplier.jar`
- [ ] Java `@Model(name="demo_supplier")` 与视图 JSON `"model":"demo_supplier"` 和菜单 `"model":"demo_supplier"` 完全一致
- [ ] `views/demo_supplier_view.json` JSON 可解析，grid/search/form 三视图存在
- [ ] `data/menus.json` JSON 可解析，菜单 key `demo_supplier_menu` 存在
- [ ] 字典种子 `supplier_status` 含 `ENABLED`/`DISABLED` 两条记录
- [ ] 能编译：`mvn -s ./settings.xml -DskipTests clean package`
