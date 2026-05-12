---
name: create-project
description: Use when 用户要创建、规划、标准化或编写 IIDP 项目文档，包括 SDD 项目宪法、需求规格、实现计划、任务拆解、验收清单、前后端契约映射，或把项目想法迁移成规格文档。
---

# 创建项目

## 概述

使用本技能把 IIDP 项目想法或现有项目需求转成可执行的 SDD 产物。先读取 `references/sdd.md` 索引，再按需读取拆分后的 `references/sdd-*.md` 专题；后端和前端事实来源分别是 `skills/backend` 与 `skills/frontend`。

## 核心流程

1. 先读取本 `SKILL.md`。
2. 当用户需要完整项目手册、项目宪法、SDD 流程、规格、计划、任务、验收或端到端建项时，先加载 `references/sdd.md` 索引。
3. 编写后端规格或代码前，先通过 `skills/backend/SKILL.md` 及其引用文件处理后端规则。
4. 编写前端规格或代码前，先通过 `skills/frontend/SKILL.md` 及其子技能处理前端规则。
5. 需要 UI 原型描述时，先提示用户补充任意支持 MCP 的原型/设计/产品工具服务、资源 URI、页面或 Frame 信息；能读取 MCP 资源时，用原型内容补全 UI 原型规格，不限定具体产品。
6. 明确 IIDP 契约：`appName`、`appPkg`、`moduleName`、`model_name`、视图 key、菜单 key、服务、入参、权限码、前端实现分支、节点 id、数据源。
7. 需要前端交互描述时，输出页面节点树、`selector`、`ds_config`、`bind_`、`bind_on_`、commands、hook 和状态验收点。
8. 未知模型、节点 id、权限、枚举、路由、接口和原型来源都标记为 `待确认`；不要编造平台事实。

## 按需加载

| 用户目标 | 读取文件 |
|---|---|
| 项目宪法、UI 宪法、路线图、集成总览 | `references/sdd-constitution.md` |
| SDD 流程、需求规格、计划、任务、验收目录 | `references/sdd-workflow.md` |
| 后端规格 | `references/sdd-backend.md`，再读取 `skills/backend/SKILL.md` |
| 前端规格、UI 原型、节点树、交互状态 | `references/sdd-frontend.md`，再读取 `skills/frontend/SKILL.md` |
| 前端交互设计规格、行为状态、响应式和可用性 | `references/sdd-frontend-interaction.md`，再读取 `skills/frontend/SKILL.md` |
| JSON-RPC、Filter、按钮服务、节点属性契约 | `references/sdd-contracts.md` |
| 验证、失败处理、阶段复盘、变更记录 | `references/sdd-validation.md` |
| 存量 IIDP 项目接入 | `references/sdd-brownfield.md` |
| create-project 详细使用说明 | `references/usage.md` |
| 修改本 skill 或检查 skill 规范 | `references/sdd-skill-maintenance.md` |
| Prompt、工具命令、核心原则、常见陷阱 | `references/sdd-prompts-tools-principles.md` |

## 生成 SDD 产物时

使用专题文件中的结构：

- 项目宪法：`mission.md`、`iidp-stack.md`、`roadmap.md`、`integration-map.md`、`decisions.md`。
- UI 宪法：`ui-constitution.md`，包含标准模板视觉约束、IIDP 组件规则、容器/iframe、可访问性、可用性和原型落地规则。
- 功能文档：`requirements.md`、`plan.md`、`tasks.md`、`validation.md`。
- 后端规格：命名、Maven 模块、`app.json`、Java 模型、服务、视图、菜单、种子数据、权限和验收。
- 前端规格：标准模板/在线视图判断、hook、扩展视图、Vue2 组件、节点树、selector、节点 id、数据源、绑定、事件和 commands。
- UI 原型规格：支持 MCP 的原型工具来源、页面骨架、区域布局、用户流程、状态、按钮、弹窗、空态、异常态，以及对应的 IIDP 承载方式。
- 集成契约：JSON-RPC `model/service/args`、Filter、视图 key、菜单 key、权限码、前端分支和未决事项。
- 复盘维护：Phase 完成后同步 `roadmap.md`、`integration-map.md`、`decisions.md` 和 `CHANGELOG.md`。

## IIDP 决策规则

- 传统管理后台页面优先使用后端标准视图和菜单。
- 写前端代码前，优先判断 IIDP 标准模板和在线视图是否已满足。
- 标准页生命周期、查询、保存、删除、权限和错误处理优先使用 hook。
- 节点插入、合并、替换和局部布局调整使用扩展视图。
- 只有 IIDP 节点无法表达复杂交互时，才使用自定义 Vue2 组件。
- 后端写服务必须校验状态、权限、作用域和必填入参；前端显隐不足以保证安全。

## 验证

声明完成前：

- 确认 `references/sdd.md` 是索引，详细内容已拆入 `references/sdd-*.md` 专题文件。
- 确认生成的 Markdown 没有未处理模板占位符，除非这些内容被明确标记为 `待确认`。
- 确认 UI 原型描述有 MCP、截图、导出文档或文字来源；无法读取时已标记“原型来源：待确认”。
- 确认前端规格包含 UI 宪法约束、页面节点树、selector、节点 id 来源、数据源、绑定、事件和 commands。
- 如果完成的是阶段或重要功能，确认 `roadmap.md`、`integration-map.md`、`decisions.md`、`CHANGELOG.md` 按需更新。
- 后端产物使用 `skills/backend/references/core/validation-checklist.md`。
- 前端产物使用相关 `skills/frontend` 子技能的验证规则。
- 按场景运行可用静态检查，例如 `git diff --check`、Markdown 代码块平衡、JSON 解析、Maven 构建或前端 lint/build。
