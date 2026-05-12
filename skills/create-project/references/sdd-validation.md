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
- [ ] `app.json.view` 登记视图
- [ ] `app.json.data` 登记菜单、字典、种子、附件
- [ ] `apps/apps.json` 登记新增 jar

### 命名一致

- [ ] `@Model(name)` 与视图 `model`、菜单 `model` 一致
- [ ] 视图 key 和菜单 key 带业务前缀
- [ ] 自定义按钮 `service` 对应 Java 服务

### 服务与权限

- [ ] 写服务校验状态、权限、作用域和必填参数
- [ ] 查询服务支持 Filter、分页、排序和字段选择
- [ ] 菜单、按钮、服务权限码稳定

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
- [ ] 目标节点 id 来源明确
- [ ] hook 路径、扩展类型、数据源名、commands 名称清晰
- [ ] 未修改 `node_modules`、`dist`、`distApp`、`distTmp`、`umdComps`、`build`

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
