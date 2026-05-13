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

**跨 App 外部模型引用（弱引用 + 冗余字段模式）**：

当需要引用其他 App 的模型（如产品、工作中心、组织）且不做强依赖时，使用此模式：

| 字段 | 类型 | 注解 | 说明 |
|---|---|---|---|
| `[externalId]` | `Long` | `@Property(displayName="[外部对象]ID")` | 存外部模型主键，弱引用 |
| `[externalName]` | `String` | `@Property(displayName="[外部对象]名称")` | 创建/编辑时从外部接口冗余，防止外部改名影响本表 |
| `[externalCode]` | `String` | `@Property(displayName="[外部对象]编码")` | 同上，按需冗余 |

- `app.json` 中声明 `"depends": [{"app": "[外部 appName]", "type": "weak"}]`
- `create`/`update` 服务负责读取外部数据并写入冗余字段；若外部服务不可用，要求调用方显式传入冗余字段
- 不使用 IIDP `@ManyToOne` 跨 App 关联（会引入强依赖）

## 4. 服务设计

| 服务 | 类型 | Java 方法 | 入参 | 出参 | 权限 | 副作用 | 说明 |
|---|---|---|---|---|---|---|---|
| `search` | 内置 | `search` | `Filter/properties/limit/offset/order` | 列表 | `{model_name}:read` | 无 | 保留平台契约 |
| `create` | 内置/重写 | `create` | `[业务字段]` | `[ModelName]` | `{model_name}:create` | 写入主表 | [说明] |
| `update` | 内置/重写 | `update` | `id + [可编辑字段]` | `[ModelName]` | `{model_name}:update` | 更新主表 | [说明] |
| `delete` | 内置/重写 | `delete` | `id` | 无 | `{model_name}:delete` | 软删除 | [说明] |
| `[serviceName]` | 自定义 | `[methodName]` | `[args]` | `[result]` | `{model_name}:[auth]` | [副作用] | [说明] |

> **副作用列**：列出服务执行成功后自动写入的字段、生成的关联记录、外部调用等。失败时由 IIDP 请求级事务统一回滚（见事务决策树）。

服务规则：
- 内置 CRUD 重写不得破坏平台入参契约。
- 自定义服务通过 `@MethodService` 暴露。
- 写服务必须校验状态、权限、作用域和必填参数。
- 原生 SQL 只在必要时使用，必须参数化。
- **三级异常规范**（来自 `sdd-contracts.md` §5）：
  - 字段级必填/长度/唯一 → 模型层 `@Validate`
  - 参数缺失或格式错误 → 抛 `ValidationException`（**不**触发事务回滚）
  - 状态不允许、权限不足、业务流程失败 → 抛 `ModelException`（触发请求级事务回滚）

**状态机服务（多服务状态流转）**：

业务对象有多个状态且每个流转对应独立操作时，为每个流转定义独立的 `@MethodService`，不合并为通用 `changeStatus`：

| 服务 | 类型 | 前置状态 | 后置状态 | 入参 | 权限 | 副作用 |
|---|---|---|---|---|---|---|
| `[action1]` | 自定义 | `[FROM]` | `[TO]` | `id: Long` | `{model_name}:[action1]` | [如：写 actualStartTime] |
| `[action2]` | 自定义 | `[TO]` | `[NEXT]` | `id: Long` | `{model_name}:[action2]` | [如：写 actualEndTime] |

> 完整状态机契约（状态 × 服务映射、按钮显示规则、异常分层）在 `sdd-contracts.md` 状态机章节维护，backend-spec.md 只声明服务清单和副作用，不重复状态机规则。

- 每个状态变更服务：**先查库当前状态 → 状态不符合抛 `ModelException`**（请求级事务自动回滚）
- `@MethodService(name="[action]", displayName="[中文操作名]", auth="{model_name}:[action]")`
- 视图按钮 `service` 字段使用 `@MethodService.name`，不使用 Java 方法名

**事务决策树（跨模型 / 跨 App 写操作）**：

```
同一 App，所有操作在一次请求内完成？
  ├─ 是 → IIDP 请求级事务（默认）
  │        失败抛 ModelException，整体回滚，无需额外处理
  │
  └─ 否
       ├─ 需要部分提交（如：草稿保存后发通知，通知失败不回滚主数据）？
       │    └─ 使用 Meta.flush()/commit() 手动分段提交
       │       在 backend-spec.md §4 说明注释中标注"分段提交：原因"
       │
       └─ 跨 App 写操作（如：工单确认 → 仓库扣料）？
            └─ 设计补偿服务 → 见下方"跨 App 写操作补偿声明"
```

**跨 App 写操作补偿声明**（跨 App 无统一事务，必须声明补偿服务）：

| 服务 | 挂载模型 | 调用的跨 App 操作 | 失败场景 | 补偿服务 | 幂等保证 |
|---|---|---|---|---|---|
| `[serviceName]` | `[主模型]` | `[外部 App].[服务]` | 外部调用失败 / 超时 | `[compensationService]`（逆向操作） | 入参携带业务唯一键，重试不产生重复数据 |

> 跨 App 写操作补偿服务在 Phase 1 可标记"待实现"并说明原因；但补偿策略必须在 backend-spec.md 中声明，不得推到实现阶段才考虑。

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

**前端实现决策（详情页）**：

| 详情页需求 | Phase 1 处理方式 | Phase 2 处理方式 |
|---|---|---|
| 标准字段查看/编辑 | IIDP 标准 form 视图，无需前端代码 | — |
| 状态步骤条（步骤条组件） | 标记为 Phase 2，Phase 1 用状态字段 Tag 代替 | 前端扩展视图插入步骤条节点 |
| 左右分栏布局 + 操作日志 | 标记为 Phase 2 | 扩展视图或自定义 Vue2 组件 |
| 内联编辑/只读切换（行内 toggle） | 标记为 Phase 2 | hook 控制字段 `disabled` 状态 |

> 遇到复杂详情页布局时，先在 backend-spec.md §5 末尾添加"**附：遗留问题（Phase 2）**"表格记录，不在 Phase 1 规格中展开，避免规格超出交付范围。

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
- [ ] §8.1–8.4 规格-产物对照表已填写，无空行、无 ❌ 项
- [ ] §8.5 自定义服务实现核查已填写，所有 @MethodService 均有对应核查组，无 ❌ 项

## 8. 规格-产物对照表（生成后必填）

> **填写规则**：生成完所有产物后逐行填写。"一致"列填 ✅（一致）或 ❌（不一致，需立即修正），不得留空或标"待确认"。发现 ❌ 必须在提交前修正并重新填 ✅。

### 8.1 字段对照（§3 → Java @Property）

| §3 字段名 | Java @Property name | 类型一致 | 注解一致 |
|---|---|---|---|
| `[fieldName]` | `[fieldName]` | ✅/❌ | ✅/❌ |

### 8.2 服务对照（§4 → Java @MethodService / 内置服务）

| §4 服务名 | Java 方法 / 内置 | auth 码与 §4 一致 |
|---|---|---|
| `[serviceName]` | `@MethodService(name="[serviceName]")` / 内置 | ✅/❌ |

### 8.3 按钮对照（§5 buttons → §4 服务 → §6 权限码）

| 按钮 service | §4 中存在 | auth 码 | §6 中存在 |
|---|---|---|---|
| `[serviceName]` | ✅/❌ | `[model_name]:[auth]` | ✅/❌ |

### 8.4 视图与菜单登记对照（§5 → app.json）

| 视图 key | app.json.view 已登记 | 菜单 view 字段引用 |
|---|---|---|
| `[model_name]_grid` | ✅/❌ | ✅/❌ |
| `[model_name]_search` | ✅/❌ | ✅/❌ |
| `[model_name]_form` | ✅/❌ | ✅/❌ |

### 8.5 自定义服务实现核查（每个 @MethodService 一组）

> 参照 `skills/backend/references/core/method-service.md` 的实际代码模式逐项核查，不得使用业务具体名称作占位符。

**通用核查项（每个自定义服务必查）**：

| 核查项 | 标准（来自 method-service.md） | 确认 |
|---|---|---|
| 注解 name 与视图 service 一致 | `@MethodService(name="[serviceName]")` 中的 `name` 值 == 视图 button `"service": "[serviceName]"` | ✅/❌ |
| 方法签名与服务类型匹配 | 重写 create → `(List<Map<String,Object>> valuesList)`；业务方法 → `(RecordSet rs, [业务入参])`；计算字段 → `(Map<String,Object> valMap)` | ✅/❌ |
| 异常处理结构正确 | `catch (ModelException\|ValidationException e){throw e;}` + `catch(Exception e){throw new ValidationException("...")}` | ✅/❌ |
| §4 副作用字段全部赋值 | §4 服务行"副作用"列声明的字段，在方法体中能找到对应赋值语句 | ✅/❌ |

**状态机服务额外核查项**（仅状态变更类 @MethodService）：

| 核查项 | 标准 | 确认 |
|---|---|---|
| 从 DB 查当前记录 | 用 `DbUtils.search(Filter.equal("id", id), ...)` 或 `rs.callSuper(null, MethodConst.FIND, ...)` 查库，**不**用入参中的状态字段 | ✅/❌ |
| 前置状态校验抛 ModelException | `if (!前置状态.equals(record.getStatus())) throw new ModelException("...")` | ✅/❌ |
| 后置状态字段显式赋值 | `values.put("status", [后置状态常量])` 或直接 `record.setStatus(...)` | ✅/❌ |

**跨模型/跨 App 服务额外核查项**：

| 核查项 | 标准 | 确认 |
|---|---|---|
| 使用平台 RPC 调用 | `getMeta().get("[target_model_name]").call("[serviceName]", ...)` | ✅/❌ |
| 跨 App 调用有补偿声明 | §4 跨 App 补偿声明表中已登记（允许 Phase 1 标"待实现"） | ✅/❌ |
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

### 8. 规格-产物对照表
...（同单模型模板，按本模型字段/服务填写）
---
```

### 跨模型自定义服务

涉及多个模型的业务流程服务（如"提交审批"需要同时操作主表和关联记录），规格中必须说明：

| 服务 | 挂载模型 | 涉及模型 | 事务边界 | 入参 | 校验项 |
|---|---|---|---|---|---|
| `[serviceName]` | `[主模型]` | `[主模型], [关联模型]` | IIDP 请求级事务（抛 `ModelException` 自动回滚） | `[args]` | 状态、权限、必填、作用域 |

## 审核重点

- **§8.1–8.4 规格-产物对照表是否已填写，无 ❌ 项**（字段、服务、按钮、视图/菜单全部对齐）。
- **§8.5 自定义服务实现核查是否已填写**，覆盖所有 @MethodService；状态机服务核查了"从 DB 查状态"和"抛 ModelException"；跨模型服务核查了 `getMeta().get(...).call(...)` 模式。
- 后端规格是否先确认 `appName`、`appPkg`、`moduleName`、`model_name`、菜单 key。
- **多模型业务是否有 ER 总览表，且每个模型都有独立的完整规格块。**
- **跨模型服务是否说明挂载模型、涉及模型、事务边界和校验项。**
- 服务表格是否有**副作用列**，副作用（自动写字段、生成关联记录、外部调用）已明确声明。
- **有状态机的模型是否输出状态 × 服务映射表**（含前置状态、后置状态、系统副作用）。
- **跨 App 写操作是否有补偿服务声明**（补偿服务、幂等保证、失败场景），不得推到实现阶段。
- 事务边界是否经过决策树判断（请求级 / Meta 分段提交 / 跨 App 补偿），不得默认用请求级事务处理跨 App 写。
- 是否把视图、菜单、种子数据、附件、jar 登记到对应清单。
- 自定义服务是否保留 JSON-RPC、Filter、分页、排序和字段选择契约。
- 写服务三级异常是否对齐：`@Validate` / `ValidationException` / `ModelException`。
- 权限是否同时覆盖菜单、按钮、服务，不依赖前端显隐兜底。
