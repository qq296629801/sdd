# IIDP SDD 工作流

## 总体工作流

```text
[第 0 步] 读取项目宪法和前后端 Skill
    ↓
[需求进入] 识别 IIDP 能力域
    ↓
[Specify] 写功能规格：模型、页面、服务、权限、数据、验收
    ↓
[Critique / Step 1.6] ★ 产品战略 + 工程风险双视角批判，暂停等用户确认（可选）
    ↓
[Backend Spec] 按 sdd-backend.md 模板生成 backend-spec.md（必写）
    ↓
[Frontend Spec] 需要前端代码？→ 按 sdd-frontend.md 模板生成 frontend-spec.md；标准模板页面跳过
    ↓
[Interaction Spec] 含复杂状态/响应式/可访问性？→ 按 sdd-frontend-interaction.md 模板生成 interaction-spec.md
    ↓
[Plan] 生成实现计划：后端优先，前端按实现分支决策
    ↓
[Plan Review Gate] ★ 展示计划摘要，暂停等用户确认后才生成任务
    ↓
[Tasks] 拆解可执行任务：文件级、服务级、视图级、验证级
    ↓
[Blueprint] ★ 生成代码蓝图（伪代码+文件清单），暂停等用户确认（可选）
    ↓
[Implement] 按任务落地：一次一个任务，子 skill 读取 spec 文件
    ↓
[Validate] 按 IIDP 清单验证；完成后提示生成 PR Bridge 描述
```

## 规格分层

| 层级     | 说明                                                  | 典型文件                                                              |
| -------- | ----------------------------------------------------- | --------------------------------------------------------------------- |
| 项目宪法 | 稳定原则、工程约束、路线图、UI 约束                   | `specs/mission.md`、`specs/iidp-stack.md`、`specs/ui-constitution.md` |
| 功能规格 | 单个业务能力的需求与契约                              | `specs/features/<feature>/requirements.md`                            |
| 后端规格 | IIDP 后端技术落地细节（模型、服务、视图、菜单、权限） | `specs/features/<feature>/backend-spec.md`                            |
| 前端规格 | IIDP 前端技术落地细节（仅需要前端代码时生成）         | `specs/features/<feature>/frontend-spec.md`                           |
| 实现计划 | 文件清单、实现顺序、风险                              | `specs/features/<feature>/plan.md`                                    |
| 任务清单 | 可逐项执行的工作项                                    | `specs/features/<feature>/tasks.md`                                   |
| 验收清单 | 静态、构建、运行、页面检查                            | `specs/features/<feature>/validation.md`                              |
| 维护记录 | 阶段复盘、决策、变更记录                              | `specs/roadmap.md`、`specs/decisions.md`、`CHANGELOG.md`              |

## 后端能力识别

每个需求先映射到 `skills/backend/references/core/capability-map.md` 中的能力域：

- 工程、应用、部署：POM、`app.json`、`apps/apps.json`、Docker Compose。
- 模型、字段、校验、ER：`@Model`、`@Property`、`@Validate`、ORM 注解。
- 标准页面：`views/*.json`、`grid/search/form/tree`、`tbar/buttons/tabs`。
- 菜单和种子数据：`data/menus.json`、`data/*.json`、`file/*.json`。
- 服务和 API：`@MethodService`、内置 CRUD、Filter、JSON-RPC、原生 SQL。
- 权限、多租户、多语言：`auth`、菜单权限、服务权限、语言包。
- 文件、Excel、打印、任务：MinIO、Excel 导入导出、打印、XXL-Job。

## 前端实现分支

| 分支              | 适用场景                                           | 处理方式                                                  |
| ----------------- | -------------------------------------------------- | --------------------------------------------------------- |
| 标准模板/在线视图 | 传统后台：搜索、表格、表单、树、子表               | 前端不新增代码，主要由后端视图和菜单配置完成              |
| hook              | 查询、保存、删除、权限控制、错误提示等标准流程增强 | 在扩展应用中写 hook，必要时调用 `vm.super`                |
| 扩展视图          | 插入按钮、调整节点、补充弹窗、合并属性             | 写 `before/after/append/merge/replace/delete/custom` 扩展 |
| 自定义 Vue2 组件  | IIDP 节点难以表达的复杂交互                        | 写 Vue2 兼容组件并按项目入口注册                          |

## 前端交互与节点规格

旧版通用前端 Spec 里的“组件层级、Props、事件、状态”在当前项目中要改写为 IIDP 视图节点、数据源和后端契约：

| 通用前端概念        | IIDP 当前项目写法                                                      |
| ------------------- | ---------------------------------------------------------------------- |
| Page/Component 层级 | `app > page > container > search/grid/form/dialog/table/button` 节点树 |
| Props               | 节点属性、`dataSource`、`ds_config.options`、后端视图字段配置          |
| State               | `$ds`、节点 `dataSource`、标准页 `vm.biz.data`、后端模型状态字段       |
| Event handler       | `bind_on_事件名`、hook、commands、按钮 `service/actionAfter`           |
| Selector            | `selector`、`vm.$select`、`vm.page.getNode(id)`，节点 id 必须有来源    |
| API call            | `ds_config` 的 `meta/api/static` 数据源或 JSON-RPC 服务契约            |
| Responsive layout   | 标准模板容器、`container`、`row`、`span`、弹窗/抽屉滚动策略            |

节点规格最少包含：

- `type`：IIDP 组件类型，必须来自平台组件或已注册自定义组件。
- `id`：稳定业务 id；无法确认时写“待确认”，不得按文案猜测。
- `selector`：扩展视图命中的目标节点，说明来源和验证方式。
- `ds_config`：数据源类型、`name`、`autoRequest`、`options.params`、`reqPrep`、`reqAfter`。
- `bind_`/`bind_two_`：属性和数据源的绑定关系。
- `bind_on_`：真实组件事件名和回调职责。
- `commands`/`$cmd`：公共命令名称、入参和调用位置。
- `hook`：标准页 hook 路径、是否调用 `vm.super`、读写的 `vm.biz` 字段。

## 功能目录

```text
specs/features/
└── phase1-[feature-name]/
    ├── requirements.md         # 功能需求与契约
    ├── backend-spec.md         # 后端技术落地规格（命名、模型、服务、视图、菜单、权限）
    ├── frontend-spec.md        # 前端技术落地规格（仅需要前端代码时生成）
    ├── interaction-spec.md     # 前端交互规格（含复杂状态/响应式/可访问性时生成）
    ├── plan.md
    ├── tasks.md
    └── validation.md
```

## Step 0：能力识别

```markdown
## 能力识别

- 业务对象：
- 模型清单（有多个模型时必填，按依赖顺序排列）：
  | # | 模型名 | Java 类名 | model_name | 与其他模型的关系 |
  |---|---|---|---|---|
  | 1 | [主模型] | `[MainEntity]` | `[app_main]` | 一对多 → [SubEntity] |
  | 2 | [子模型] | `[SubEntity]` | `[app_sub]` | ManyToOne → [MainEntity] |
- 跨模型自定义服务（有复杂业务流程时必填）：
  | 服务名 | 挂载模型 | 涉及模型 | 业务目标 |
  |---|---|---|---|
  | `[serviceName]` | `[model]` | `[model1], [model2]` | [说明] |
- 后端能力域：
- 前端实现分支：
- 是否需要新增 App：
- 是否需要新增前端扩展应用：
- 是否涉及权限/多租户/多语言：
- 是否涉及文件/Excel/打印/任务：
- 待确认事项：
```

## Step 1：Specify

`requirements.md` 必须回答：

| 问题             | 当前项目写法                                        |
| ---------------- | --------------------------------------------------- |
| 用户要完成什么   | 用业务语言描述流程和结果                            |
| 后端要提供什么   | 模型、字段、服务、视图、菜单、数据、权限            |
| 前端是否要写代码 | 标准模板/在线视图优先，必要时说明 hook 或扩展       |
| 契约是什么       | JSON-RPC、Filter、args、视图 key、节点 id、数据源名 |
| 如何验收         | 文件登记、编译、启动、页面链路、权限和异常          |

## Step 1.5：Spec（技术落地规格）

`requirements.md` 完成后、进入 Plan 之前，生成技术落地规格。后端规格每个功能必写；前端规格仅在有前端代码时生成。

### Step 1.5a：Backend Spec（必须生成）

读取 `sdd-backend.md` 模板，输出 `backend-spec.md`。**所有功能必须生成此文件**，它是 Step 4 后端实现的输入。

必须包含：

| 章节       | 内容                                                            | 对应 sdd-backend.md 章节 |
| ---------- | --------------------------------------------------------------- | ------------------------ |
| 命名       | appName、appPkg、moduleName、model_name、Java 类名、菜单 key    | §1                       |
| 工程文件   | 新增/修改文件清单（POM、app.json、视图、菜单、apps.json）       | §2                       |
| 模型设计   | 字段、IIDP 注解、校验、索引；多模型时先写 ER 总览再逐模型展开   | §3                       |
| 服务设计   | 内置/自定义服务、入参、出参、权限、事务边界                     | §4                       |
| 视图和菜单 | 视图 key、类型、grid/search/form 配置；菜单 key、model、view    | §5                       |
| 数据和权限 | 种子数据、字典、菜单/按钮/服务权限码（引用 integration-map.md） | §6                       |
| 验收       | app.json 登记、命名一致、JSON 可解析、编译                      | §7                       |

### Step 1.5b：Frontend Spec（条件生成）

**仅在 `requirements.md` 第 4 节判定"需要前端代码"时执行此步骤。** 标准模板/在线视图可完成的页面跳过，不生成此文件。

读取 `sdd-frontend.md` 模板，输出 `frontend-spec.md`，必须包含：

| 章节                 | 内容                                                    | 对应 sdd-frontend.md 章节 |
| -------------------- | ------------------------------------------------------- | ------------------------- |
| 页面定位             | 实现分支、目标工程、目标应用                            | §1                        |
| 页面结构             | 搜索区、表格区、工具栏、弹窗等区域划分                  | §3                        |
| UI 宪法约束          | 标准模板匹配度、容器策略                                | §4                        |
| 页面骨架与 IIDP 映射 | 骨架树、原型到 IIDP 承载方式                            | §5                        |
| 节点与绑定规格       | 节点树、selector、绑定/事件/命令                        | §6                        |
| 字段规格             | 控件类型、校验、默认值                                  | §7                        |
| 操作与事件           | 入口、触发条件、前端行为、后端服务                      | §8                        |
| IIDP 实现分支        | 标准模板/hook/扩展视图/自定义 Vue2 的选择依据和实现要求 | §9                        |
| 数据源与绑定         | ds_config、reqPrep/reqAfter、autoRequest                | §10                       |
| 待确认事项           | 节点 id、接口、模型、权限码                             | §13                       |

**`backend-spec.md` 和 `frontend-spec.md` 是 Step 4 实现阶段子 skill 的输入。** 缺失时必须先补全再进入实现。

### Step 1.5c：Interaction Spec（条件生成）

**触发条件**：满足以下任一时生成，否则跳过：
- 页面含复杂状态机（超过 3 个状态且有状态转移动作）
- 需要明确响应式布局策略（弹窗滚动、表格高度、栅格列宽）
- 有可访问性要求（WCAG、键盘操作、屏幕阅读器）
- 含批量操作、危险操作确认或多步向导流程

读取 `sdd-frontend-interaction.md` 模板，输出 `interaction-spec.md`，必须包含：

| 章节 | 内容 | 对应 sdd-frontend-interaction.md 章节 |
|---|---|---|
| 页面目标 | 目标用户、核心任务、推荐实现方式 | §1 |
| 用户流程 | 进入→操作→成功/失败完整路径 | §2 |
| IIDP 节点层级 | 页面节点树（page > container > 子节点） | §3 |
| 交互状态表 | 每个节点的 default/loading/empty/error/disabled 及触发条件 | §4 |
| 节点属性与数据契约 | id 来源、selector、ds_config、bind_on_、hook | §5 |
| 响应式与容器策略 | 栅格、弹窗尺寸、滚动归属、TABIFRAME | §6 |
| 可访问性与可用性 | label、必填标识、错误文字、危险操作确认 | §7 |
| 验收标准 | 可操作、可验证、可追踪的检查项 | §8 |

`interaction-spec.md` 与 `frontend-spec.md` **不重复输出**：节点 id / selector / ds_config 等技术细节写在 `frontend-spec.md`，交互状态表、用户流程、响应式策略写在 `interaction-spec.md`。

**三个技术规格文件总结：**

| 文件 | 来源模板 | 生成条件 | Step 4 读取方 |
|---|---|---|---|
| `backend-spec.md` | `sdd-backend.md` | 必须生成 | backend skill |
| `frontend-spec.md` | `sdd-frontend.md` | 需要前端代码时 | frontend skill |
| `interaction-spec.md` | `sdd-frontend-interaction.md` | 含复杂状态/响应式/可访问性时 | frontend skill |

---

## Step 1.6：Critique（可选，规格重大或不确定性高时触发）

Critique 从两个对立视角对 `requirements.md` 做批判性审查，目的是在进入实现计划前识别高风险假设和业务决策漏洞。**不修改规格，只输出发现报告并等待用户决策。**

### 触发条件

用户明确要求，或满足以下任一：规格涉及 3 个以上模型、包含状态机、有明显歧义的成功标准、待确认事项超过 3 项。

### 产品战略视角质疑点

- 这个功能解决的是用户真实痛点，还是技术负债或低频诉求？
- 功能边界是否清晰，是否会在实现中自然扩大范围（scope creep）？
- 成功标准是可量化验收的，还是模糊的主观判断？
- 是否存在更轻量的实现方式（如后端在线视图已满足，无需前端扩展）？
- 多模型设计是否必要，还是过度设计？

### 工程风险视角质疑点

- 规格中是否有"待确认"事项会在 Implement 阶段造成阻塞？
- ER 关系、状态机、跨模型服务的事务边界是否足够清晰？
- 权限码、视图 key、菜单 key 是否已经稳定，还是可能在实现中变更？
- 前端实现分支判断是否会因为节点 id 未确认导致返工？
- 是否有遗漏的验收场景（异常态、权限边界、并发修改、多租户）？

### Critique 输出格式

```markdown
## Critique 报告：[功能名称]

### 产品战略发现

- [高/中/低] [发现描述]

### 工程风险发现

- [高/中/低] [发现描述]

### 需要用户决策的事项

1. [问题描述] — 选项 A：[...] / 选项 B：[...]
```

输出后**暂停，等待用户决策**后再进入 Step 2 Plan。

---

## Step 2：Plan

```markdown
# 实现计划：[功能名称]

## 方案概述

[2-3 句话说明为什么采用这些 IIDP 能力]

## 后端改动

- Maven 模块：
- app.json：
- Java 模型：
- 服务：
- 视图 JSON：
- 菜单与种子数据：
- apps/apps.json：

## 前端改动

- 实现分支：标准模板/在线视图/hook/扩展视图/Vue2 组件
- 目标工程：
- 目标应用：
- 目标页面或节点 id：
- 需要修改的目录：

## 风险与待确认

- [接口/模型/节点 id/权限码/枚举]
```

## Plan Review Gate（Step 2 完成后的强制门控）

`plan.md` 生成后，AI **必须暂停**，向用户展示计划摘要并等待明确确认，再进入 Step 3 生成任务清单。**不得在用户确认前自动执行 Step 3。**

### 计划摘要模板

```markdown
## 计划摘要确认：[功能名称]

**后端改动量**：[N] 个模型 / [N] 个服务 / [N] 个视图文件
**前端实现分支**：[标准模板/hook/扩展视图/Vue2 组件]（[N] 个页面需前端代码，[N] 个无需）
**风险与待确认**：

- [待确认事项 1]
- [待确认事项 2]
  **估计复杂度**：S（< 1 天）/ M（1–3 天）/ L（> 3 天）

计划已就绪。**确认后生成任务清单**；如需调整，请说明修改点。
```

---

## Step 3：Tasks

单模型业务使用下方基础模板；**多模型业务必须为每个模型生成独立任务块**，并在末尾补充跨模型服务任务。

```markdown
# 任务清单：[功能名称]

## Git·分支准备（每个 feature 执行一次，写代码前必须完成）

- [ ] [S] 确认当前所在分支：`git branch --show-current`
- [ ] [S] 从主干创建 feature 分支：`git checkout -b feature/{appName}/phase{N}-{feature-name}`
- [ ] [S] 确认切换成功后再开始任何文件改动

## 后端·工程基础（只做一次）

- [ ] [S] 确认所有模型命名：appName、appPkg、moduleName、各 model_name、菜单 key
- [ ] [M] 新增或修改业务模块 POM
- [ ] [M] 新增 `app.json` 并预登记所有模型的 view/data 路径

## 后端·模型 1：[ModelName]

- [ ] [M] 编写 Java `@Model` 和字段（含 ER 关系注解）
- [ ] [M] 编写 `views/[model_name]_view.json`（grid/search/form）
- [ ] [S] 编写菜单入口和种子数据

## 后端·模型 2：[ModelName]

- [ ] [M] 编写 Java `@Model` 和字段（含 ManyToOne 关联父模型）
- [ ] [M] 编写 `views/[model_name]_view.json`
- [ ] [S] 编写菜单入口和种子数据

<!-- 有更多模型时，复制上方模型块，按依赖顺序追加 -->

## 后端·跨模型自定义服务（有复杂流程时必写）

- [ ] [M] 编写 `[serviceName]`：校验状态、权限、作用域；操作涉及模型；事务采用 IIDP 请求级事务（失败抛 `ModelException` 自动回滚），如需分段提交则用 `Meta.flush/commit`
- [ ] [M] 编写 `[serviceName2]`（如有）

## 后端·登记与收尾

- [ ] [S] 更新 `apps/apps.json` 登记新 jar

## 前端·实现分支判断（必先做）

- [ ] [S] 对每个页面/模型判断实现分支：标准模板 / hook / 扩展视图 / Vue2 组件
- [ ] [S] 标准模板可完成的页面，明确"前端无需新增代码"，跳过前端段
- [ ] [S] 需要前端介入的页面，列出目标工程、目标应用 `apps/<appName>`、目标节点 id 和 id 来源

## 前端·工程基础（仅全新前端时执行一次）

- [ ] [S] 使用 `tech project <projectName>` 创建工程
- [ ] [M] 使用 `tech app <appName>` 创建扩展应用
- [ ] [S] 配置 `apps/<appName>/config/app.json`、`effectPaths`

## 前端·页面 1：[页面名]（按页面/模型逐个展开）

- [ ] [M] 编写扩展视图 `apps/<appName>/views/<business>/<business>.js`
- [ ] [S] 在 `apps/<appName>/views/index.js` 汇总导出
- [ ] [M] 如需 hook：写在同一个 business 文件中，使用 `selector` + `type: "merge"` + `hook` 结构
- [ ] [M] 如需自定义 Vue 组件：放 `apps/<appName>/common/comps.js` 注册（扩展视图 JS 禁止 import `.vue`）
- [ ] [S] 如需公共组件复用：放 `apps/component/`

<!-- 有更多页面时，复制上方页面块，按业务顺序追加 -->

## 前端·收尾

- [ ] [S] 运行 `npm run lint` 或说明无法运行原因
- [ ] [S] 浏览器控制台用 `tech_app.printObj(节点id)` / `Tech.$page(id)` 验证节点

## 验证

- [ ] [S] 运行空白和 JSON 校验（所有模型视图文件）
- [ ] [M] 运行 Maven 编译或说明无法运行原因
- [ ] [M] 运行前端 lint/build 或说明无法运行原因
- [ ] [S] 对照页面流程人工验证（覆盖所有模型的增删改查和跨模型服务）
```

## Step 3.5：Blueprint（可选，复杂功能或团队初次接入 IIDP 时触发）

Blueprint 在开始写代码之前，先生成一份完整的代码蓝图（伪代码 + 文件清单），让用户验证方向正确后再执行。**不写真实代码，只描述结构和逻辑。**

### 触发条件

用户明确要求，或满足以下任一：L 级复杂度、包含跨模型服务、团队第一次在该项目使用 IIDP 前端扩展。

### 蓝图内容要求

**后端蓝图**：

- 每个 Java 文件的路径、类名、关键字段和方法签名（伪代码，不写完整实现）
- 每个视图 JSON 的结构骨架（key、model、主要按钮 service）
- `app.json` 的 view/data 登记路径清单

**前端蓝图**（需要前端代码时）：

- 每个扩展文件的路径和扩展类型（`before/after/merge/replace/custom`）
- 关键节点的 `selector` 和 `type`，标注节点 id 来源（用户提供 / 推导 / 待确认）
- hook 的挂载路径和伪代码逻辑（不写真实 JS）

### Blueprint 输出格式

```markdown
## 代码蓝图：[功能名称]

### 后端文件清单

| 文件路径                | 类型             | 关键内容                            |
| ----------------------- | ---------------- | ----------------------------------- |
| `src/.../XxxModel.java` | `@Model`         | 字段：id / name / status / ...      |
| `views/xxx_view.json`   | grid/search/form | 按钮：新增 / 编辑 / 删除 / [自定义] |
| `data/menus.json`       | 菜单             | 挂载：[父菜单 key]                  |

### 前端文件清单（仅需前端代码时）

| 文件路径                    | 扩展类型 | 目标节点    | id 来源 |
| --------------------------- | -------- | ----------- | ------- |
| `apps/[app]/views/[biz].js` | merge    | `[node_id]` | 待确认  |
```

输出后**暂停，等待用户确认蓝图**，确认后进入 Step 4 Implement。

---

## Step 4：Implement

**实现阶段必须经由对应子 skill，不要直接基于 create-project 的轻量任务清单写代码。**

| 任务类型                               | 必须经由                                                         | 输入文件                       |
| -------------------------------------- | ---------------------------------------------------------------- | ------------------------------ |
| 后端工程、模型、视图、菜单、服务、数据 | `skills/backend/SKILL.md` 的 Step 1～10 流程                     | `backend-spec.md`（必须存在）  |
| 前端规格 → 代码（含工程初始化）        | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-spec-code`      | `frontend-spec.md`（必须存在） |
| 前端扩展开发（已有工程内）             | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-extension-code` | `frontend-spec.md`             |
| 框架/组件/扩展协议查阅                 | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-dev-manual`     | 无                             |
| 节点 id 后缀推导                       | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-standard-ids`   | 无                             |

**重要**：

- 后端 skill (`skills/backend/SKILL.md`) 必须读取 `backend-spec.md` 作为输入，按 Step 1～10 执行代码生成。如果 `backend-spec.md` 缺失，必须先回到 Step 1.5a 生成。
- `iidp-frontend-spec-code` 必须读取 `frontend-spec.md`。如果 Step 1.5b 跳过了（标准模板页面），前端代码生成阶段也跳过。如果 `frontend-spec.md` 缺失但 tasks.md 标记有前端任务，必须先回到 Step 1.5b 生成。

执行规则：

- 一次只处理一个任务。
- 修改前读取相邻文件，沿用当前工程风格。
- 不顺手重构无关模块。
- 不回退用户已有变更。
- 对缺失事实写"待确认"，不补成假事实。
- **每个任务完成后回到 `tasks.md` 勾选状态，再启动下一个任务**；不要批量声明完成。

## Step 5：Validate

验收必须覆盖文件登记、命名一致、Java 模型、视图菜单、服务 API、前端扩展和部署运行。详细清单见 `sdd-validation.md`。
