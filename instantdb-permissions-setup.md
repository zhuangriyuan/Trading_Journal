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
