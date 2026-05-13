# IIDP SDD 工作流

## 总体工作流

```text
[第 0 步] 读取项目宪法和前后端 Skill
    ↓
[需求进入] 识别 IIDP 能力域
    ↓
[Specify] 写功能规格：模型、页面、服务、权限、数据、验收
    ↓
[Plan] 生成实现计划：后端优先，前端按实现分支决策
    ↓
[Tasks] 拆解可执行任务：文件级、服务级、视图级、验证级
    ↓
[Implement] 按任务落地：一次一个任务
    ↓
[Validate] 按 IIDP 清单验证：静态检查、构建、运行、页面验证
```

## 规格分层

| 层级 | 说明 | 典型文件 |
|---|---|---|
| 项目宪法 | 稳定原则、工程约束、路线图、UI 约束 | `specs/mission.md`、`specs/iidp-stack.md`、`specs/ui-constitution.md` |
| 功能规格 | 单个业务能力的需求与契约 | `specs/features/<feature>/requirements.md` |
| 实现计划 | 文件清单、实现顺序、风险 | `specs/features/<feature>/plan.md` |
| 任务清单 | 可逐项执行的工作项 | `specs/features/<feature>/tasks.md` |
| 验收清单 | 静态、构建、运行、页面检查 | `specs/features/<feature>/validation.md` |
| 维护记录 | 阶段复盘、决策、变更记录 | `specs/roadmap.md`、`specs/decisions.md`、`CHANGELOG.md` |

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

| 分支 | 适用场景 | 处理方式 |
|---|---|---|
| 标准模板/在线视图 | 传统后台：搜索、表格、表单、树、子表 | 前端不新增代码，主要由后端视图和菜单配置完成 |
| hook | 查询、保存、删除、权限控制、错误提示等标准流程增强 | 在扩展应用中写 hook，必要时调用 `vm.super` |
| 扩展视图 | 插入按钮、调整节点、补充弹窗、合并属性 | 写 `before/after/append/merge/replace/delete/custom` 扩展 |
| 自定义 Vue2 组件 | IIDP 节点难以表达的复杂交互 | 写 Vue2 兼容组件并按项目入口注册 |

## 前端交互与节点规格

旧版通用前端 Spec 里的“组件层级、Props、事件、状态”在当前项目中要改写为 IIDP 视图节点、数据源和后端契约：

| 通用前端概念 | IIDP 当前项目写法 |
|---|---|
| Page/Component 层级 | `app > page > container > search/grid/form/dialog/table/button` 节点树 |
| Props | 节点属性、`dataSource`、`ds_config.options`、后端视图字段配置 |
| State | `$ds`、节点 `dataSource`、标准页 `vm.biz.data`、后端模型状态字段 |
| Event handler | `bind_on_事件名`、hook、commands、按钮 `service/actionAfter` |
| Selector | `selector`、`vm.$select`、`vm.page.getNode(id)`，节点 id 必须有来源 |
| API call | `ds_config` 的 `meta/api/static` 数据源或 JSON-RPC 服务契约 |
| Responsive layout | 标准模板容器、`container`、`row`、`span`、弹窗/抽屉滚动策略 |

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
    ├── requirements.md
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

| 问题 | 当前项目写法 |
|---|---|
| 用户要完成什么 | 用业务语言描述流程和结果 |
| 后端要提供什么 | 模型、字段、服务、视图、菜单、数据、权限 |
| 前端是否要写代码 | 标准模板/在线视图优先，必要时说明 hook 或扩展 |
| 契约是什么 | JSON-RPC、Filter、args、视图 key、节点 id、数据源名 |
| 如何验收 | 文件登记、编译、启动、页面链路、权限和异常 |

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

## Step 4：Implement

**实现阶段必须经由对应子 skill，不要直接基于 create-project 的轻量任务清单写代码。**

| 任务类型 | 必须经由 |
|---|---|
| 后端工程、模型、视图、菜单、服务、数据 | `skills/backend/SKILL.md` 的 Step 1～10 流程 |
| 前端规格 → 代码（含工程初始化） | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-spec-code`（自动编排 init + extension-dev） |
| 前端扩展开发（已有工程内） | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-extension-dev` |
| 框架/组件/扩展协议查阅 | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-dev-manual` |
| 节点 id 推导 | `skills/frontend/SKILL.md` 路由 → `iidp-frontend-standard-ids` |

执行规则：

- 一次只处理一个任务。
- 修改前读取相邻文件，沿用当前工程风格。
- 不顺手重构无关模块。
- 不回退用户已有变更。
- 对缺失事实写"待确认"，不补成假事实。
- **每个任务完成后回到 `tasks.md` 勾选状态，再启动下一个任务**；不要批量声明完成。

## Step 5：Validate

验收必须覆盖文件登记、命名一致、Java 模型、视图菜单、服务 API、前端扩展和部署运行。详细清单见 `sdd-validation.md`。
