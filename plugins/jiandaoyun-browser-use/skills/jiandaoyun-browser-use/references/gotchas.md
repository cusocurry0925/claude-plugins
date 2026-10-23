# 平台行为与踩坑（2026-10 实测）

## 内部接口
- 字段引用（公式、联动、聚合表、助手、仪表盘）都按字段 ID（`_widget_…`）——只改显示名不会断引用。
- 表单设计器「保存」在没有改动时是禁用的；`button.click()` 对禁用按钮无效 → 用 R3 直接提交。大表单用真实鼠标点保存常不触发，JS `click()` 可靠。
- 拦截保存时 abort 请求后，按钮会卡在「保存中」，需刷新页面再继续。
- 已启用的流程版本不允许改节点结构（3005）；先 `workflow/create` 出设计版本。
- 流程节点 `optAuth` 是位掩码：1 可见、2 可编辑、4 未明（常见 5、7）、子表单另有 243 等操作位。**「必填」对只读（只可见）字段同样生效**：只读必填字段为空无法提交（用户确认，2026-10-04）。前端事件可以往只可见字段写值——「点按钮写提示、提示必填只读」即可强制「先预览才能提交」，且经办人无法手填绕过。
- 流程节点 `chargers.roles` 读出是完整角色对象，保存要传角色 `_id` 列表（原样回传报 3005，`meta.flowId` 指向出错节点）。
- 聚合表设计器默认把公式加密进 `transmission`，但服务端也接受**不带 `transmission`、公式明文放在 `view`** 的提交（设计器里 `window.jdy_config.transmission_encryption_enable=false` 时就是这么发的）——可直接构造整份 `view` 提交（R12）。改结构后返回 `needInit:true`——**设计器此时会接着调 `/_/admin/app/aggregate_view/calculate {appId, entryId}` 触发重算；直接提交时必须自己补这一步**，否则聚合表停在旧数据、连源表新写入也不再增量更新（2026-10-04 生产移动加权表实测：补调后约 1 分钟算完）。
- 聚合表指标公式里 `$_aggregation_x#` 是本指标 `aggVars` 的聚合变量，`$_widget_x#` 引用其他指标；可前向引用。过滤写法：`{type,method:"in"|"ne"|"empty"|"not_empty",value?,entryId,field}`（主表字段）或加 `subform,subField`（子表字段）。
- 从生产复制的应用：表单 ID、字段 ID 与生产一致；每个关联数据字段的 `relationship`、聚合字段的 `relationshipId` 是应用独有的；助手、数据推送会复制过来但停用，**推送地址仍指向生产助手**。
- **直接提交表单定义（R3/R6）时，成员 / 部门字段的默认值必须写成 ID 字符串**：读出是 `{"_id":"100000000000000000000001","nickname":"当前用户"}`，原样回传会被静默丢弃（2026-10-04 测试应用 5 张表踩坑）。提交前把所有 user/dept/usergroup/deptgroup 的 `value` 转成 `_id`（当前用户 `100000000000000000000001`、当前用户所处部门 `110000000000000000000001`），提交后逐项核对。设计器自己保存时就是这么发的。
- 复制出的应用里当前账号可能没有表单「提交」数据权限（界面提示「数据操作权限不足」），需在表单发布 → 成员权限里加。
- 数据推送「流程状态变更时」只由界面上的流程操作触发；开放 API 新建并 `is_start_workflow` 发起的流程不产生推送。

- 聚合表「过滤空值」开启时，任一维度字段为空的明细行整行丢弃（含出库行），会导致聚合表少扣 / 少计；排查数量差异时先查维度字段是否为空。

- **跨表复制字段（尤其子表单列）时，必须把 `widget.rely`（公式 / 默认值 / 联动）里引用的旧字段 ID 按「旧→新」映射替换**，否则设计器里显示「失效变量」「存在已删除字段」。复制后扫描所有新字段的 `rely`，确认只引用本表字段（2026-10-04 生产领料「FIFO分配明细.入库时间」踩坑）。

- **改字段公式不能靠拦截保存请求**：设计器保存时 `widget.rely.formula.formula` 是空串，公式正文另走加密通道；拦截只能改到 `rely.widgets`（依赖列表），会造成「依赖已改、公式没改」的半改状态（2026-10-04 采购退货「辅_D·汇总」测试应用踩坑）。正确做法：设计器里选中字段 → 默认值「已设置公式」打开公式编辑器 → `.CodeMirror` 的 `CodeMirror.setValue('FUNC()')` 并把光标放进括号 → 在「搜索变量」框输入字段名 → **真实鼠标**点击搜索结果插入变量（JS 派发事件无效）→ 点「确定」→ 保存；读回 `/edit` 核对 `rely.widgets` 与 `rely.formula.formula`。
- **流程表单的提交校验按节点配置，表单级 `content.validators` 不生效**（2026-10-04 测试应用实测：表单级加 `MIN(...)>=0` 后页面提交照样成功；改到流程节点 `validator` 后立即拦截）。节点校验在 `workflows` 每个节点的 `validator: [{formula, remind, remark}]`，公式明文、可经 `workflow/update` 直接写入并生效；只加在能编辑相关字段的节点（通常是发起节点）。生产其他出库表单级的「出库数量不能大于当前可用库存数量」即因此从未生效。
- 表单设计器保存时校验公式同样走加密通道（`validators[].formula` 为空串 + 顶层 `transmission`）。
- **`workflow/update` 节点负责人 `chargers.users` 传用户 `_id`**；传 username 会报 3005 `meta.flowId=该节点`（2026-10-04 其他出库「FIFO」节点）。R7 的 KEYS 还须带 `revoke,draft_text,allowDraft`，否则发起节点「暂存」会被重置为 true。新版本读回的开始 / 结束 / 抄送节点会补 `comment / pickChargers` 默认值、审批节点 `revoke` 空列表读回为无——属正常。
- **复制字段当模板时别取系统绑定字段**：其他出库「流程状态」下拉框带 `widget.alias:"flow_state"`，复制到新表单后数据键变成 `flow_state`、记录里不再有 `flowState`；alias 经 R3 删不掉（服务端保留），只能新建干净字段、旧字段隐藏作废。复制前先 `delete widget.alias` 并检查是否还有其他系统属性（2026-10-04 成本调整单 503 → 516）。
- **流程节点的版本绑定**：进行中的流程实例按发起时的流程版本走，改了节点（如关掉「手写签名 / 操作意见必填」）只对之后发起的单据生效。
- 新建表单若要走审批，除了 workflow/create + update 启用版本，还必须 `form/update {hasFlow:true}`（见 endpoints），否则提交即视为流转完成。
- **拦截保存请求时不要 `abort()`**：被中止的请求不会回调，设计器「保存」按钮会一直转圈、页面卡死且离开时被 beforeunload 拦下。校验不通过时应还原改动后照常发送（等于原样提交）。
- **截图坐标 ≠ 页面坐标**（内置浏览器 2026-10-23 实测）：`computer` 的坐标按最近一次截图报告的「coordinate frame」（如 800×643），页面 `clientX = 坐标 × innerWidth / frame 宽`（实测 innerWidth 1120 → ×1.4，纵向同比例）。反过来点某元素：坐标 = `getBoundingClientRect()` 中心 × frame 宽 / innerWidth。能拿到 `ref` 就用 `ref` 点（落在元素正中），别按截图目测。坐标点击和 `ref` 点击都是真实鼠标事件（`isTrusted:true`），可用于「JS 派发事件无效」的场景；拿不准时先挂 `mousedown` 捕获监听（`preventDefault` 吞掉事件）校准一次。

## 开放 API v5
- 子表单更新：只传行 `_id` + 部分字段会**清空该行其余字段**——必须整行回写（读出整行，改值后带 `_id` 回传）。
- 写入不重算表单公式：依赖公式的小计、合计要自己算好一起写。
- `is_start_trigger:false`（默认）写入不触发智能助手，不会循环。
- 流水号字段由系统生成，新建时传入的值被忽略——新建后读回编号。
- 新建数据的 `createTime` 为当前时间；流程表单经 API 新建且不发起流程时 `flowState` 为空（常被当作「流转完成」）。
- 对下拉字段用 `in` 筛选可用。
- 3005「The request contains invalid parameter(s)」常见原因：apiKey 无效（例如带了模板尖括号 `<…>`），先查 key。
- 自建插件调试页：请求最外层键是 `auth_conf` / `agent_conf` / `trigger_conf`，通用参数不会自动带入，缺 apiKey 报 17020。
- 自建插件导入包只支持 text 入参 + any 出参；子表单 / 数字入参、object[] 出参会导致预览报「开放平台服务调用失败」，导入后手工补建。单函数 72KB 代码可导入。

## 内置浏览器工具（2026-10-23 实测）
- `javascript_tool` 返回 7KB 以上不截断；含 `=` `&` `?` / URL 的输出不拦截。CSRF 令牌、apiKey 仍绝不输出。
- 单次执行 45 秒超时，超时后脚本可能仍在页面里运行；页面刷新 / 跳转会清空 `window` 变量，`sessionStorage` 保留。
- 每个调用都带自己的 `tabId`；不带时作用于当前前台标签，可能是用户的页面。
- `preview_start` 不会新开标签：每次都把固定的 `seed` 标签跳到新 URL 并切到前台。新开标签用 `tabs_create`（返回 `tab-1` 这类 ID）再 `navigate {url, tabId}`。
- 未登录时 `www.jiandaoyun.com/dashboard` 会跳到 `portal.finecloud.com/portal/signin`；登录由用户在 Browser 面板里完成。
- 工作台刚加载时应用链接还没渲染（实测刷新后立即读到 0 个链接），R0 里要等链接出现再读。
- `navigate` 只改 hash 时页面不刷新，网络记录不会重置。
- `read_network_requests` 列表只有方法、URL、状态；按 `requestId` 取响应体。要看请求体需自装 XHR 记录器（pass-through，记录 `body` 与 `responseText`）。页面加载早于登录跳转完成时，部分请求可能不在列表里，必要时刷新页面再读。

## 前端事件、插件与助手
- **前端事件（调用插件）只把 `request_rely` 里的字段值发给服务端**：用 R3 给 `formEvents[].request.assignments` 新增主表字段参数时，必须同时把该字段 widgetName 加进 `request_rely`，否则插件收到空值（2026-10-04 其他出库 docNo）。事件指向的插件实例在 `request.plugin_id / plugin_agent_id / symbol`，复制应用后会仍指向原应用的插件实例。
- **插件函数参数模板**：开放平台保存函数走 `/open/owner/plugin/trigger/update {plugin_id, symbol, runtime, source_code, arguments_template | return_template, key, iv}`，每次只带当前页那一份模板，但服务端会把另一份按页面缓存一起覆盖——要改两份时拦截一次保存、同时写入 `arguments_template` 与 `return_template`；读回用 `/open/owner/plugin/trigger/list {plugin_id}`。导入包建出的插件缺子表单入参 / 对象数组出参，可照此从同构插件复制模板。
- R3 提交表单时**每次**都要先把成员 / 部门默认值转 ID（见上），否则「当前用户」类默认值被清空——只改 formEvents 也一样。
- **关联查询字段（`linkquery`）结构**：`{type:'linkquery', linkForm:目标表单ID(本表单为 100000000000000000000001), linkFilter:[{link:目标字段, method:'eq', type:'widget', rely:本表字段, fixed:null, linkType:目标字段类型}, {link:目标字段, method:'eq', type:'fixed', rely:null, fixed:{value:'生效', type:'combo'}, linkType:'combo'}], linkFilterRel:'and', linkFields:[{text,type,name}], multi:true, allowAdd:false, rely:{widgets:[本表依赖字段]}, sort:null}`；item 层 `{description,label,labelHidden,lineWidth:12}`。R3 可直接新增，下拉固定值过滤已实测生效（2026-10-04）。流程表单新增字段默认各节点不可见，要在流程节点 optAuth 里加。
- **空跑拦截写请求要按路径末段匹配**：开放 API 批量新建是 `/v5/app/entry/data/batch_create`，`"/create" in path` 拦不住（2026-10-05 因此在生产误建 3 个批次）。用 `any(k in path.rsplit("/",1)[-1] for k in ("create","update","delete"))`。
- **助手生成 / 修改 / 删除的数据默认不触发其他助手**：要在生成数据的助手里设「联动触发 → 可触发指定智能助手 Pro」（`trigger_other {type:1, process_ids}`；type 2 + 空列表 = 不触发）。`trigger_other` 里留着已删除助手的 ID 时，保存报 100078「Linkage trigger configuration is illegal」，去掉该 ID 即可。
