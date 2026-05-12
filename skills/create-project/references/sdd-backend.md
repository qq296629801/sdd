# IIDP 后端 SDD 规格模板

生成后端规格时，先读取 `skills/backend/SKILL.md`，再按能力域读取对应专题。本文只提供 `create-project` 侧的 SDD 输出结构，不替代 backend skill 的硬性规则。

```markdown
# [功能名称] IIDP 后端规格

## 1. 命名

| 项 | 值 | 说明 |
|---|---|---|
| appName | `[demo-xxx]` | 小写中划线 |
| appPkg | `[xxx]` | Java 包段，全小写 |
| moduleName | `[xxx]` | 业务模块目录 |
| model_name | `[app_entity]` | 小写下划线，全局唯一 |
| Java 类名 | `[EntityName]` | UpperCamelCase |
| 菜单 key | `[app_module_menu]` | 带业务前缀 |

## 2. 工程文件

| 文件 | 新增/修改 | 说明 |
|---|---|---|
| `sie-iidp-demo-apps/pom.xml` | 修改 | 登记业务模块 |
| `sie-iidp-demo-apps/sie-iidp-demo-{appName}/pom.xml` | 新增 | 子模块 POM |
| `src/main/java/com/sie/iidp/{appPkg}/app.json` | 新增 | App 描述和资源登记 |
| `model/{ModelName}.java` | 新增 | 元模型 |
| `views/{model_name}_view.json` | 新增 | 后端视图 |
| `data/menus.json` | 新增/修改 | 菜单 |
| `apps/apps.json` | 修改 | 登记 jar |

## 3. 模型设计

| 字段 | 类型 | IIDP 注解 | 必填 | 校验 | 索引 | 说明 |
|---|---|---|---|---|---|---|
| `[fieldName]` | `[String/Date/...]` | `@Property` | 是/否 | `@Validate` | 是/否 | [说明] |

模型规则：
- 模型类使用 `@StaticVar @Getter @Setter @Model`，需要日志时加 `@Slf4j`。
- 继承 `BaseModel<T>`。
- 业务字段必须有 `@Property(displayName = "...")`。
- 选项、字典、关联字段用 `@Selection`、`@Dict` 或 ORM 注解。
- 唯一编码、外键、高频过滤字段和排序字段要考虑索引。

## 4. 服务设计

| 服务 | 类型 | Java 方法 | 入参 | 出参 | 权限 | 说明 |
|---|---|---|---|---|---|---|
| `search` | 内置 | `search` | `Filter/properties/limit/offset/order` | 列表 | `read` | 保留平台契约 |
| `[serviceName]` | 自定义 | `[methodName]` | `[args]` | `[result]` | `[auth]` | [说明] |

服务规则：
- 内置 CRUD 重写不得破坏平台入参契约。
- 自定义服务通过 `@MethodService` 暴露。
- 写服务必须校验状态、权限、作用域和必填参数。
- 原生 SQL 只在必要时使用，必须参数化。

## 5. 视图和菜单

| 视图 key | 类型 | 说明 |
|---|---|---|
| `{model_name}_grid` | grid | 列表 |
| `{model_name}_search` | search | 搜索 |
| `{model_name}_form` | form | 表单 |

菜单：
- 菜单文件：`data/menus.json`
- 菜单 key：
- `model`：
- `view`：`{model_name}_grid,{model_name}_search,{model_name}_form`
- `parent_id`：

## 6. 数据和权限

- 种子数据：
- 字典：
- 附件：
- 菜单权限：
- 按钮权限：
- 服务权限：
- 多租户/作用域：
- 多语言：

## 7. 验收

- [ ] `app.json` 路径与 `resolved` 一致
- [ ] `app.json.view/data` 登记完整
- [ ] `apps/apps.json` 登记新增 jar
- [ ] Java `@Model(name)` 与视图、菜单一致
- [ ] JSON 文件可解析
- [ ] 能编译或已说明无法编译的原因
```

## 审核重点

- 后端规格是否先确认 `appName`、`appPkg`、`moduleName`、`model_name`、菜单 key。
- 是否把视图、菜单、种子数据、附件、jar 登记到对应清单。
- 自定义服务是否保留 JSON-RPC、Filter、分页、排序和字段选择契约。
- 写服务是否有状态、权限、作用域、必填参数和并发边界校验。
- 权限是否同时覆盖菜单、按钮、服务，不依赖前端显隐兜底。
