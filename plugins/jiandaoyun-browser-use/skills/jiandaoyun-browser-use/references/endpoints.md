# 简道云内部接口清单（2026-10 实测，钉钉版简道云）

钉钉版租户，内置浏览器经 `www.jiandaoyun.com` 访问；读取类接口 2026-10-23 在内置浏览器中用 recipes「R14」复测。

通用：`POST`，`content-type: application/json`，`x-csrf-token: <meta[name=csrf-token]>`。
不要同时带 meta 的 `x-csrf-token` 和设计器抓到的 `X-CSRF-Token`（会 402）。`{app}` = 应用 ID，`{entry}` = 表单 ID。

## 应用 / 菜单
| 用途 | 接口 | 请求体 | 备注 |
|---|---|---|---|
| 应用列表 | 无接口；工作台 `/dashboard` 页面 DOM | — | `a[href*="#/app/"]` 取 24 位应用 ID 与名称（recipes R0） |
| 菜单（全部表单、仪表盘及 ID） | `/_/app/{app}/menu` | `{}` | 返回 `{entryList, entryMap, hasFlowMenu, homeList}`：`entryList` 只是目录树（`{id, child[]}`，不含名称）；表单 / 仪表盘取 `entryMap` 的值（`{type: form‖dash‖group, name, entryId, _id}`），ID 取 `entryId‖_id`；`homeList` 是首页置顶项 |
| 应用信息 | `/_/app/{app}/info_get` | `{}` | |

## 表单
| 用途 | 接口 | 请求体 | 备注 |
|---|---|---|---|
| 读定义 | `/_/admin/app/{app}/form/{entry}/edit` | `{}` | 返回 `{name, content{items, field_layout, …}, attr{hasFlow, hasCoop, flow, formEvents, …}, syncFields}`；字段在 `content.items[*].widget`，子表单字段在 `widget.items` |
| 保存定义 | `/_/admin/app/{app}/form/{entry}/update` | `{content, hasCache, hasCustomWidget, submit_btn, draft_btn, mobileLayout, pcLayout, formEvents, syncFields}` | 除 `content` 外取自 `attr`；新增字段要同时追加 `content.items` 与 `content.field_layout` |
| 表单改名 | `/_/admin/app/{app}/entry/{entry}/rename` | `{name}` | |
| 读流程 | `/_/admin/app/{app}/form/{entry}/workflows` | `{}` 或 `{version:"2"}` | 不带版本 = 已启用版本；带版本可读设计中版本（`editable:true`） |
| 新建流程设计版本 | `/_/admin/app/{app}/form/{entry}/workflow/create` | `{}` | 之后 `edit.attr.flow.designVer` 出现新版本号 |
| 写入 / 启用流程 | `/_/admin/app/{app}/form/{entry}/workflow/update` | `{flows, version, theme, flowFormViews, inuse}` | `inuse:true` = 启用；已启用版本不允许改节点结构；节点 `chargers.roles/departs` 须传 ID 列表（`users` 传 `_id`；传 username 曾报 3005） |
| 节点字段权限 | 流程节点 `optAuth` | `{widgetName或"子表单.字段": 位掩码}` | 1 = 可见，2 = 可编辑（3 = 可见可编辑） |

## 聚合表
| 用途 | 接口 | 请求体 | 备注 |
|---|---|---|---|
| 列表 | `/_/admin/app/aggregate_view/list` | `{appId}` | |
| 读配置 | `/_/admin/app/aggregate_view/edit` | `{appId, entryId}` | `view{source[], dimension[], metric[{name, formula, aggVars}], filterEmptyDimension}`；`aggVars` 可读出公式明文 |
| 改表名 | `/_/admin/app/aggregate_view/update` | `{appId, entryId, newName}` | |
| 设计器保存 | `/_/admin/app/aggregate_view/update` | `{appId, entryId, configChanged:true, view}` | 不带 `transmission` 时公式取 `view` 明文（已验证）；结构变化返回 `needInit:true` |
| 触发重算 | `/_/admin/app/aggregate_view/calculate` | `{appId, entryId}` | update 返回 `needInit:true` 后必须调（设计器 `doCalculate`）；返回 200 空体；进度看 list 里 `dataModifyTime` |
| 读计算结果 | `/_/data_process/admin/data/find`（`count` 同理） | `{appId, entryId:聚合表ID, filter, skip, limit, fields}` | `fields` 用维度 / 指标的 `widgetName`（`{name, type, title}`）；开放 API 读不了聚合表（3000） |

## 仪表盘
| 用途 | 接口 | 请求体 |
|---|---|---|
| 读 | `/_/admin/app/{app}/dash/{id}/edit` | `{}`（返回 `{_id, appId, entryId, name, components, styles, mobileStyles, attr, checkConfig}`；`components` 是以组件 ID 为键的对象，不是数组） |
| 写 | `/_/admin/app/{app}/dash/{id}/update` | `{components, styles, mobileStyles}` |

图表字段对象形如 `{name:"_widget_x", form, type, title}`。

## 智能助手
| 用途 | 接口 | 请求体 | 备注 |
|---|---|---|---|
| 列表 | `/automation/apis/process/list` | `{appId}` | `process_list[{process_id, name, enable, trigger_type, trigger_props, unpublished_tag}]` |
| 元数据 | `/automation/apis/process/get` | `{appId, process_id}` | 不含画布节点 |
| 改名 | `/automation/apis/process/rename` | `{appId, process_id, name}` | 模板类「A--->B」同步助手改名报 100116 |
| 启用 | `/automation/apis/process/enable` | `{appId, process_id}` | 只能启用（传 `enable:false` 也按启用处理，已启用时报 100026） |
| 停用 | `/automation/apis/process/disable` | `{appId, process_id}` | 返回 `{"status":"success"}`。注意 `update` 带 `inuse:true` 会把停用的助手发布并启用——要保持停用须随后调 disable |
| 读完整画布 | `/automation/apis/process/get_config` | `{appId, process_id}` | `flows[]`：`trigger_data`（`props.trigger_form`、`trigger_actions`）/ `plugin`（`props.plugin{plugin_id, symbol, agent_id, in_mappings}`）/ `end`；`version_info.stage` design / inuse |
| 新建 | `/automation/apis/process/create` | `{appId, trigger_type:"data_event", name, trigger_form:{app_id, entry_id}, tags:[]}` | 返回 `process_id`；新建后只有触发节点 |
| 保存 / 发布 | `/automation/apis/process/update` | `{appId, process_id, inuse, flows, name, trigger_other}` | `inuse:false` = 仅保存（design 版本）；`inuse:true` = 保存并启用（发布 + 启用）。**不改名**（改名用 rename）。flows 每项只传 `name/type/flow_id/next_ids/prev_ids/props`，插件节点须带 `output_fields:[]`，`props.plugin` 去掉只读的 `name` / `enable`，否则 400 |
| 插件实例与函数 | `/open/manage/agent/list` `{appId}`；`/open/manage/agent/list_trigger` `{appId, agent_id, has_applets:false}` | | 插件节点的 `agent_id` = 插件在本租户的实例；`symbol` 按函数名从 list_trigger 取 |

HTTP 触发助手的 webhook 地址：`https://api.jiandaoyun.com/api/v1/automation/tenant/{tenant}/hooks/` + `appId[0:12] + processId[0:12] + appId[12:] + processId[12:]`。

## 表单新建 / 流程表单 / 插件通用参数（2026-10-04 实测）
| 用途 | 接口 | 请求体 | 备注 |
|---|---|---|---|
| 新建表单 | `/_/admin/app/{app}/form/create` | `{name}` | 返回 `entry{entryId, type:"form", hasFlow:false, content:{items:[]}}`；之后用 R3 加字段 |
| 转为流程表单 | `/_/admin/app/{app}/form/{entry}/update` | `{hasFlow:true}` | 设计器「流程设定 → 开始使用」发的就是这个；**只配流程版本不开此项，提交 / API 新建都不会发起流程（flowState 为空 = 视为流转完成）** |
| 改字段别名 | `/_/admin/app/{app}/form/{entry}/update_widget_alias` | 未实测 | 普通 R3 提交改不掉 `widget.alias` |
| 插件实例（通用参数） | `/open/owner/agent/get` `{agent_id}` → `plugin_agent.assignments[{id, resolver_type:"literal", resolver_arg}]` | | apiKey 明文在 assignments 里，**不得输出**；`agent_id` 取自助手插件节点 `props.plugin.agent_id` |
| 改通用参数 | `/open/owner/agent/update` | `{agent_id, assignments}` | 只传这两项即可；读回仅 `assignments.*.resolver_arg` 与 `update_time` 变化 |

## 数据推送（webhook）
| 用途 | 接口 | 请求体 |
|---|---|---|
| 列表 | `/_/admin/app/webhook/list` | `{appId, limit, skip}` |
| 新建 | `/_/admin/workflow/webhook/create` | `{appId, entryId, webhook:{mode:"custom", url, triggers:["process_update"], type:"flow", name, remark, secret}}` |
| 推送日志 | `/_/admin/workflow/webhook/logs` | `{status:"success"‖"failed", limit, skip, appId, entryId, webhookId}` |
| 表单字段（推送配置页用） | `/_/admin/app/{app}/entry/{entry}/webhook/form_item/get` | `{}` |

`type:"flow"` + `process_update` = 「流程变更时推送·流程状态变更时」。

## 数据工厂
| 用途 | 接口 | 请求体 |
|---|---|---|
| 列表 | `/_/admin/app/{app}/list_etl` | `{}`（返回 `etlList`） |
| 读定义 | `/_/admin/app/{app}/etl/{id}/edit` | `{}` |
| 改名 | `/_/admin/app/{app}/etl/{id}/update` | `{name}` |

## 业务数据（管理端）
| 用途 | 接口 | 请求体 |
|---|---|---|
| 查询 | `/_/data_process/admin/data/find` | `{appId, entryId, filter:{rel, cond}, skip, limit, sort, fields:[字段对象]}`（`fields` 从数据管理页的请求里抓一份再筛选，子表单可裁剪 `items`） |
| 计数 | `/_/data_process/admin/data/count` | `{appId, entryId, filter}` |

## 开放 API v5（写数据用，需 apiKey）
`/v5/app/entry/data/{list,get,create,batch_create,update,delete}`，`Authorization: Bearer <apiKey>`。行为见 gotchas。
