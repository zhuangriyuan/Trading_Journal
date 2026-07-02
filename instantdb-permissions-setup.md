把下面这段 JSON 粘贴/合并到 InstantDB Dashboard → 你的 App → Permissions 里。

打开方式：https://instantdb.com/dash → 选择你的 App（APP_ID: ）→ 左侧 Permissions。

⚠️ 如果你之前已经写过别的权限规则，不要整段覆盖 —— 把 accounts 和 records 这两个 key 合并进你现有的规则里即可，其余 namespace（比如 $users、$files）保持不变。

{
  "attrs": {
    "allow": {
      "$default": "false"
    }
  },
  "$users": {
    "allow": {
      "create": "data.email in ['xxx@gmail.com']"
    }
  },
  "records": {
    "bind": [
      "isOwner",
      "auth.id != null && auth.id == data.ownerId"
    ],
    "allow": {
      "view": "isOwner || (ruleParams.shareToken != null && ruleParams.shareToken == data.shareToken)",
      "create": "auth.id != null && auth.id == data.ownerId",
      "delete": "isOwner",
      "update": "isOwner"
    }
  },
  "accounts": {
    "bind": [
      "isOwner",
      "auth.id != null && auth.id == data.ownerId"
    ],
    "allow": {
      "view": "isOwner || (ruleParams.shareToken != null && ruleParams.shareToken == data.shareToken)",
      "create": "auth.id != null && auth.id == data.ownerId",
      "delete": "isOwner",
      "update": "isOwner"
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

InstantDB 网站后台配置。请把上面的 JSON 粘贴进去，保存后分享功能才会真正生效 —— 在那之前，点"生成只读链接"仍然可以工作（因为你是 owner），但陌生人打开链接会看到"无法读取"，因为规则还没放开。

