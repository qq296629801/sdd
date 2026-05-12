# create-project Skill 维护规则

本手册已经迁入 `skills/create-project`，后续维护应围绕这个 skill 自身展开，不再创建旧版通用 SDD skill 或通用开源框架 skill。

## 目录结构

```text
skills/create-project/
├── SKILL.md                         # skill 入口：触发条件、核心流程、按需加载、验证门禁
├── agents/
│   └── openai.yaml                  # Codex UI 展示信息
└── references/
    ├── sdd.md                       # SDD 手册索引，不放长模板
    ├── sdd-constitution.md          # 项目宪法、UI 宪法、路线图、集成总览模板
    ├── sdd-workflow.md              # SDD 分层、功能迭代闭环、计划、任务、验收目录
    ├── sdd-backend.md               # IIDP 后端规格模板
    ├── sdd-frontend.md              # IIDP 前端、UI 原型、节点树、交互规格模板
    ├── sdd-frontend-interaction.md  # 前端交互设计规格
    ├── sdd-contracts.md             # JSON-RPC、Filter、按钮服务、节点属性与事件契约
    ├── sdd-validation.md            # 验证、失败分类、阶段复盘、变更记录
    ├── sdd-brownfield.md            # 存量 IIDP 项目接入
    ├── usage.md                     # create-project 详细使用说明
    ├── sdd-skill-maintenance.md     # 本 skill 的维护规则和校验命令
    └── sdd-prompts-tools-principles.md # Prompt、工具命令、核心原则、常见陷阱
```

## 维护原则

| 文件 | 维护原则 |
|---|---|
| `SKILL.md` | 保持精炼，只写触发条件、加载顺序、核心产物和验证门禁 |
| `references/sdd.md` | 只做导航索引和总原则，不承载长模板 |
| `references/sdd-*.md` | 按专题放详细模板、Prompt、契约、清单和示例 |
| `references/usage.md` | 放 create-project 详细使用说明，替代不符合 skill 规范的 README |
| `agents/openai.yaml` | 与 `SKILL.md` 的显示名称和默认提示保持一致 |
| `skills/backend` | 后端事实来源，本 skill 不覆盖其硬性规范 |
| `skills/frontend` | 前端事实来源，本 skill 不覆盖其节点、hook、扩展规则 |

## SKILL.md 必须覆盖

- 触发：创建、规划、标准化或编写 IIDP 项目文档。
- 加载：先读 `references/sdd.md` 索引，再按任务读取 `sdd-*.md` 专题。
- 产物：项目宪法、UI 宪法、功能规格、前后端契约、任务、验收、复盘和变更记录。
- 原型：需要 UI 原型描述时，先提示用户补充任意支持 MCP 的原型/设计/产品工具服务和资源 URI，不限定具体产品。
- 禁止：不编造模型、节点 id、权限码；不把后台页默认写成自定义前端；不引入无关开源框架模板。

## 分拆规范

- `SKILL.md` 不放长模板。
- `references/sdd.md` 控制在索引级别，优先让模型按需读取专题。
- 新增专题文件必须在 `references/sdd.md` 和 `SKILL.md` 的按需加载表中登记。
- 不新增 README、INSTALLATION、CHANGELOG 等无关说明文件。
- 如果某个 reference 超过约 500 行，继续按主题拆分。

## 校验命令

修改 skill 后至少运行：

```bash
python3 /Users/mjy/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/create-project
find skills/create-project -name '*.md' -print0 | xargs -0 -n1 awk 'BEGIN{c=0} /^```/{c++} END{if (c % 2 == 0) exit 0; print FILENAME ": unbalanced " c; exit 1}'
awk '/[ \t]$/{print FILENAME ":" FNR ":" $0; bad=1} END{exit bad}' skills/create-project/SKILL.md skills/create-project/references/*.md
```
