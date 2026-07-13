把下面这段 JSON 粘贴/合并到 InstantDB Dashboard → 你的 App → Permissions 里。

打开方式：https://instantdb.com/dash → 选择你的 App（APP_ID: c39b06a1-34be-46b6-9d7e-f306be9b7682）→ 左侧 Permissions。

⚠️ 如果你之前已经写过别的权限规则，不要整段覆盖 —— 把 accounts 和 records 这两个 key 合并进你现有的规则里即可，其余 namespace（比如 $users、$files）保持不变。

{
  "accounts": {
    "bind": [
      "isOwner", "auth.id != null && auth.id == data.ownerId"
    ],
    "allow": {
      "view": "isOwner || (ruleParams.shareToken != null && ruleParams.shareToken == data.shareToken)",
      "create": "auth.id != null",
      "update": "isOwner",
      "delete": "isOwner"
    }
  },
  "records": {
    "bind": [
      "isOwner", "auth.id != null && auth.id == data.ownerId"
    ],
    "allow": {
      "view": "isOwner || (ruleParams.shareToken != null && ruleParams.shareToken == data.shareToken)",
      "create": "auth.id != null && auth.id == data.ownerId",
      "update": "isOwner",
      "delete": "isOwner"
    }
  }
}

## 这段规则做了什么

- accounts / records 的 view（读取）权限：账户主人（ownerId 匹配）永远可以看；
  或者，如果请求带着正确的 shareToken（也就是分享链接里 ?share= 后面那串），也能看 —— 但仅限那一个账户和它名下打了同样 shareToken 的记录。
- create / update / delete 权限：只有 ownerId 匹配的本人才能改 —— 分享出去的人即使拿到链接，也无法新增、修改或删除任何数据。

## 前端这边我已经做的事

1. index.html 里加了「🔗 只读分享」按钮（在账户工具栏，退出登录旁边）。点击后会：
   - 给当前账户生成一个随机 shareToken，并把它也写到这个账户名下的每一条记录上（因为权限规则是逐条记录判断的，没有用到 InstantDB 的 schema 关联/links，走的是最简单的"每条数据自己带一个令牌"的方式）。
   - 显示可复制的链接，格式是：你的网址 + ?share=xxxxxxxx
2. 之后你新增的交易记录，会自动带上当前账户的 shareToken（如果这个账户开着分享的话），所以对方能实时看到新数据，不需要你重新分享。
3. 任何人打开这个带 ?share= 的链接，会直接进入一个简化的"只读视图"（不需要登录），按月份列出交易和入金明细、当月盈亏汇总 —— 但看不到、也碰不到你的账户切换、删除、导入导出等功能。
4. 想撤回分享，点开分享面板里的「取消分享」——旧链接会立刻失效（token 被清空）。

## 需要你手动做的一步

前端代码已经写好了，但 **权限规则必须去 InstantDB 网站后台配置**，我这边没有权限帮你直接改（需要你的账号登录）。请把上面的 JSON 粘贴进去，保存后分享功能才会真正生效 —— 在那之前，点"生成只读链接"仍然可以工作（因为你是 owner），但陌生人打开链接会看到"无法读取"，因为规则还没放开。

## 一点点技术上的保留

`ruleParams` 在只读链接页面里的具体调用方式（`db.subscribeQuery(query, callback, { ruleParams })`）是我参照 InstantDB 官方文档拼出来的，但官方文档目前只给了 React 版 (`db.useQuery(query, { ruleParams })`) 的例子，没有专门给纯 JS/CDN 版本的例子，我没法在这个沙盒里连真实网络测试。如果你打开分享链接后浏览器控制台报错（比如提示 "ruleParams... is not a function" 之类），大概率是这个调用顺序需要调整——去 https://www.instantdb.com/docs/permissions 的 ruleParams 章节核对一下，或者告诉我报错信息，我再帮你改。

## IBKR 持仓跨设备同步（ibkrSnapshots 表）

装了 IBKR Gateway 的那台电脑每次刷新持仓后，会把数据顺手存一份到 InstantDB 的 `ibkrSnapshots` 这张表里，其他设备（手机、部署到网上的版本）直接读这张表里"上一次同步"的数据，不需要自己连 Gateway。

同样因为 `attrs` 被锁住了，第一次用之前要去 InstantDB 后台的 Explorer 里手动给 `ibkrSnapshots` 建好这张表和下面这几个字段：
- `ownerId`（string）
- `accountId`（string）
- `positions`（string，存的是持仓数组序列化后的 JSON 文本）
- `updatedAt`（number）

权限规则（合并进 Permissions 里，跟 `accounts`/`records` 平级）：

```json
"ibkrSnapshots": {
  "bind": [
    "isOwner", "auth.id != null && auth.id == data.ownerId"
  ],
  "allow": {
    "view": "isOwner",
    "create": "auth.id != null && auth.id == data.ownerId",
    "update": "isOwner",
    "delete": "isOwner"
  }
}
```

这张表只有你自己能看，没有做成"只读分享"能看到的内容（IBKR 面板本身在只读分享视图里也是隐藏的）。

## 日历笔记（dayNotes 表）

日历里点开某一天的详情弹窗，现在可以给这一天写复盘笔记（富文本，支持加粗/斜体/下划线/列表，不支持图片）。

同样因为 `attrs` 被锁住了，第一次用之前要去 Explorer 手动建好 `dayNotes` 这张表和下面几个字段：
- `ownerId`（string）
- `accountId`（string）
- `date`（string，格式 `YYYY-MM-DD`，跟 `records` 表的日期格式一致）
- `content`（string，存的是富文本编辑器输出的 HTML）
- `updatedAt`（number）
- `shareToken`（string，非必填——只读分享账户时，对方也能看到这一天的笔记，但不能编辑）

权限规则（合并进 Permissions 里，跟 `accounts`/`records` 平级）：

```json
"dayNotes": {
  "bind": [
    "isOwner", "auth.id != null && auth.id == data.ownerId"
  ],
  "allow": {
    "view": "isOwner || (ruleParams.shareToken != null && ruleParams.shareToken == data.shareToken)",
    "create": "auth.id != null && auth.id == data.ownerId",
    "update": "isOwner",
    "delete": "isOwner"
  }
}
```

跟只读分享链接的权限规则思路完全一样：账户主人随时能看能改；分享链接对方能看，但改不了。

## 交易的入场价 / 止盈价 / 止损价

新增了三个可选字段（不需要改权限规则，`records` 表本来就有 view/create/update/delete 规则了，只是又多了几个新字段，同样受 `attrs` 锁的影响）：

- `records.entryPrice`（number，非必填）
- `records.takeProfitPrice`（number，非必填）
- `records.stopLossPrice`（number，非必填）

跟之前 `direction`/`shares` 一样，第一次用之前要么去 Explorer 手动建好这三个字段，要么用"改 attrs 为 true → 保存一笔带这三个字段的交易 → 改回 false"的方法（见上面"快速建表步骤"）。

## 日历图片链接（dayNotes.images）

日历详情弹窗的笔记下面新增了"图片链接"区域，可以贴图床的直链进去，不需要改权限规则（跟笔记正文用的是同一张 `dayNotes` 表），但需要新增一个字段：

- `dayNotes.images`（string，非必填——存的是图片链接数组序列化后的 JSON 文本，跟 `content` 字段是同样的存储方式）

老规矩：第一次用之前，要么去 Explorer 手动建这个字段，要么用"改 attrs 为 true → 添加一张图片 → 改回 false"的方法（见上面"快速建表步骤"）。

图片本身不会存进 InstantDB，只是存了一个链接文字，图片文件还在你贴的那个图床（imgbb / postimages 等）上，所以不会对 InstantDB 的存储配额有什么实际影响。

## 图片上传（改用 ImgBB 直传）

原本"图片链接"功能升级成了真正的"从电脑/相册上传"——用的是 [ImgBB](https://api.imgbb.com/) 的免费上传接口，浏览器直接把图片传过去，拿到直链后存进 `dayNotes.images` 字段（跟之前一样，存的还是链接的 JSON 数组文本，只是现在每张图存的是 `{url, thumb}` 一对链接，`thumb` 是 ImgBB 自动生成的缩略图，加载更快）。

- 不需要改 InstantDB 的表/字段/权限规则，`dayNotes.images` 还是原来那个字段，存储格式只是从"字符串数组"变成"对象数组"，代码里做了兼容处理，老数据不会丢
- 需要你自己去 [api.imgbb.com](https://api.imgbb.com/) 免费申请一个 API Key，填进 App 里的"⚙️ 上传设置"，只存在你自己浏览器本地（`localStorage`），不会经过 InstantDB 也不会传给我
- 图片文件本身存在 ImgBB，不会占用 InstantDB 的存储配额

## ImgBB API Key 跨设备同步（userSettings 表）

图片上传的 API Key 现在会同步进 InstantDB，不再只存在单个浏览器里，需要新建一张表：

- 表名：`userSettings`
- 字段：`ownerId`（string）、`imgbbApiKey`（string，非必填）、`updatedAt`（number）

权限规则（合并进 Permissions，跟其他表平级）：

```json
"userSettings": {
  "bind": ["isOwner", "auth.id != null && auth.id == data.ownerId"],
  "allow": {
    "view": "isOwner",
    "create": "auth.id != null && auth.id == data.ownerId",
    "update": "isOwner",
    "delete": "isOwner"
  }
}
```

这张表只有你自己能看，不参与"只读分享"（别人打开你的分享链接看不到、也不需要用到这个 Key）。同样要走一遍"快速建表步骤"（改 attrs 为 true → 在"上传设置"里保存一次 Key → 改回 false，或者去 Explorer 手动建）。

## 进场评级（records.grade）

新增了一个可选字段，不需要改权限规则（`records` 表本来就有），只是多了个新字段：

- `records.grade`（string，非必填，取值 "A"/"B"/"C"）

老规矩：第一次用之前，要么去 Explorer 手动建这个字段，要么用"改 attrs 为 true → 保存一笔带评级的交易 → 改回 false"的方法。

## 交易规则（userSettings.tradingRules）

最右侧新增的"交易规则"面板，跟 ImgBB API Key 一样存在 `userSettings` 表里（同一张表，只是多一个字段），不区分账户，登录后所有账户共用同一份规则列表，不会出现在只读分享链接里。

- 字段：`userSettings.tradingRules`（string，非必填——存的是规则文字数组序列化后的 JSON 文本，跟笔记图片的存法一样）

如果你已经按前面步骤建过 `userSettings` 表（用来存 ImgBB Key），这次只需要**再加一个字段**：`tradingRules`（string，非必填）。同样可以用"改 attrs 为 true → 加一条规则 → 改回 false"的方法，或者去 Explorer 手动加。

（后续更新：交易规则现在支持分类了，存储的 JSON 结构从"文字数组"变成"分类数组"，字段名和类型都没变，还是同一个 `tradingRules` 字符串字段，不需要重新建字段。旧数据会在第一次加载时自动归到一个"未分类"分类下，不会丢。）
