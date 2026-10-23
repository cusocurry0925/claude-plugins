# cusocurry0925-plugins

个人 Claude 插件市场。

## 插件

| 插件 | 说明 |
|------|------|
| `jiandaoyun-integration-guide` | 简道云对接配置指南（开放 API、Webhook、SSO、前端事件、自建插件、MCP） |
| `jiandaoyun-support` | 简道云技术专家（表单、流程、数据管理、权限、智能助手、问题排查） |
| `jiandaoyun-browser-use` | 简道云后台配置批量操作（在 Claude 桌面 App 内置浏览器里调网页端内部接口改字段、流程、聚合表、助手等） |

## 安装

在 Claude Code 中：

```
/plugin marketplace add cusocurry0925/claude-plugins
/plugin install jiandaoyun-integration-guide@cusocurry0925-plugins
/plugin install jiandaoyun-support@cusocurry0925-plugins
/plugin install jiandaoyun-browser-use@cusocurry0925-plugins
```

`jiandaoyun-browser-use` 原名 `jiandaoyun-chrome`（2026-10-23 改名，并从 Claude in Chrome 改为内置浏览器）。`marketplace.json` 里的 `renames` 会把已安装的旧名迁移到新名；如提示 `not cached`，执行一次 `/plugin install jiandaoyun-browser-use@cusocurry0925-plugins`。

## 来源与版权

`jiandaoyun-integration-guide` 技能内容来自 SkillHub：
<https://skillhub.cn/skills/@org-nznpea4p/jiandaoyun-integration-guide>（v1.0.0，作者：南京致和），
此处仅做插件格式封装，内容版权归原作者所有。

`jiandaoyun-support` 技能内容来自 SkillHub：
<https://skillhub.cn/skills/@org-nznpea4p/jiandaoyun-support>（v1.0.0，作者：南京致和），
同样仅做插件格式封装。

`scripts/jdy_api.py` 需要设置环境变量 `JDY_API_KEY` 才能调用简道云 API，请勿把密钥提交到仓库。

`jiandaoyun-browser-use` 为自研技能，需在 Claude 桌面 App 的内置浏览器（Browser 面板）里登录简道云后使用，在已登录的简道云页面内执行；CSRF 令牌只留在页面内存，apiKey 只放本机文件，请勿提交到仓库。
