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
