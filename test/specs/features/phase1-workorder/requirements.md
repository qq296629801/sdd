# 工单信息管理 IIDP 功能需求

## 用户要完成什么

生产计划员需要创建、查询、编辑生产工单，并驱动工单沿"草稿 → 已下达 → 执行中 → 已完工 → 已关闭"五态流转；车间主管确认开工；操作工/班组长录入良品和废品数量；系统管理员可对所有工单进行管理。

## 后端要提供什么

- 工单模型，字段包含：工单号（自动生成）、产品（关联）、计划数量、良品数量、废品数量、状态、优先级、计划开始/结束日期、实际开始/结束时间、工作中心（关联）、BOM 版本、工艺路线版本、备注
- 软删除字段 `deleted`（`@Deleted`）
- 标准 CRUD 服务（新增、编辑、查询列表、查询详情）
- 5 个状态变更服务：`release`（下达）、`startProgress`（开工）、`complete`（完工）、`close`（关闭）

## 前端是否要写代码

- **列表 + 搜索 + 表单**：使用 IIDP 标准模板（grid + search + form），无需前端扩展代码
- **详情页**（步骤条 + 左右布局 + 内联编辑切换）：待确认是否标准模板可覆盖，还是需要扩展视图/自定义组件

## 契约

- appName: `demo-mes`
- appPkg: `mes`
- model_name: `mes_work_order`
- 视图 key 前缀: `mes_work_order_`
- 菜单 key: `mes_work_order_menu`
- 状态变更服务: `release`(`mes_work_order:release`) / `startProgress`(`mes_work_order:start`) / `complete`(`mes_work_order:complete`) / `close`(`mes_work_order:close`)
- 外部依赖模型：跨 App `weak` 引用，冗余字段存储，不建 ER 关联

## 核心业务规则

### 状态机

| 当前状态 | 目标状态 | 操作人 | 触发动作 |
|---|---|---|---|
| DRAFT（草稿） | RELEASED（已下达） | 生产计划员 | `release` |
| RELEASED | IN_PROGRESS（执行中） | 车间主管 | `startProgress`；设置 `actualStartTime` |
| IN_PROGRESS | COMPLETED（已完工） | 操作员/班组长 | `complete`；设置 `actualEndTime` |
| COMPLETED | CLOSED（已关闭） | 生产计划员 | `close` |

- 状态不可回退（管理员特殊操作除外，本期不实现）

### 字段可编辑性规则

| 状态 | 生产计划员可编辑 | 车间主管可编辑 | 操作员可编辑 |
|---|---|---|---|
| DRAFT | 所有业务字段 | — | — |
| RELEASED | 计划日期、工作中心 | 工作中心 | — |
| IN_PROGRESS | — | — | 良品数量、废品数量 |
| COMPLETED / CLOSED | 只读 | 只读 | 只读 |

### 删除规则

- 仅 `DRAFT` 状态允许删除，执行软删除（`deleted = true`）
- 非草稿状态不显示删除按钮

### 工单号生成规则

- 格式：`WO` + `YYYYMMDD` + 4 位流水号（如 `WO202604280001`）
- 在 `create` 服务内自动生成，用户不填

## 如何验收

- 能在菜单中找到工单管理入口
- 列表正常加载，按工单号（模糊）、产品名称（模糊）、状态、计划日期范围过滤有效
- 新增工单，工单号自动生成，状态默认草稿
- 编辑工单，草稿状态下所有字段可编辑
- 删除操作（二次确认），仅草稿态可删
- 下达、开工、完工、关闭状态变更正常触发，状态不可逆
- 非草稿状态下关键字段只读
- 无权限用户操作时后端返回权限错误

## Clarifications

### Session 2026-05-13

- Q: appName/appPkg/model_name 命名方案？ → A: `demo-mes` / `mes` / `mes_work_order` / `MesWorkOrder`
- Q: `pm_product` 和 `pm_work_center` 是跨 App 还是同 App？ → A: 跨 App 弱引用（`weak`）；工单冗余存 productName/productCode/uom/workCenterName，不在本 App 建 ER 关联
- Q: 状态变更是 4 个独立服务还是通用 changeStatus？ → A: 4 个独立 `@MethodService`（release/startProgress/complete/close），各自独立校验
- Q: 详情页步骤条+左右布局是否 Phase 1 实现？ → A: Phase 1 只用标准 form 视图；步骤条/左右布局留 Phase 2 前端扩展
- Q: 是否需要乐观锁 version 字段？ → A: 不加；状态变更服务内查库当前状态做状态校验，不符合抛 ModelException
