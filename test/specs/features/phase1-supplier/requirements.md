# 供应商管理 IIDP 功能需求

## 用户要完成什么

运营人员需要维护公司合作的供应商基本信息，支持新增、编辑、禁用/启用供应商，以及按名称和状态搜索供应商列表。

## 后端要提供什么

- 供应商模型，字段包含：名称、联系人、联系电话、状态（启用/禁用）、备注
- 标准 CRUD 服务（新增、编辑、删除、查询列表）
- 禁用/启用切换服务

## 前端是否要写代码

使用 IIDP 标准模板（grid + search + form），无需前端扩展代码。

## 契约是什么

- appName: `demo-supplier`，appPkg: `supplier`，model_name: `demo_supplier`
- 视图 key 前缀：`demo_supplier_`
- 菜单 key：`demo_supplier_menu`
- 禁用/启用服务：`toggleStatus`，auth: `demo_supplier:toggle`

## Clarifications

### Session 2026-05-13

- Q: 状态枚举存储值？ → A: `ENABLED` / `DISABLED`（英文），displayName "启用"/"禁用"
- Q: appName 和 model_name？ → A: `appName=demo-supplier`，`model_name=demo_supplier`
- Q: 删除方式？ → A: 软删除，加 `deleted` 字段（`@Property` + `@Deleted`）
- Q: 供应商名称唯一？ → A: 是，加唯一索引
- Q: 禁用后关联单据？ → A: 不影响已有单据，仅禁止新建关联

## 如何验收

- 能在菜单中找到供应商管理入口
- 列表正常加载，搜索按名称和状态过滤有效
- 新增、编辑、删除操作正常
- 禁用操作后供应商状态变更
