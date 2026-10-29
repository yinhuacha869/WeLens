# WeLens 使用说明

WeLens 是一个桌面小助手：常驻一个可拖动的悬浮球，让电脑上的 AI 助手（Claude Desktop、Cursor 等支持 MCP 的工具）能够**只读**查看本机微信的会话、联系人、聊天记录，并能查看聊天中的图片、表情、收听语音。

> WeLens 只能"看"，不能"发"——它不会替你发送任何微信消息，数据全部保存在本机，不会上传。

> **版本要求**：目前仅支持 **PC 微信 4.1.15.x 及以上版本**（微信主程序 `Weixin.exe`）。低于该版本的微信无法读取登录账号与聊天数据，升级微信后再使用。

## 快速开始（三步接入 AI）

1. **启动 WeLens**
   双击 WeLens.exe，桌面出现悬浮球，MCP 服务会自动启动。

2. **把 WeLens 添加到 AI 工具**
   单击悬浮球打开配置窗 → 在「接入 AI 客户端」区域点 **复制 MCP 配置** → 把复制到的内容按 AI 工具的要求粘贴进它的 MCP 设置里保存（例如 Claude Desktop 是 `claude_desktop_config.json`），然后重启 AI 工具。

3. **让 AI 学会使用**
   点 **复制 AI 使用提示词** → 粘贴到 AI 工具的对话框发送。之后就可以直接用自然语言提问了，比如：
   - "帮我看看最近有哪些未读会话"
   - "获取和某某的聊天记录"
   - "把昨天那张图片解出来给我看"

## 悬浮球

| 操作 | 效果 |
| --- | --- |
| 按住拖动 | 移动位置（位置会自动记住） |
| 左键单击 | 打开配置窗 |
| 右键 | 菜单：打开配置 / 启停服务 / 复制配置 / 隐藏 / 退出 |
| 鼠标悬停 | 球体轻微放大 |

**图标含义**

- 右下角小圆点 = MCP 服务状态：<br>
  🟢 绿色 运行中 ｜ 🟡 黄色 启动/停止中 ｜ 🔴 红色 未启动或失败
- 右上角数字 = 当前登录微信的未读消息总数（超过 99 显示 99+，无未读时隐藏；公众号等官方账号不计入）

## 配置窗说明

- **服务状态**：显示 MCP 服务运行状态、地址与运行时长。
- **服务配置**：MCP 端口（默认 4588）、密钥（可显示/复制/重新生成）。一般保持默认即可。
- **接入 AI 客户端**：服务地址、鉴权请求头、**复制 MCP 配置**、**复制 AI 使用提示词**。
- **启动设置**：是否随 WeLens 自动启动服务、是否开机自启。
- 底部：打开日志目录、退出 WeLens。

## MCP 工具列表

WeLens 向 AI 客户端提供 5 个只读工具。所有工具返回统一信封：成功为 `{ "ok": true, "data": ... }`，失败为 `{ "ok": false, "error": { "code", "message" } }`。

### 调用流程

`get_login_accounts` 取 `wx_id` → `get_sessions` 取 `session_id` → `get_messages` 取聊天记录（翻页用返回的 `next_cursor`）→ `decrypt_media` 取图片/语音/表情的本地文件。

### 1. get_login_accounts

获取本机微信登录账号列表（含历史登录账号），无需参数。返回 `items[].wxId / name / number`，其中 `wxId` 是其他所有工具的必填参数。

### 2. get_sessions

获取会话列表，按最后消息时间倒序（默认不含公众号）。

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `wx_id` | 是 | 登录用户微信 ID，来自 get_login_accounts |
| `limit` | 否 | 返回条数 1-100，默认 20 |
| `cursor` | 否 | 分页游标，来自上一次返回的 `next_cursor` |
| `updated_after` | 否 | Unix 秒，仅返回该时间后有新消息的会话；0 表示不限 |
| `include_official` | 否 | 是否包含公众号会话，默认 false |
| `keyword` | 否 | 按昵称、备注、微信号或会话标题搜索 |

返回项：`sessionId`、`name`、`type`（`single` 单聊 / `group` 群聊 / `official` 公众号）、`lastMessagePreview`、`lastMessageAt`（Unix 秒）、`unreadCount`。

### 3. get_contacts

获取联系人与群聊资料；`contactId` 即群聊的 `session_id`，可直接用于 get_messages。

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `wx_id` | 是 | 登录用户微信 ID |
| `limit` | 否 | 返回条数 1-100，默认 20 |
| `cursor` | 否 | 分页游标 |
| `keyword` | 否 | 模糊匹配昵称、备注、微信号 |
| `type` | 否 | `friend` 普通联系人（默认）/ `group` 群聊 / `all` 全部（不含群成员） |
| `only_starred` | 否 | 仅返回星标联系人，默认 false |

返回项：`contactId`、`type`、`nickname`、`remark`、`alias`（微信号）、`region`、`signature`、`isStarred`。

### 4. get_messages

读取某个会话的聊天记录，默认按时间从新到旧返回。

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `wx_id` | 是 | 登录用户微信 ID |
| `session_id` | 是 | 会话 ID，来自 get_sessions 的 `sessionId` 或 get_contacts 的 `contactId` |
| `start_time` | 否 | Unix 秒，只返回该时间之后（含）的消息 |
| `end_time` | 否 | Unix 秒，只返回该时间之前（含）的消息 |
| `keyword` | 否 | 仅匹配文本消息内容 |
| `limit` | 否 | 返回条数 1-200，默认 50 |
| `cursor` | 否 | 分页游标，来自上一次返回的 `next_cursor` |
| `direction` | 否 | `older` 向更早翻（默认）/ `newer` 向更新翻 |

返回项：`messageId`、`senderId`、`senderName`、`type`（text/image/voice/video/sticker/file/card/location/system/other）、`content`、`timestamp`（Unix 秒）、`isSelf`，并附会话概要 `session`。

### 5. decrypt_media

解密某条消息中的媒体文件（仅支持 image / voice / sticker），返回本地文件路径，重复调用命中缓存。

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `wx_id` | 是 | 登录用户微信 ID |
| `session_id` | 是 | 会话 ID |
| `server_id` | 是 | 消息 ID，即 get_messages 返回的 `messageId`（字符串） |

返回项：`media_type`、`content_type`（如 `image/jpeg`、`audio/mpeg`）、`path`（本地已解密文件路径）。

### 常见错误码

| 错误码 | 含义 |
| --- | --- |
| `INVALID_PARAMS` | 参数缺失或取值不合法 |
| `SESSION_NOT_FOUND` | 会话不存在 |
| `MESSAGE_NOT_FOUND` | 消息不存在 |
| `UNSUPPORTED_MEDIA_TYPE` | 该消息类型不支持解密 |
| `MEDIA_DECRYPT_FAILED` | 媒体解密失败（源文件缺失或密钥不可用） |
| `INVALID_CURSOR` | 游标失效，需从第一页重来 |
| `DB_KEY_UNAVAILABLE` / `DB_READ_FAILED` | 数据库密钥不可用 / 读取失败 |

## 托盘图标

关闭窗口后 WeLens 仍在托盘运行。托盘图标右键可显示悬浮球、启停服务、打开配置、退出。

## 常见问题

**Q：AI 说连不上 / 工具列表是空的？**
① 确认悬浮球右上角是绿点（服务运行中）；② 确认已把"复制 MCP 配置"的内容粘贴进 AI 工具并重启了它；③ 若重新生成过密钥，需重新复制配置并更新 AI 工具。

**Q：AI 说"未找到该微信账号"？**
先打开配置窗确认服务已启动，然后在 AI 对话里发一次"复制 AI 使用提示词"里的内容（或直接说"调用 get_login_accounts 获取账号"），让 AI 重新获取账号列表。

**Q：换了端口或重新生成密钥后 AI 失效？**
重新点"复制 MCP 配置"，更新到 AI 工具的设置里。

**Q：悬浮球不见了？**
在托盘图标右键 → 显示悬浮球。

**Q：复制按钮点了没反应？**
剪贴板可能被其他程序（如输入法、剪贴板管理器）短暂占用，再点一次即可；连续失败请到日志目录查看原因。

**Q：卸载 / 退出后我的数据在哪？**
聊天记录始终在微信本机数据目录中，WeLens 不复制、不上传；解密出来的图片/语音缓存在程序目录的 `media_cache` 文件夹，可随时整个删除。
