# dsh-pet ↔ DSH 会话打通：修改方案（已实施）

> 需求：桌宠的对话与 DSH 某个工作区下指定的某个会话关联，在 DSH 里和桌宠聊天。
> 前置阅读：`00_dsh-xiaojing_实现分析.md`。

## 1. 已确认的需求结论

1. **绑定粒度**：全局绑一个 DSH 会话（多只宠物共享同一绑定）。
2. **聊天形态**：桌宠从「有自己的脑子」变为「会话客户端」——
   - 两个聊天界面并存：桌宠右侧栏面板 + DSH 会话页面本身；
   - 两个入口的消息全部汇入同一个绑定的 DSH 会话，回复从该会话流回并显示到桌宠；
   - 原直聊链路（桌宠自己调 `ctx.llm.stream` 生成回复）**替换**，不再单独生成回复。
3. **发送路线（已验证）**：客户端注入 `remote.session` 服务，调 `remote.session.prompt({requestId, sessionId, mode, content, clientTimeZone})` 发普通用户消息。
   - 铁证：dsh-api-session-controller/lib/client.js:4249 的 inject 列表含 `remote.session`；官方聊天框 dsh-client-ui-conversation/lib/client.js:4665 即 `session.prompt([{type:"text", text}], "queue")`。
   - content 为分块数组，文本块 `{type:"text", text}`；mode = `"queue"`（排队）或 `"steer"`（打断当前轮）；requestId 用 randomUUID()。
   - `remote.commands.execute`（sc client.js:2473 command()）是斜杠命令专用（注释「leading slash included」），不用于普通消息。
4. **Helper（已实现）**：独立桌面 Helper（Electron 小窗）与网页端共用同一条绑定。Helper 本身没有 `remote` 服务，发送走宿主 `POST /dsh-pet-7340/chat` → `chatInSession`（宿主进程内调 `sessionController.prompt`），接收侧复用宿主会话观察器；绑定入口做进宠物右键菜单的「DSH 会话」枝。

## 1.1 实施状态（结论）

| 项 | 状态 |
| --- | --- |
| 一期：网页内打通（面板 + 弹窗 + 右键菜单绑定） | ✅ 已完成并实测通过 |
| 二期：Helper 小窗打通（右键菜单绑定 + 复用宿主发送链路） | ✅ 已完成 |
| 两端绑定互通 | ✅ 同一个 `sessionBinding` 值，同一条宿主路由读写 |

细节见 `02_dsh-xiaojing_网页版实施记录.md`、`04_dsh-xiaojing_Helper实施记录.md`。

## 2. 总体架构

```
桌宠对话面板 ──┐
              ├──▶ 绑定的 DSH 会话（用户消息入口）
DSH 会话页面 ──┘
绑定会话 ──assistant/message 事件──▶ 宿主 session-watch 过滤器 ──▶ 桌宠面板/气泡显示
```

不推翻「不碰 DSH 内核」边界：只读订阅 session/event + 使用客户端已有服务，控制面收敛在 dsh-pet 自己的路由与状态叶子。

## 3. 分期改动清单

### 第一期（DSH 网页内打通）✅ 已完成

| 模块 | 改动 | 文件 |
|---|---|---|
| 配置 | main-config.jsonc 新增全局 `sessionBinding: { sessionId, enabled }`（JSONC 定点改写，保留注释） | `src/host/config/session-binding.ts`、`src/host/config/jsonc-edit.ts` |
| 宿主接收侧 | session-watch 新增绑定会话过滤器：绑定会话的 user/message、assistant/message、turn/end → 写 `S.pets.<id>.sessionChat` 新叶子 | `src/host/plugin/session-chat.ts`（挂在 `session-watch.ts` 的第一个监听器） |
| 宿主发送侧 | `POST /dsh-pet-7340/chat` 无图分支优先走 `chatInSession`，失败/未绑定/超时回落 `chatWithPet` | `src/host/plugin/session-prompt.ts`、`src/host/routes/chat.ts` |
| 客户端发送侧 | 面板 sendMessage 改走 `ctx.get('remote.session').prompt({requestId, sessionId, mode:"queue", content:[{type:"text", text}]})` | `src/client/app.ts`（`buildPetChatSessionPort`） |
| 客户端面板 | 绑定会话选择器（`ctx.get('sessions').list` 快照）+ 消息视图渲染 sessionChat 叶子；宠物右键菜单「DSH 会话」枝 | `src/client/chat/*`、`src/client/pet/session-menu.ts` |
| 记忆 | 原直聊 memory.json 链路随直聊替换而退役（文件保留，不再写入） | `src/client/chat/chat-memory.ts` |

### 第二期（Helper 打通）✅ 已完成

- **绑定入口**：宠物右键菜单新增「DSH 会话」枝 —— `DSH 会话 ▸ 工作区 ▸ 会话`，与网页端同一份分组/过滤逻辑（`src/shared/session-list.ts`），勾选当前绑定、首项「不绑定（解除）」；宿主不可达时整枝隐藏。
- **列表来源**：新增 `GET /dsh-pet-7340/session-binding/sessions`（`sessionQuery.listSessions` + `readTitleSnapshots` + `workspaceRegistry.list`，服务缺失时降级）。
- **发送链路**：Helper 无需新代码——小窗对话弹窗本来就打宿主 `POST /dsh-pet-7340/chat`，一期接上 `chatInSession` 后自动进绑定会话。
- **接收链路**：复用宿主会话观察器（宿主在事件源头，天然覆盖 Helper）。
- **菜单稳定性**：会话列表改为「静默预取 + 15 秒过期」，打开菜单的路径绝不反手重开菜单（否则 DOM 反复重建，表现为菜单一悬停就消失）。

## 4. 实施顺序

1. **Spike A（发送）**：✅ 已完成：`remote.commands.execute` 是斜杠命令专用，正式路线 = `remote.session.prompt`，官方聊天框同款用法。
2. **Spike B（接收）**：✅ 已完成：session-watch 订阅 `assistant/message` 能拿到正文，流式过程由 `assistantOpen` 标志收成末条，`turn/end` 收尾。
3. 配置项 + 宿主过滤器落地。✅
4. 面板 UI（选择器 + 消息视图 + 发送分流）。✅
5. Helper 通道。✅

## 5. 技术备忘

- 宿主半侧改动需重开 DSH 生效；客户端半侧刷新页面即可（README.md:318）。
- remote/remote.commands 仅存在于客户端半侧，宿主进程内没有——所以 Helper 一律走 HTTP 回宿主，由宿主自己调 `sessionController`。
- 写绑定用 JSONC 定点改写（`src/host/config/jsonc-edit.ts`）：只替换 `sessionBinding` 那一段字符，用户手写的 `//` 注释与缩进留得住；校验不通过就抛错不写盘。
- 已知取舍：带图消息仍走旧的 `chatWithPet` 直聊（绑定链路只投文本块）；Helper 端拿不到 `blank`/`retainedBy`，未命名占位会话不过滤；菜单列表缓存 15 秒，另一端改绑后需重开菜单才见 ✓。
- dsh-pet 仓库改动先在本文档记录方案再动代码，改动尽量小、可单独成 patch。
