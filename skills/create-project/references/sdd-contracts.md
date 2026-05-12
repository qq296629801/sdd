# IIDP 前后端契约

## JSON-RPC 基本契约

```json
{
  "id": "guid",
  "jsonrpc": "2.0",
  "method": "service",
  "params": {
    "model": "{model_name}",
    "service": "search",
    "tag": "master",
    "app": "sie-iidp-demo-{appName}",
    "args": {
      "filter": [["status", "=", "ENABLE"]],
      "properties": ["id", "name"],
      "limit": 20,
      "offset": 0,
      "order": "id desc"
    }
  }
}
```

## Filter 规则

- 单条件：`[["name", "=", "ABC"]]`
- 关系字段：`[["org.parent.name", "like", "%华南%"]]`
- 逻辑符：`|`、`&`、`!`
- 常用操作符：`= != > >= < <= like ilike not like not ilike in not in child_of parent_of`
- 自定义查询服务必须继续支持平台 Filter、分页、排序和字段选择。

## 按钮与服务契约

```json
{
  "name": "启用",
  "action": "enable",
  "auth": "enable",
  "model": "{model_name}",
  "service": "enableDisable",
  "args": {
    "bind_ids": "$ds.checkedDataIds",
    "statusFlag": "ENABLE"
  },
  "actionAfter": "refreshTable"
}
```

后端必须提供匹配的 `@MethodService`，并在服务中重新查询记录、校验状态和权限，不能只信任前端按钮显隐。

## 节点 id 契约

前端扩展需要节点 id 时按顺序获取：

1. 用户明确提供。
2. 标准模板 ID 规则库推导，并标记为“待确认”。
3. 询问用户提供，或指导在浏览器控制台验证。

不得根据菜单名、按钮文案或模型名自行拼接节点 id。

## 节点属性与事件契约

前端扩展规格必须把“原型里的交互”转换为可执行的 IIDP 节点契约。

| 契约项 | 必写内容 | 常见风险 |
|---|---|---|
| 节点层级 | `page/container/search/grid/form/dialog/table/button` | 只写视觉区域，没有写节点落点 |
| `selector` | 目标节点 id、属性选择器、来源和验证方式 | selector 命中错误节点或运行时 id |
| `ds_config` | `type/name/autoRequest/options/reqPrep/reqAfter` | 数据源名不稳定、参数和后端服务不一致 |
| `bind_` | 绑定属性、数据路径、transform | `$ds` 路径不存在或跨节点选择器错误 |
| `bind_two_` | 表单字段和数据源双向关系 | 双向绑定导致保存参数结构漂移 |
| `bind_on_` | 真实事件名、回调职责、失败处理 | 臆造事件名或混用表格/按钮事件 |
| `commands` | 命令名、入参、调用位置 | 命令隐藏副作用，没有验收点 |
| hook | hook 路径、`vm.super`、读写 `vm.biz` | 覆盖平台默认查询、保存或权限逻辑 |

自定义 Vue2 组件只描述组件边界，不把组件内部状态当作 IIDP 平台事实。组件接入层仍要说明节点属性如何映射到组件 props，以及事件如何回传给 `bind_on_`、数据源或后端服务。
