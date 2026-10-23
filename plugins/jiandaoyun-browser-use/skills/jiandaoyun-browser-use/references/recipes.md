# 页面脚本（在内置浏览器的简道云页面里用 javascript_tool 执行）

每次调用都带自己标签页的 `tabId`。脚本按 REPL 语义执行：顶层 `await` 可用，最后一个表达式的值就是返回结果（不写 `return`）。脚本里的 `APP` / `E` / `SRC` / `DST` 等常量在同一次执行的开头自己定义。

所有脚本开头的公共片段：

```js
const tok=document.querySelector('meta[name=csrf-token]').content;
const H={'content-type':'application/json','x-csrf-token':tok};
const post=async(u,b)=>await (await fetch(u,{method:'POST',headers:H,body:JSON.stringify(b||{})})).json();
// 遍历表单定义里的所有字段（含子表单字段）
const walkWidgets=(content,fn)=>{ (function wk(x,sub){ if(Array.isArray(x)) return x.forEach(y=>wk(y,sub)); if(x&&typeof x==='object'){ if(x.widget&&x.widget.widgetName&&typeof x.label==='string'){ fn(x,sub); if(x.widget.type==='subform'){ (x.widget.items||[]).forEach(c=>wk(c,x)); return; } } Object.values(x).forEach(y=>wk(y,sub)); } })(content.items,null); };
// 逐键差异
const diff=(a,b)=>{ const d=[]; (function df(x,y,p){ if(JSON.stringify(x)===JSON.stringify(y)) return; if(x&&y&&typeof x==='object'&&typeof y==='object') new Set([...Object.keys(x),...Object.keys(y)]).forEach(k=>df(x[k],y[k],p+'.'+k)); else d.push(p); })(a,b,''); return d; };
```

页面需在简道云域内（任一应用后台 / 设计器页面即可）。改哪个应用由请求里的 `{app}` 决定，与当前页面属于哪个应用无关。

---

## R0 就绪检查 + 应用列表（在工作台 `/dashboard` 执行）

按 SKILL.md「打开内置浏览器」开好自己的标签页、`navigate` 到 `https://www.jiandaoyun.com/dashboard` 后运行。`host` 是 `portal.finecloud.com` 说明没登录，请用户在 Browser 面板里登录后重跑。

```js
for(let i=0;i<20&&!document.querySelector('a[href*="#/app/"]');i++) await new Promise(r=>setTimeout(r,500));
const apps={};
document.querySelectorAll('a[href*="#/app/"]').forEach(a=>{ const id=(a.getAttribute('href').match(/app\/([0-9a-f]{24})/)||[])[1]; const n=a.innerText.replace(/\u200b/g,'').trim().split('\n')[0]; if(id&&!apps[id]) apps[id]=n; });
window.__apps=apps;
({host:location.host, path:location.pathname, hasCsrf:!!document.querySelector('meta[name=csrf-token]'), apps})
```

刷新后还要复用的脚本存进 `sessionStorage`，用 `AsyncFunction` 取出执行（函数体里要写 `return`）：

```js
sessionStorage.setItem('jdy_probe',"return {host:location.host, csrf:!!document.querySelector('meta[name=csrf-token]')};");
const AsyncFunction=Object.getPrototypeOf(async function(){}).constructor;
await new AsyncFunction(sessionStorage.getItem('jdy_probe'))()
```

## R1 读取表单字段与节点权限（只读复核）

```js
const d=await post(`/_/admin/app/${APP}/form/${E}/edit`,{});
const lab={}; walkWidgets(d.content,(x,sub)=>{ lab[x.widget.widgetName]=(sub?sub.label+'.':'')+x.label; });
const w=await post(`/_/admin/app/${APP}/form/${E}/workflows`,{});
w.flows.filter(n=>n.flowId>=0).sort((a,b)=>a.flowId-b.flowId).map(n=>{
  const ed=[],ro=[]; for(const [k,v] of Object.entries(n.optAuth||{})){ const L=lab[k]||lab[k.split('.').pop()]||k; if(v&2) ed.push(L); else if(v&1) ro.push(L); }
  return n.flowId+' '+n.name+' 负责人:'+((n.chargers||{}).roles||[]).map(r=>r.name).join('/')+' 编辑['+ed+'] 只读['+ro+']'; }).join('\n')
```

字段联动 / 数据带出：`widget.rely`（`{filter, data:{formId, field}, subLink}`）、关联数据字段的 `widget.linkDataMaps` / `linkFields` / `linkFilter`。

## R2 批量改字段名（设计器保存拦截，最安全）

1. 打开设计器 `…/dashboard/app/{app}/form/{entry}/edit#/edit`（URL 不带 `#/app`）。
2. 安装拦截（`MAP=[[widgetName, 旧名, 新名]]`）：

```js
window.__map=MAP.map(([w,o,n])=>({w,o,n})); window.__log=[];
const _op=XMLHttpRequest.prototype.open,_sd=XMLHttpRequest.prototype.send;
XMLHttpRequest.prototype.open=function(m,u){this.__u=u;return _op.apply(this,arguments)};
XMLHttpRequest.prototype.send=function(b){ const u=(this.__u||'').split('?')[0];
  if(window.__armed && /\/form\/[0-9a-f]{24}\/update$/.test(u)){ try{ const j=JSON.parse(b); const m=Object.fromEntries(window.__map.map(x=>[x.w,x])); const seen={};
    (function wk(o){ if(Array.isArray(o)) return o.forEach(wk); if(o&&typeof o==='object'){ const wn=o.widget&&o.widget.widgetName; if(wn&&m[wn]&&typeof o.label==='string'){ if(o.label===m[wn].o){o.label=m[wn].n;seen[wn]=1;} else if(o.label===m[wn].n) seen[wn]=1; else seen[wn]='bad:'+o.label; } Object.values(o).forEach(wk);} })(j);
    const bad=window.__map.filter(x=>seen[x.w]!==1); if(bad.length) throw new Error('guard '+JSON.stringify(bad.map(x=>x.w)));
    window.__log.push({changed:window.__map.length}); window.__armed=false; b=JSON.stringify(j);
  }catch(e){ window.__log.push({blocked:String(e)}); window.__armed=false; this.abort(); return; } }
  return _sd.call(this,b); };
window.__armed=true;
```

3. 先读基线（`edit`），再 `button.click()` 点「保存」（按钮文字可能带零宽字符：`textContent.replace(/​/g,'').trim()==='保存'`）。
4. 重读 `edit` 与基线 `diff`：只允许 `.label` 变化及设计器补默认值（`mask`、`supportUnmaskedView`、`preSubmitConfirm`、`preSubmitActions`、`formEvents.*.trigger_type`、`quick_fill`、`fieldGroupConfigs`、`submitRule` 等）。
5. 保存后刷新设计器，否则旧标签会在下次保存时被写回。

## R3 直接提交表单定义（加字段 / 改下拉选项；设计器无改动时「保存」禁用）

```js
const A=await post(`/_/admin/app/${APP}/form/${E}/edit`,{}); const c=JSON.parse(JSON.stringify(A.content));
// 例：加单行文本 / 数字 / 下拉（widgetName 可指定，以便与另一应用保持同 ID）
const W='_widget_'+Date.now();
c.items.push({description:'',label:'新字段',labelHidden:false,lineWidth:12,widget:{visible:true,enable:true,allowBlank:true,type:'text',widgetName:W,mask:false,supportUnmaskedView:false}});
c.field_layout.push(W);
// 数字：type:'number', displayMode:'number', precision:null, thousandsSeparator:false, maxNumber:null, minNumber:null
// 下拉：type:'combo', colorEnable:false, items:[{value:'A',text:'A'}]
// ⚠ 成员 / 部门默认值转 ID 字符串，否则「当前用户」默认值会被丢弃（见 gotchas）
const walk=(items,fn)=>items.forEach(i=>{if(!i.widget)return;fn(i);if(i.widget.type==='subform')walk(i.widget.items||[],fn);});
walk(c.items,i=>{const v=i.widget.value;if(['user','dept','usergroup','deptgroup'].includes(i.widget.type)&&v!==undefined)i.widget.value=Array.isArray(v)?v.map(x=>x&&x._id||x):(v&&v._id?v._id:v);});
const a=A.attr||{};
const r=await fetch(`/_/admin/app/${APP}/form/${E}/update`,{method:'POST',headers:H,body:JSON.stringify({content:c,hasCache:!!a.hasCache,hasCustomWidget:false,submit_btn:a.submit_btn||{},draft_btn:a.draft_btn||{},mobileLayout:a.mobileLayout,pcLayout:a.pcLayout,formEvents:a.formEvents||[],syncFields:A.syncFields||{}})});
const B=await post(`/_/admin/app/${APP}/form/${E}/edit`,{});
// 核对：新字段存在；A 中原有字段在 B 中 JSON 完全一致
```

## R4 聚合表维度 / 指标改名

打开 `…/aggregate_table/{entry}/edit`，拦截 `/_/admin/app/aggregate_view/update`，在 `body.view.dimension[] / metric[]` 里按 `widgetName` 且 `name===旧名` 改 `name`，任一不符则 abort；点保存；重读 `aggregate_view/edit` 对比（只应有 name 差异）。不要碰 `transmission`。

## R5 读聚合表配置为可读文本（只读）

`aggregate_view/edit` → 对每个 `metric`：`formula` 中的 `$_aggregation_x#` 用 `aggVars[x]` 替换为 `op(表单.字段 若 过滤)`（`aggVars[x].source` 前 24 位为表单 ID，`field.subfield‖field.field` 为字段）；`$_widget_x#` 替换为引用的其他指标名。源过滤在 `view.source[i].filter.cond`。

## R6 跨应用同步表单定义（保留目标应用关联对象）

```js
const p=await post(`/_/admin/app/${SRC}/form/${E}/edit`,{}), t=await post(`/_/admin/app/${DST}/form/${E}/edit`,{});
const rel={}, relId={}; (function wk(x){ if(Array.isArray(x)) return x.forEach(wk); if(x&&typeof x==='object'){ if(x.widgetName&&x.relationship!==undefined) rel[x.widgetName]=x.relationship; if(x.aggVars) for(const [k,a] of Object.entries(x.aggVars)) if(a&&a.relationshipId!==undefined) relId[k]=a.relationshipId; Object.values(x).forEach(wk);} })(t.content);
const c=JSON.parse(JSON.stringify(p.content)); const missing=[];
(function wk(x){ if(Array.isArray(x)) return x.forEach(wk); if(x&&typeof x==='object'){ if(x.widgetName&&x.relationship!==undefined){ if(rel[x.widgetName]!==undefined) x.relationship=rel[x.widgetName]; else missing.push(x.widgetName); } if(x.aggVars) for(const [k,a] of Object.entries(x.aggVars)) if(a&&a.relationshipId!==undefined){ if(relId[k]!==undefined) a.relationshipId=relId[k]; else missing.push(k); } Object.values(x).forEach(wk);} })(c);
// missing 非空 → 停止（目标应用缺关联对象，不能复制）
// 然后按 R3 的格式提交到 DST；比较时用 JSON.parse(..., (k,v)=>(k==='relationship'||k==='relationshipId')?'<rel>':v) 归一化
```

先全量对比找出真正有差异的表（大量请求用后台异步 + 轮询），只同步需要的表；不要把领先的一方反向覆盖。

## R7 跨应用同步流程（新建版本 → 写入 → 启用）

```js
const KEYS='name,parents,path,flowId,x,y,type,optAuth,approvalMethod,approvalType,autoApproveRule,chargers,ccUsers,comment,signature,allowSubmit,submit_text,allowSave,save_text,allowPrint,print_text,allowClose,close_text,allowBatchForward,back_text,backNodes,backRule,backFlowType,transfer_text,candidates,validator,schedule,sign,reject,allowSavePrint,save_print_text,printIds,noChargersRule,pickChargers,lines,condition,dataInMaps,dataOutMaps,pluginOutMappings,revoke,draft_text,allowDraft'.split(',');
const ids=x=>{ if(!x||typeof x!=='object') return x; const o=Object.assign({},x); for(const k of ['roles','departs']) if(Array.isArray(o[k])) o[k]=o[k].map(v=>v&&typeof v==='object'?(v._id||v.id):v); if(Array.isArray(o.users)) o.users=o.users.map(v=>v&&typeof v==='object'?(v._id||v.username):v); return o; };
let d=await post(`/_/admin/app/${DST}/form/${E}/edit`,{}); let ver=(d.attr.flow.designVer||[]).slice(-1)[0];
if(!ver){ await post(`/_/admin/app/${DST}/form/${E}/workflow/create`,{}); d=await post(`/_/admin/app/${DST}/form/${E}/edit`,{}); ver=(d.attr.flow.designVer||[]).slice(-1)[0]; }
const pw=await post(`/_/admin/app/${SRC}/form/${E}/workflows`,{}), tv=await post(`/_/admin/app/${DST}/form/${E}/workflows`,{version:ver});
const tId={}; tv.flows.forEach(n=>tId[n.flowId]=n._id);
const flows=pw.flows.map(n=>{ const o={}; KEYS.forEach(k=>{ if(n[k]!==undefined) o[k]=n[k]; }); o.chargers=ids(o.chargers); o.ccUsers=ids(o.ccUsers); if(o.candidates) o.candidates=ids(o.candidates); if(o.sign&&o.sign.candidates) o.sign=Object.assign({},o.sign,{candidates:ids(o.sign.candidates)}); o.appId=DST; o.entryId=E; o.version=ver; if(tId[n.flowId]) o._id=tId[n.flowId]; return o; });
const r=await fetch(`/_/admin/app/${DST}/form/${E}/workflow/update`,{method:'POST',headers:H,body:JSON.stringify({flows,version:ver,theme:pw.theme||1,flowFormViews:pw.flowFormViews||[],inuse:true})});
// 核对：两边 workflows 按 flowId 比较 name / parents / 角色名 / optAuth / backNodes / allowClose
```

先 `inuse:false` 写入核对、确认后再 `inuse:true` 启用更稳。同应用改节点（如加节点提交校验）：源 = 本应用已启用版本，改目标节点后写入**新建**的设计版本再启用（先 `workflow/create` 取新增的那个版本号；不要复用 `designVer` 里已有的版本——那可能是别人未发布的草稿，生产 2026-10-04 三张表都有）；启用前先存一份 `workflows` 快照，启用后按 flowId 逐键（键排序后）比较。报 3005 且 `meta.flowId=N` 时，逐个字段替换该节点定位问题键。

## R8 新建数据推送（需用户授权）

```js
await post('/_/admin/workflow/webhook/create',{appId:APP,entryId:E,webhook:{mode:'custom',url:HOOK_URL,triggers:['process_update'],type:'flow',name:'未命名数据推送',remark:'',secret:''}});
// 核对：/_/admin/app/webhook/list {appId,limit:100,skip:0} 中该表 enable:true、url 指向正确助手
```

HOOK_URL 指向本应用自己的 HTTP 助手（地址规律见 endpoints）。用 `/_/admin/workflow/webhook/logs` 查成功 / 失败日志。

## R9 启用 / 停用智能助手（需用户授权）

```js
await post('/automation/apis/process/enable',{appId:APP,process_id:PID,enable:true});
```

## R10 读业务数据（管理端，只读）

1. 打开表单数据管理页 `…/form/{entry}/edit#/data`，拦截一次 `data_process/admin/data/find` 拿到完整 `fields` 结构。
2. 筛出需要的字段（子表单可裁剪 `items`），`limit:100` 按 `skip` 翻页。

## R11 仪表盘标题同步

`dash/{id}/edit` → 遍历组件找 `{name:widget, form, title}`，按 `(form 包含 entryId, name)` 且 `title===旧名` 改 `title` → `dash/{id}/update {components, styles, mobileStyles}` → 重读对比只应有 `.title` 差异。

## R12 聚合表加数据源 / 指标 / 改公式（明文提交）

```js
const v=(await post('/_/admin/app/aggregate_view/edit',{appId:APP,entryId:AGG})).view; // 基线，深拷贝后修改
const key=`${E}@${APP}#${SUB}`;                                   // 数据源键：表单@应用#子表单
v.source.push({type:'sub_form',appId:APP,entryId:E,subform:SUB,filter:{rel:'and',cond:[{type:'text',method:'in',value:['流转完成'],hasEmpty:false,entryId:E,field:FLOW_STATE}]}});
v.dimension.forEach(d=>d.union[key]={type:'sub_field',subform:SUB,subfield:MAP[d.name],fieldType:'text'}); // 主表字段用 {type:'main_field',field,fieldType}
const a='_aggregation_'+Date.now(), w='_widget_'+(Date.now()+1);
v.metric.push({widgetName:w,name:'新指标',formula:`$${a}#`,aggVars:{[a]:{op:'SUM',field:{type:'sub_field',subform:SUB,subfield:QTY},filter:{rel:'and',cond:[]},source:key}},remark:'',format:{type:'number',precision:null,thousandsSeparator:false}});
v.metric.find(m=>m.name==='库存数量').formula+=`+$${w}#`;          // 引用其他指标用 $_widget_x#
// 提交前再读一次，与基线全等才提交；不带 transmission
await post('/_/admin/app/aggregate_view/update',{appId:APP,entryId:AGG,configChanged:true,view:v});
// 回读 edit，比对 source / dimension / metric 与提交一致
// 返回 needInit:true 时：await post('/_/admin/app/aggregate_view/calculate',{appId:APP,entryId:AGG}); 再按 list.dataModifyTime / data/find 确认已重算
```

先用表单 `edit` 核对每个字段 ID 的标签与类型；新指标、聚合变量 ID 保证唯一。改完刷新设计器确认能打开。

## R13 给下拉框加选项（设计器保存拦截）
下拉 / 单选的 `widget.items` 是明文 `[{value,text,color}, …, {text:'其他',value:'其他',isOther:true}]`，可在保存请求里直接改（不像公式走加密）。做法：设计器页面拦截 `/form/{entry}/update` → 先读 `/edit` 存基线 → JS 派发事件选中该字段、把标题临时改成「原名x」让设计器变「已修改」→ 点保存；拦截器里校验标题为「原名x / 原名」且 items 与基线全等后，把标题还原并在 `isOther` 项前 `splice` 新选项；校验不过只还原标题照常发送（**不要 abort**）。保存后重读 `/edit` 与基线 diff，应只有 `widget.items` 变化。2026-10-04 其他入库「冲销入库」、其他出库「冲销出库」测试 + 生产均按此完成。

## R14 只读自检（换环境 / 换账号 / 换浏览器后先跑一遍）

逐个调用 endpoints 里的读取类接口，每项返回 `ok` / `skip 原因` / `FAIL 原因`。只读，不输出令牌、apiKey、推送地址。开头定义 `APP`（R0 里的应用 ID，选表单、仪表盘、聚合表、助手较多的应用）；对象较多时在 45 秒内跑不完，就按 SKILL.md 放后台跑。

```js
const tok=document.querySelector('meta[name=csrf-token]').content;
const H={'content-type':'application/json','x-csrf-token':tok};
const call=async(u,b)=>{ const r=await fetch(u,{method:'POST',headers:H,body:JSON.stringify(b||{})}); let j=null; try{ j=await r.json(); }catch(e){} return {s:r.status,j}; };
const need=(r,pick)=>{ const v=r.s===200&&r.j?pick(r.j):undefined; if(v===undefined||v===null||v===false) throw new Error('HTTP '+r.s+' '+JSON.stringify(r.j).slice(0,100)); return v; };
const arr=j=>Array.isArray(j)?j:(Object.values(j||{}).find(Array.isArray)||[]);
const R={}, C={};
const T=async(k,fn)=>{ try{ const v=await fn(); R[k]='ok'+(v?' '+v:''); }catch(e){ R[k]='FAIL '+String(e.message||e).slice(0,120); } };
const skip=(k,why)=>{ R[k]='skip '+why; };

// 应用 / 菜单
const forms=[], dashes=[];
await T('menu', async()=>{ const map=need(await call(`/_/app/${APP}/menu`), j=>j.entryMap);
  Object.values(map).forEach(x=>{ const id=x.entryId||x._id; if(x.name&&id&&(x.type==='form'||x.type==='dash')) (x.type==='form'?forms:dashes).push({id,name:x.name}); });
  return `表单${forms.length} 仪表盘${dashes.length}`; });
await T('info_get', async()=>{ need(await call(`/_/app/${APP}/info_get`), j=>j); });

// 表单 / 流程
let flowForm=null;
await T('form/edit', async()=>{ if(!forms.length) throw new Error('应用内没有表单');
  for(const f of forms.slice(0,30)){ const d=need(await call(`/_/admin/app/${APP}/form/${f.id}/edit`), j=>j.content&&j);
    if(!C.form){ C.form=f; C.items=d.content.items.length; C.widgets=d.content.items.filter(i=>i.widget&&i.widget.widgetName&&i.widget.type!=='subform'&&i.widget.type!=='separator').slice(0,3).map(i=>({name:i.widget.widgetName,type:i.widget.type,title:i.label})); }
    if(d.attr&&d.attr.hasFlow){ flowForm={...f, designVer:(d.attr.flow&&d.attr.flow.designVer)||[]}; break; } }
  return `${C.form.name}（顶层控件${C.items}）`+(flowForm?` 流程表单:${flowForm.name}`:''); });
if(flowForm){
  await T('form/workflows', async()=>{ const w=need(await call(`/_/admin/app/${APP}/form/${flowForm.id}/workflows`), j=>j.flows); return `${flowForm.name} 节点${w.filter(n=>n.flowId>=0).length}`; });
  const v=flowForm.designVer.slice(-1)[0];
  if(v!==undefined) await T('form/workflows{version}', async()=>{ const w=need(await call(`/_/admin/app/${APP}/form/${flowForm.id}/workflows`,{version:String(v)}), j=>j.flows); return `设计版本${v} 节点${w.length}`; });
  else skip('form/workflows{version}','没有设计中的流程版本');
} else { skip('form/workflows','前 30 张表单里没有流程表单'); skip('form/workflows{version}','同上'); }
if(C.form) await T('webhook/form_item/get', async()=>{ need(await call(`/_/admin/app/${APP}/entry/${C.form.id}/webhook/form_item/get`), j=>j); });

// 业务数据
if(C.form){
  await T('data/count', async()=>{ const j=need(await call('/_/data_process/admin/data/count',{appId:APP,entryId:C.form.id,filter:{rel:'and',cond:[]}}), j=>j); return JSON.stringify(j).slice(0,40); });
  await T('data/find', async()=>{ const j=need(await call('/_/data_process/admin/data/find',{appId:APP,entryId:C.form.id,filter:{rel:'and',cond:[]},skip:0,limit:1,fields:C.widgets}), j=>j); return `返回键 ${Object.keys(j).slice(0,5).join(',')}`; });
}

// 仪表盘
if(dashes.length) await T('dash/edit', async()=>{ const j=need(await call(`/_/admin/app/${APP}/dash/${dashes[0].id}/edit`), j=>j); return `${dashes[0].name} 组件${Object.keys(j.components||{}).length}`; });
else skip('dash/edit','应用内没有仪表盘');

// 聚合表
let aggs=[];
await T('aggregate_view/list', async()=>{ aggs=arr(need(await call('/_/admin/app/aggregate_view/list',{appId:APP}), j=>j)); return `聚合表${aggs.length}`; });
if(aggs.length) await T('aggregate_view/edit', async()=>{ const a=aggs[0]; const v=need(await call('/_/admin/app/aggregate_view/edit',{appId:APP,entryId:a.entryId||a._id}), j=>j.view); const vars=(v.metric||[]).reduce((n,m)=>n+Object.keys(m.aggVars||{}).length,0); return `${a.name||''} 数据源${(v.source||[]).length} 维度${(v.dimension||[]).length} 指标${(v.metric||[]).length} 聚合变量${vars}`; });
else skip('aggregate_view/edit','应用内没有聚合表');

// 智能助手
let procs=[];
await T('process/list', async()=>{ procs=need(await call('/automation/apis/process/list',{appId:APP}), j=>j.process_list); return `助手${procs.length}`; });
if(procs.length){ const pid=procs[0].process_id;
  await T('process/get', async()=>{ need(await call('/automation/apis/process/get',{appId:APP,process_id:pid}), j=>j); });
  await T('process/get_config', async()=>{ const f=need(await call('/automation/apis/process/get_config',{appId:APP,process_id:pid}), j=>j.flows||(j.data&&j.data.flows)); return `节点${f.length}`; });
} else { skip('process/get','应用内没有助手'); skip('process/get_config','同上'); }

// 插件实例（不读 /open/owner/agent/get：里面有 apiKey 明文）
let agents=[];
await T('agent/list', async()=>{ agents=arr(need(await call('/open/manage/agent/list',{appId:APP}), j=>j)); return `插件实例${agents.length}`; });
if(agents.length) await T('agent/list_trigger', async()=>{ const a=agents[0]; const j=need(await call('/open/manage/agent/list_trigger',{appId:APP,agent_id:a.agent_id||a._id||a.id,has_applets:false}), j=>j); return `函数${arr(j).length}`; });
else skip('agent/list_trigger','应用内没有插件实例');

// 数据推送
let hooks=[];
await T('webhook/list', async()=>{ hooks=arr(need(await call('/_/admin/app/webhook/list',{appId:APP,limit:100,skip:0}), j=>j)); return `推送${hooks.length}`; });
if(hooks.length) await T('webhook/logs', async()=>{ const h=hooks[0]; need(await call('/_/admin/workflow/webhook/logs',{status:'success',limit:5,skip:0,appId:APP,entryId:h.entryId,webhookId:h._id||h.webhookId||h.id}), j=>j); });
else skip('webhook/logs','应用内没有数据推送');

// 数据工厂
let etls=[];
await T('list_etl', async()=>{ etls=need(await call(`/_/admin/app/${APP}/list_etl`), j=>j.etlList); return `数据工厂${etls.length}`; });
if(etls.length) await T('etl/edit', async()=>{ const e=etls[0]; need(await call(`/_/admin/app/${APP}/etl/${e._id||e.etlId||e.id}/edit`), j=>j); return e.name||''; });
else skip('etl/edit','应用内没有数据工厂');

({pass:!Object.values(R).some(v=>v.startsWith('FAIL')), results:R})
```
