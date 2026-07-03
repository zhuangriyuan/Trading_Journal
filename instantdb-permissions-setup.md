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


