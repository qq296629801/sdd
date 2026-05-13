# IIDP 实现、验证与复盘

## 标准执行顺序

```text
1. 读取规格和相关 Skill
2. 识别当前任务的后端能力域和前端实现分支
3. 修改最小必要文件
4. 运行静态检查
5. 能运行时执行构建或启动验证
6. 对照 validation.md 写验证结果
```

## 后端验证清单

### 文件登记

- [ ] `sie-iidp-demo-apps/pom.xml` 登记新增模块
- [ ] `app.json` 位于 `resolved` 包路径
- [ ] `app.json.view` 登记**所有模型**的视图文件路径
- [ ] `app.json.data` 登记菜单、字典、种子、附件
- [ ] `apps/apps.json` 登记新增 jar
- [ ] 多模型业务：模型清单中的每个 model 都对应有 Java 类文件、视图文件、菜单入口或子表挂载点

### 命名一致（每个模型独立检查）

对模型清单中的**每一个模型**重复以下检查：

- [ ] 模型 [N]：`@Model(name)` 与视图 JSON 的 `model`、菜单的 `model` 完全一致
- [ ] 模型 [N]：视图 key 和菜单 key 带业务前缀（如 `{appPkg}_`）
- [ ] 模型 [N]：自定义按钮 `service` 对应该模型的 Java `@MethodService`
- [ ] ER 关系：`@ManyToOne`/`@OneToMany` 指向的目标模型已存在（本应用内或跨应用 `strong` 依赖）

### 服务与权限

- [ ] 每个模型的写服务都校验：状态、权限、作用域、必填参数
- [ ] 每个模型的查询服务都支持：Filter、分页、排序、字段选择
- [ ] 跨模型服务：标注挂载模型、涉及模型、事务边界（默认 IIDP 请求级事务，抛 `ModelException` 自动回滚；分段提交需说明 `Meta.flush/commit` 用法），并对所有涉及模型的状态做联合校验
- [ ] 菜单、按钮、服务权限码与权限码总览表一致
- [ ] 权限码在三处对齐：后端服务 `auth` + 视图按钮 `auth` + 权限码总览表

### 数据兼容性（存量项目接入时）

- [ ] 模型字段类型与已有数据库表兼容；增加非空字段时已规划默认值或迁移脚本
- [ ] 状态枚举（如 `DRAFT/RELEASED/...`）与已有数据兼容；新增枚举值不破坏旧数据
- [ ] 唯一索引、外键约束变更已评估锁表风险

### 后端命令

```bash
git diff --check
find iidp-backend-demo-ai -name '*.json' -print0 | xargs -0 -n1 node -e 'JSON.parse(require("fs").readFileSync(process.argv[1],"utf8"))'
(cd iidp-backend-demo-ai && mvn -s ./settings.xml -DskipTests clean package)
(cd iidp-backend-demo-ai && docker compose config)
```

## 前端验证清单

### 工程和应用

- [ ] `tech --help` 可用，或已说明无法使用
- [ ] 工程由 `tech project` 创建
- [ ] 扩展应用由 `tech app` 创建
- [ ] `apps/<appName>/config/app.json` 不含 `^/需要配置/`
- [ ] `effectPaths.includeRegExp` 不带 `/iidp/` 前缀

### 扩展语义

- [ ] 实现分支判断正确，没有为标准页硬写前端代码
- [ ] 目标节点 id 来源明确（用户提供 / 标准模板规则库推导 / 浏览器控制台验证），不得按文案猜测
- [ ] hook 路径、扩展类型、数据源名、commands 名称清晰
- [ ] 未修改 `node_modules`、`dist`、`distApp`、`distTmp`、`umdComps`、`build`
- [ ] **完全替换标准模板页面时使用 `replace`，不得使用 `type: 'page'`**（`type: 'page'` 仅用于新增独立路由页面）

### Hook 与扩展规则

- [ ] 所有 hook 方法**必须 return 数据**（业务逻辑在 return 之前完成）
- [ ] 调用 `vm.super['hook.path']` 时必须 return 其返回值，不丢失平台原逻辑
- [ ] `before*` hook 返回处理后的 params；`after*` hook 返回原方法或业务数据；`can*` hook 返回 true/false
- [ ] 刷新表格用 `vm.biz.grid.methods.runRefresh()`，不用 `query()`
- [ ] 不编造 `vm.biz` 上的方法或属性路径；不确定时标注待确认

### 安全与代码质量

- [ ] 动态内容拼接到 HTML 时通过 `escapeHtml` 转义
- [ ] API 请求统一走 IIDP 数据源或 `window.Tech.httpMeta`，禁止引入 axios/fetch
- [ ] 变量声明使用 `const`/`let`，禁用 `var`
- [ ] Vue 组件注册只在 `apps/<appName>/common/comps.js` 完成；扩展视图 JS 文件禁止 import `.vue`

### 前端命令

```bash
npm run lint
npm run build
npm run start
```

## 偏差处理 Prompt

```text
你的实现偏离了 IIDP 规格：[指出具体位置]。

请按以下顺序处理：
1. 对照 requirements.md 和 skills/backend、skills/frontend 说明偏差原因。
2. 如果是实现错误，修正实现。
3. 如果是规格不适合当前 IIDP 架构，先提出修改规格的建议，等待确认后再改。
4. 不要引入通用开源框架写法替代 IIDP 标准能力。
```

## 失败分类与处理

验证失败时先诊断，不要直接改代码。按以下分类写入 `validation.md` 或阶段复盘：

| 分类 | 判断依据 | 处理方式 |
|---|---|---|
| 规格问题 | requirements/plan 中的模型、节点、权限、流程与 IIDP 能力冲突 | 先修改规格或提出 ADR，确认后再改实现 |
| 实现问题 | 规格正确，但文件、命名、登记、代码或配置不符合 | 修正最小必要实现，并重新运行失败项 |
| 环境问题 | 私服、数据库、Redis、MinIO、端口、CLI、权限不可用 | 记录环境事实、阻塞项和可替代静态验证 |
| 平台能力限制 | 标准模板、在线视图、hook 或组件无法表达需求 | 标记限制，给出扩展视图或自定义组件方案 |
| 原型输入不足 | MCP 不可用、Frame 未提供、截图不完整 | 请求用户补充任意支持 MCP 的原型/设计/产品工具服务，或把来源标记为待确认 |

测试或构建失败 Prompt：

```text
验证失败，错误信息如下：
[粘贴错误]

请先分类：规格问题、实现问题、环境问题、平台能力限制，还是原型输入不足。
然后说明证据、影响范围和最小修复方案。
在分类完成前不要直接修改代码。
```

## 阶段复盘与变更记录

每完成一个 Phase 或重要功能后，更新项目维护文档：

- `roadmap.md`：标记已完成项、调整后续阶段、记录技术债优先级。
- `integration-map.md`：同步新增或变更的模型、视图 key、菜单 key、服务、权限码、前端分支。
- `decisions.md`：记录会影响后续实现的技术或业务决策。
- `CHANGELOG.md`：记录用户可感知功能、修复和技术债。

`decisions.md` 建议格式：

```markdown
# 决策记录

## ADR-[编号]：[决策标题]

- 日期：
- 背景：
- 决策：
- 影响：
- 替代方案：
- 后续动作：
```

`CHANGELOG.md` 建议格式：

```markdown
# CHANGELOG

## [未发布]

### 新增
- [功能或页面]

### 变更
- [行为或契约变化]

### 修复
- [问题修复]

### 技术债
- [已知限制和计划处理阶段]
```

阶段复盘 Prompt：

```text
Phase [N] 已完成。请只更新规格和维护文档，不写代码：
1. 总结本阶段完成项和计划差异。
2. 更新 roadmap.md 的完成状态和下一阶段。
3. 更新 integration-map.md 的前后端契约。
4. 更新 decisions.md，记录影响后续实现的决策。
5. 更新 CHANGELOG.md 的 [未发布] 部分。
6. 列出遗留技术债、验证缺口和待确认事项。
```

## Spec Sync：规格与实现漂移检测

在以下时机触发漂移检测：合并 PR 后、存量项目接入后、发现实现与规格描述不一致时。

### 检测范围

| 检测项 | 规格来源 | 实现来源 | 漂移判断 |
|---|---|---|---|
| 模型字段 | `requirements.md` 字段清单 | Java `@Model` 文件 | 字段名、类型、必填不一致 |
| 视图按钮与服务 | `integration-map.md` 契约表 | `views/*.json` 按钮 `service` | service 名、auth 不对齐 |
| 菜单挂载 | `requirements.md` 菜单路径 | `data/menus.json` | 父菜单 key 或 view key 变更 |
| 权限码 | `integration-map.md` 权限码总览 | Java `@MethodService` `auth` 参数 | 权限码增删或改名 |
| 前端扩展节点 | 前端规格 `selector` 描述 | 实际扩展文件 | selector 目标节点 id 变更 |

### 漂移处理规则

发现漂移时，**先判断方向再决定处理方式**：

| 方向 | 含义 | 处理方式 |
|---|---|---|
| 代码超前于规格 | 实现加了功能但规格没更新 | 更新 requirements/integration-map，decisions.md 记录原因 |
| 规格超前于代码 | 规格写了但未实现 | 加入任务清单或标记为技术债 |
| 真实冲突 | 规格与实现互相矛盾 | 写入 decisions.md，明确以哪个为准后再处理 |

### 漂移检测命令

```bash
# 查找视图按钮的 service 声明，对照 integration-map
rg -n '"service"' iidp-backend-demo-ai/sie-iidp-demo-apps --include="*.json"

# 查找后端服务 auth 参数，对照权限码总览
rg -n '"auth"' iidp-backend-demo-ai/sie-iidp-demo-apps --include="*.json"

# 查找 @Model(name)，对照 model_name 命名
rg -n '@Model\(name' iidp-backend-demo-ai/sie-iidp-demo-apps --include="*.java"

# 查找前端扩展 selector，对照前端规格
rg -n 'selector' [frontend-project]/apps/<appName>/views

# 查找规格中的待确认项
rg -n '待确认' specs/features/
```

Spec Sync Prompt：

```text
请对 [specs/features/phaseN-feature/] 和当前工程代码做漂移检测：
1. 对照上方检测范围逐项检查。
2. 按"代码超前 / 规格超前 / 真实冲突"分类列出每条漂移。
3. 给出每条漂移的最小处理建议（更新规格 / 加任务 / 写 ADR）。
4. 在分类完成前不要修改任何文件。
```

### 漂移修复：propose / apply / backfill

发现漂移后，**不要立即修改文件**。按三步流程完成修复闭环：

```text
propose（提方案）→ 用户审批 → apply（执行方案）
                              ↗ 若为代码超前且无规格 → backfill（反向回填）
```

#### Step 1 — propose：生成修复提案

对每条漂移，AI 输出结构化修复提案，**不执行任何文件操作**：

提案格式：

```markdown
## 漂移修复提案

### D-01（代码超前于规格）
- **漂移描述**：OrderService.submit 新增了 `remark` 字段，但 requirements.md 未登记。
- **目标文件**：`specs/features/phase1-order/requirements.md`
- **建议变更**：在字段清单中补充 `remark: String?`（可选备注）。
- **影响评估**：不影响其他功能；纯规格补录。
- **操作类型**：update-spec

### D-02（规格超前于代码）
- **漂移描述**：integration-map.md 登记了 `order:export` 权限码，但 OrderService 未实现导出方法。
- **目标文件**：`specs/features/phase1-order/tasks.md`
- **建议变更**：新增任务 `实现 OrderService.export`，优先级 P1。
- **操作类型**：add-task

### D-03（真实冲突）
- **漂移描述**：requirements.md 要求状态枚举为 DRAFT/SUBMITTED，代码实现为 DRAFT/PENDING/APPROVED。
- **目标文件**：`specs/features/phase1-order/decisions.md`
- **建议变更**：写入 ADR 说明代码三态的业务来源，以代码为准并更新规格。
- **操作类型**：write-adr + update-spec
- **需用户决策**：是以代码为准（三态），还是以规格为准（二态）？
```

propose Prompt：

```text
请对 [specs/features/phaseN-feature/] 漂移检测结果生成修复提案：
1. 每条漂移输出一个提案块（D-xx）：漂移描述、目标文件、建议变更、影响评估、操作类型。
2. 操作类型只能是：update-spec / add-task / write-adr / backfill-spec。
3. 真实冲突类漂移必须标注「需用户决策」，并列出两个选项的影响。
4. 不要执行任何修改；输出提案后等待用户逐条审批。
```

#### Step 2 — apply：执行已审批提案

用户确认提案（逐条或批量确认）后，AI 按以下规则执行：

| 操作类型 | 执行动作 | 输出 |
|---|---|---|
| `update-spec` | 更新 requirements.md 或 integration-map.md 对应字段 | 显示 diff，标注来源（反向提取/规格修正）|
| `add-task` | 在 tasks.md 末尾追加任务，标注优先级和来源漂移 ID | 显示新增任务内容 |
| `write-adr` | 在 decisions.md 追加 ADR 条目，标注以哪一侧为准 | 显示 ADR 内容 |
| `backfill-spec` | 触发 backfill 流程（见下方）| 输出回填草稿 |

执行规则：
- 只执行已明确审批的条目；跳过「需用户决策」未决条目。
- 每条执行完成后，在漂移登记表中标注 `✅ 已处理` 和处理时间。
- 同一文件有多条修改时，合并为单次 Edit 操作，不多次写入。

apply Prompt：

```text
以下漂移提案已审批，请依次执行（仅执行列出的 ID）：
- [D-01]：update-spec，目标 requirements.md
- [D-03]：write-adr，以代码三态为准

执行规则：
1. 按操作类型执行，不扩大范围。
2. 显示每条操作的 diff 或新增内容。
3. 执行完成后更新漂移登记表，标注「✅ 已处理」。
4. 未审批的条目（如 D-02）跳过，不操作。
```

#### Step 3 — backfill：代码超前时反向回填规格

当某个模块完全没有规格（存量项目接入或早期开发未写 spec），使用 backfill 从代码反向生成规格草稿：

**触发条件**：
- 能力分层结果中存在「自定义扩展（技术债）」条目。
- 漂移检测发现代码实现有 5 条以上无规格对应。
- 用户明确要求补写规格。

**回填输出**：生成草稿片段，插入对应规格文件，**全部标注 `[反向提取，待确认]`**：

```markdown
<!-- backfill: 以下内容由代码反向提取，须人工核对后去除标注 -->

### 模型字段（反向提取，待确认）
| 字段名 | Java 类型 | 必填 | 来源 |
|---|---|---|---|
| orderNo | String | 是 | OrderModel.java:@Property(name="orderNo") |
| remark | String | 否 | OrderModel.java:@Property(name="remark") |

### 服务契约（反向提取，待确认）
| 服务名 | auth | 入参 | 来源 |
|---|---|---|---|
| submit | order:submit | orderId: Long | OrderService.java:@MethodService |
```

backfill Prompt：

```text
请对 [模块路径/Java 类] 做规格反向回填：
1. 从 @Model、@Property 提取字段清单（字段名、类型、必填、注解原文）。
2. 从 @MethodService 提取服务契约（方法名、auth、核心入参、返回类型）。
3. 从 views/*.json 提取视图 key、按钮 service 和权限码。
4. 从前端扩展文件提取 selector、hook 路径、数据源名。
5. 所有输出条目标注 [反向提取，待确认] 和来源文件行号。
6. 输出格式为可直接粘贴到 requirements.md / integration-map.md 的 Markdown 片段。
7. 不修改任何文件；输出草稿等待用户确认后再执行 apply。
```

**回填后的必要动作**：
- 用户确认回填草稿后，执行 `apply` 写入规格文件。
- 在 decisions.md 记录回填来源、确认人和确认时间。
- 回填完成后运行完整漂移检测，确认无残留「代码超前于规格」条目。

---

## PR Bridge：从规格自动生成 PR 描述

Step 5 Validate 全部通过后，AI 主动提示："已可生成 PR 描述，是否生成？"；用户也可随时触发。

### 生成规则

从 `specs/features/<feature>/` 目录读取规格文件，按以下结构生成：

```markdown
## PR 描述模板

**Title**：`[Phase N] [功能名称]`（不超过 70 字符）

**Body**：

## 功能概述
[来自 requirements.md 的使命陈述，1–2 句话]

## 变更范围
### 后端
- [来自 tasks.md 后端任务块的 M 任务汇总]

### 前端
- [来自 tasks.md 前端任务块的 M 任务汇总，或"标准模板，无前端代码改动"]

## 规格目录
[`specs/features/phaseN-feature-name/`](specs/features/phaseN-feature-name/)

## 验收清单
- [ ] [来自 validation.md 的关键验收项]

## 待确认 / 技术债
- [来自 requirements.md 和 tasks.md 中未解决的待确认事项，若无则写"无"]
```

### PR Bridge Prompt

```text
请基于 specs/features/[phaseN-feature-name]/ 目录，生成 PR 描述：
- Title 不超过 70 字符，格式：[Phase N] [功能名称]
- Body 按 PR Bridge 模板输出：功能概述、变更范围（后端/前端）、规格目录链接、验收清单、待确认事项
- 变更范围只列 M 级（中等）及以上任务，S 级（小）任务归并为一行摘要
- 待确认事项来自规格，若全部已解决则写"无"
```
