# IIDP 后端 SDD 规格模板

生成后端规格时，先读取 `skills/backend/SKILL.md`，再按能力域读取对应专题。本文只提供 `create-project` 侧的 SDD 输出结构，不替代 backend skill 的硬性规则。

**输出文件路径**：`specs/features/<phase>-<feature>/backend-spec.md`

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

操作栏按钮（从 §4 服务设计按钮权限列提取）：

| 按钮 | service | auth | 位置 |
|---|---|---|---|
| 新增 | `create` | `{model_name}:create` | tbar（工具栏） |
| 编辑 | `update` | `{model_name}:update` | 行操作（buttons） |
| 删除 | `delete` | `{model_name}:delete` | 行操作（buttons） |
| [自定义操作] | `[serviceName]` | `{model_name}:[auth]` | 行操作（buttons） |

> **规则**：`service` 值必须与 §4 `@MethodService(name=...)` 或内置服务名完全一致；`auth` 值在同一 grid 视图内必须唯一；tbar 放全局操作（新增），buttons 放行级操作（编辑/删除/自定义）。

## 6. 数据和权限

- 种子数据：
- 字典：
- 附件：
- **权限码汇总**（从 §4 服务设计表的 `权限` 列聚合，格式 `{model_name}:{action}`）：
  - 菜单权限：`{model_name}:read`
  - 按钮/服务权限：`{model_name}:create` / `{model_name}:update` / `{model_name}:delete` / `[自定义 auth]`
  - > 完整定义和角色映射放在 `integration-map.md` 权限码总览；本规格只声明本模型用到的权限码，不重复定义角色映射。
- 多租户/作用域：
- 多语言：

> **权限码单一事实来源原则**：权限码的完整定义（含义、角色映射、涉及模型）只在 `integration-map.md`（项目级总览）和 `sdd-contracts.md`（前后端契约）中维护。本后端规格只引用权限码，不重复定义，避免三处不一致。

## 7. 验收

- [ ] `app.json` 路径与 `resolved` 一致
- [ ] `app.json.view/data` 登记完整
- [ ] `apps/apps.json` 登记新增 jar
- [ ] Java `@Model(name)` 与视图、菜单一致
- [ ] JSON 文件可解析
- [ ] 能编译或已说明无法编译的原因
```

## 多模型业务规格

业务涉及多个模型时（如主表 + 子表、主流程 + 关联记录），**必须在规格中枚举所有模型**，不得只生成第一个模型后停止。

### ER 关系总览（多模型时必写）

在命名表之后，先列出模型清单和关系，再逐一展开每个模型的完整规格：

```markdown
## ER 关系总览

| 模型 | Java 类名 | model_name | 说明 | 与其他模型的关系 |
|---|---|---|---|---|
| [主模型] | `[MainEntity]` | `[app_main]` | [说明] | 一对多 → [SubEntity] |
| [子模型] | `[SubEntity]` | `[app_sub]` | [说明] | ManyToOne → [MainEntity] |
| [关联模型] | `[RelEntity]` | `[app_rel]` | [说明] | ManyToOne → [MainEntity] |
```

### 逐模型规格展开

ER 关系确认后，**按依赖顺序**（被引用的模型先写）逐一展开每个模型的第 3～7 节规格。每个模型独立形成一份完整规格块：

```markdown
---
## 模型 N：[ModelName]

### 3. 模型设计
| 字段 | 类型 | IIDP 注解 | 必填 | 校验 | 索引 | 说明 |
|---|---|---|---|---|---|---|
| `[fieldName]` | `[String/Date/...]` | `@Property` | 是/否 | `@Validate` | 是/否 | [说明] |
| `[foreignKey]` | ManyToOne | `@ManyToOne` | 是 | — | 是 | 关联 [父模型] |

### 4. 服务设计
...（同单模型模板）

### 5. 视图和菜单
...

### 6. 数据和权限
...

### 7. 验收
...
---
```

### 跨模型自定义服务

涉及多个模型的业务流程服务（如"提交审批"需要同时操作主表和关联记录），规格中必须说明：

| 服务 | 挂载模型 | 涉及模型 | 事务边界 | 入参 | 校验项 |
|---|---|---|---|---|---|
| `[serviceName]` | `[主模型]` | `[主模型], [关联模型]` | IIDP 请求级事务（抛 `ModelException` 自动回滚） | `[args]` | 状态、权限、必填、作用域 |

## 审核重点

- 后端规格是否先确认 `appName`、`appPkg`、`moduleName`、`model_name`、菜单 key。
- **多模型业务是否有 ER 总览表，且每个模型都有独立的完整规格块。**
- **跨模型服务是否说明挂载模型、涉及模型、事务边界和校验项。**
- 是否把视图、菜单、种子数据、附件、jar 登记到对应清单。
- 自定义服务是否保留 JSON-RPC、Filter、分页、排序和字段选择契约。
- 写服务是否有状态、权限、作用域、必填参数和并发边界校验。
- 权限是否同时覆盖菜单、按钮、服务，不依赖前端显隐兜底。
