# Trading Journal (多账户交易日志)

一个纯前端、无需后端服务器的交易日志 App。用邮箱验证码登录，数据实时同步到 [InstantDB](https://instantdb.com)（多设备共享同一份数据），支持多账户、日历盈亏视图、图表统计、只读分享链接，并可作为 PWA 添加到手机主屏幕，或用 Electron 打包成桌面 App。

## 功能

- **邮箱验证码登录**，数据云端实时同步（电脑/手机自动同步，不依赖本地存储）
- **多账户管理**：新建 / 切换 / 删除账户，每个账户的交易和入金记录互相独立
- **日历视图**：按月/按年查看每日盈亏，点击某天可查看当天所有交易明细；手机端有单独优化过的紧凑布局
- **统计面板**：总盈亏、总入金、胜率、Profit Factor、平均盈亏（按全局/当天/当周/当月拆分）
- **图表**：每日盈亏柱状图、累计盈亏曲线、每日入金柱状图
- **交易 / 入金历史表格**：搜索、排序、分页、批量删除
- **只读分享链接**：生成一个 `?share=TOKEN` 链接，对方无需登录即可查看你的账户（日历、统计、图表、历史记录都能看），但看不到、也碰不了任何编辑功能
- **导出 / 导入备份**：JSON 格式，跨设备迁移或本地留档
- **PWA**：支持"添加到主屏幕"，有独立图标和启动主题色
- **Electron 友好**：全局隐藏了滚动条，打包成桌面 App 后不会露出网页感的滚动条

## 技术栈

不需要任何构建工具，全部通过 CDN 引入，`index.html` 单文件即可运行：

- [Vue 3](https://vuejs.org/)（`vue.global.prod.min.js`）
- [Tailwind CSS](https://tailwindcss.com/)（CDN 版）
- [Chart.js](https://www.chartjs.org/)
- [InstantDB](https://instantdb.com)（`@instantdb/core`，实时数据库 + 邮箱验证码登录）

## 目录结构

```
.
├── index.html                       App 主文件（唯一必须的文件）
├── manifest.json                    PWA 配置（"添加到主屏幕"用）
├── icons/
│   ├── favicon-32.png               浏览器标签页图标
│   ├── icon-192.png                 PWA / Android 主屏幕图标
│   ├── icon-512.png                 PWA 高清图标 / 生成 logo-full.png 的素材
│   └── apple-touch-icon.png         iOS 主屏幕图标（180×180，不透明背景）
├── logo-full.png                    完整版 Logo（图形 + "TRADING JOURNAL" 字样），用于 README / 官网 / 启动页，不用于小尺寸图标
└── instantdb-permissions-setup.md   InstantDB 后台需要粘贴的权限规则说明
```

## 首次使用前的配置

### 1. 申请 InstantDB APP_ID

去 [instantdb.com/dash](https://instantdb.com/dash) 免费注册一个 App，拿到 `APP_ID`，替换掉 `index.html` 里的这一行：

```js
const APP_ID = 'xxxxxxxxx';
```

### 2. 配置权限规则（Permissions）

InstantDB 后台 → 你的 App → **Permissions**，把 `instantdb-permissions-setup.md` 里的规则粘贴进去（如果你已经写过别的规则，只需合并 `accounts` / `records` 这两段，不要整段覆盖）。

这段规则做了两件事：
- 保证只有账户本人（`ownerId` 匹配）能新增/修改/删除数据
- 允许"只读分享链接"里携带的 `shareToken` 通过校验后被匿名访问（但仍然是只读）

### 3.（可选）如果你锁了 schema

如果 InstantDB 权限里有 `"attrs": { "allow": { "$default": "false" } }` 这样锁死新字段创建的规则，第一次使用"只读分享"功能前，需要先去 Dashboard 的 **Explorer** 里手动给 `accounts` 和 `records` 两张表各加一个 `shareToken`（string，非必填）字段，否则分享功能会报权限错误。详见 `instantdb-permissions-setup.md`。

## 本地运行 / 部署

**本地**：直接双击打开 `index.html` 即可（`file://` 协议下大部分功能正常，但"只读分享链接"生成的地址会是 `file://...`，只有你自己电脑能打开）。

**部署到 GitHub Pages / 任意静态网站托管**：把整个文件夹（`index.html`、`manifest.json`、`icons/`、`logo-full.png`）原样上传，保持相对路径不变即可，不需要任何构建步骤。部署到真实域名后，"只读分享链接"会自动变成对方也能打开的 `https://...` 地址。

**打包成 Electron 桌面 App**：把 `index.html` 作为 `BrowserWindow` 加载的页面即可；应用图标（任务栏/Dock）需要在 Electron 的打包配置（如 `electron-builder` 的 `icon` 字段）里单独指定，`favicon`/`apple-touch-icon` 这些网页 meta 标签不会影响它。

