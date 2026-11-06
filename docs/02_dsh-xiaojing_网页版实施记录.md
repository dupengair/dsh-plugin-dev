# 02 · dsh-pet 网页版会话打通 — 实施记录

> 状态：一期（网页版）已完成并经用户实测通过（2026-11-05）。所有改动**未 commit**。
> 前置文档：`00_dsh-xiaojing_实现分析.md`（插件原始架构）、`01_dsh-xiaojing_会话打通方案.md`（方案定稿）。

## 1. 目标（已确认的结论）

桌宠从「有自己的脑子（直聊 LLM）」变成「DSH 会话的客户端」：

- 全局绑定**一个** DSH 会话；
- 两个聊天界面（桌宠浮动弹窗 + DSH 主窗口右侧栏「桌宠对话」tab）都汇入同一绑定会话，替换原 `ctx.llm.stream` 直聊链路；
- 未绑定时弹窗回落直聊（行为与旧版一致）；
- Helper（Electron 独立小窗）放二期。

## 2. 实现方式与数据流

### 2.1 绑定存储

- 配置写用户层 `C:\Users\dupengair\.dsh\dsh-pet\main-config.jsonc` 的 `pets[i].sessionBinding = { sessionId, enabled }`；
- 新增 `src/host/config/session-binding.ts` 直读/直写该文件（不进配置 merge 白名单）；
- 宿主新路由 `GET/POST /dsh-pet-7340/session-binding`，客户端菜单保存绑定走这条路由。

### 2.2 发送侧（消息怎么进 DSH 会话）

- **右侧栏 tab**：客户端注入服务 `remote.session`（`src/client/app.ts`），`buildPetChatSessionPort` 用 `sessions.list` 快照建会话列表，发送调 `prompt(requestId: crypto.randomUUID(), mode: 'queue', content: [{type:'text', text}])`——与官方聊天框（dsh-client-ui-conversation/lib/client.js:4665）同一路径。服务取不到时返回 `undefined` 并 warn，面板退回直聊。
- **宠物浮动弹窗**（右键「对话」开的那个，走宿主 `POST /dsh-pet-7340/chat`）：宿主新增 `src/host/plugin/session-prompt.ts` 的 `chatInSession(petId, text)`——软取宿主 `sessionController`（namespace "session"，`ctx.get(name, false)` 模式），重读绑定 → `prompt({sessionId, requestId, mode:'queue', content})`（AbortSignal.timeout 120s）→ 等该会话 `assistant/message` 文本或 `turn/end`（按 sessionId 的待办队列，180s 超时）；返回 `{ok:true, text}`，未绑定/失败返回 `undefined` 回落旧 `chatWithPet`。`postChat` 无图分支先走 `chatInSession`；带图（配图）仍走老路径。
- 面板**零本地回显**：用户消息也从事件流回来，避免双写。

### 2.3 接收侧（回复怎么回到桌宠）

- 宿主 `src/host/plugin/session-chat.ts` 观察器挂在 session-watch 第一个监听器（`session-watch.ts` 新增 `handleSessionChatEvent(sessionId, event)`）；
- 只认绑定 sessionId 的事件：绑定 1s TTL 缓存；`user/message` 追加 + busy；`assistant/message` 同轮覆盖末条；`turn/end` 收尾；换绑整窗重建；200 条滚动窗；
- 写全局状态叶子 `sections.sessionChat = {sessionId, busy, messages}`，右侧栏 tab 从这里渲染。

### 2.4 绑定入口（右键「DSH 会话」子菜单）

- `src/client/pet/session-menu.ts`：`buildSessionMenu` 生成嵌套子枝——按工作区分组（组名=workspace title，空则 cwd 末段；组间按组内最新 updatedAt 降序，「未分组」垫底；组内降序；MAX_SESSION_ITEMS=30 全局截断）；
- 可见性**照抄官方** `sessionVisible`（dsh-client-ui-workspace/lib/client.js）：排 parentSessionId 子会话 + 排 `origin==='subagent'` + 排非 mainView 持有的 blank + 排 archivedSessionIds；工作区组只认 sessionIds 记账，无 cwd 兜底；未记账会话进「未分组」；
- `applySessionPick` → `saveSessionBinding` + 气泡文案；首项「不绑定（解除）」，当前绑定项 `✓`；
- `src/client/pet/card/entry.ts:363` 接线（叶子按对象身份映射点击）；`app.ts` 注入 `ctx.get('sessions')?.list` 与 workspaces 服务，取不到时整枝不出现不抛错。

### 2.5 memory.json 退役

- `chat-memory.ts` 的 `chatWithPet` 不再读写 `memory.json`（生成/动作/配图/say 叶子保留；deleteHistory/历史查看保留；文件未删）。

## 3. 主要修改点（git：24 改 + 8 新增，+505/−95）

**宿主新增**：`src/host/config/session-binding.ts`、`src/host/plugin/session-chat.ts`、`src/host/plugin/session-prompt.ts`（+ 各自 .test.ts、`chat-session.test.ts`）。
**客户端新增**：`src/client/pet/session-menu.ts`（+ .test.ts，19 用例）。
**主要修改**：`app.ts`（inject remote.session/sessions/workspaces）、`chat/*`（面板会话模式分支、view.ts sessionBarNode、state INITIAL）、`pet/card/entry.ts`（子菜单接线）、`host/plugin/session-watch.ts`、`host/routes/chat.ts`（chatInSession 分流）、`host/routes/session.ts`（绑定路由）、`host/config.ts`、`host/state.ts`、`shared/state.ts`、`chat-memory.ts`、`client/index.ts`（加载标记 console.log）。

## 4. 构建与安装（本机流程）

1. 仓库根 `插件开发/dsh-xiaojing`，包实体在嵌套 `dsh-xiaojing/dsh-pet/`（TS + pnpm workspace）；
2. 依赖必须用 `pnpm install --node-linker=hoisted`（isolated 布局下 rolldown 解析 react 失败、ESLint ajv 起不来；跑过普通 `pnpm install` 后需重执行）；
3. 构建 `node scripts/prepare.js`（tsdown 出 lib/* + electron-helper/shared-core）；只改客户端可 `npm run bundle`；
4. 安装 = profile `C:\Users\dupengair\.dsh\profiles\desktop\package.json` 里 dsh-xiaojing 是 `"link:D:/Work/DSH/DSH使用/插件开发/dsh-xiaojing/dsh-pet"`（Junction），**市场版 0.4.1 装的是构建产物 lib/，改 src 不重建无效**；
5. 宿主侧（node 半侧）改动需**重开 DSH** 生效；客户端 js 经 `/plugins/dsh-xiaojing/client.js` 下发且 no-store，重启即新。

## 5. 踩坑记录

- **两条聊天路径**：右键「对话」弹窗（entry.ts mountChat → shared/chat/dialog.ts → POST /chat）与右侧栏 tab（chat/register.ts）是不同链路；一期先只改了 tab，用户「看不到选择条」的根因是平时用的是弹窗。
- **弹窗「没变化」**：未绑定时 chatInSession 返回 undefined 回落直聊，UI 本来就没改——外观不变是预期，变的是消息去向。
- **会话菜单三轮迭代**：①「只列当前工作区」把用户要的「大肥鱼」工作区会话滤掉了（判定的工作区≠用户预期）；②全工作区分组版里归档/删除残留混进「DSH使用」组——根因是 cwd==workspace.path 兜底归组（官方没有这条，工作区组只认 sessionIds 记账）+ 官方 sessionVisible 还有 subagent/blank 两条没抄；③照抄官方后干净。
- **发送 API 选型**：`commands.execute` 是斜杠命令专用（「leading slash included」）；正式 API 是 `remote.session.prompt`；宿主侧对应服务名 `sessionController`（prompt 在其 lib/index.js:3218）。
- **写绑定曾会丢 main-config.jsonc 里的注释**（当时是整份 `JSON.stringify` 重写）—— 已修，改为 JSONC 定点改写，见 `04_dsh-xiaojing_Helper实施记录.md`。
- **测试环境性失败**：沙箱禁 spawn → `pnpm test` 起不来，逐文件跑仅 6 个既有失败（helper-process/log-file spawn EPERM、ai-*/runtime-state 裸 node 无 safeStorage），与改动无关；fsmonitor_ipc stderr 报错为噪音。
- **git status 里 `../.codegraph/`**：会话目录上层索引，与仓库无关。

## 6. 已验证行为（用户实测，网页版）

- 右键「DSH 会话」子菜单：按工作区分组、只显示侧栏可见活动会话（大肥鱼：日常对话；DSH使用：插件开发、配置与使用、导入与转移）；
- 绑定后弹窗气泡聊天 → 消息进绑定会话、回复以气泡返回；
- 绑定/解绑正常。

## 7. 遗留事项（后续状态）

| 项 | 状态 |
| --- | --- |
| 改动未 commit | ✅ 已提交（`ee19beb` 改名、`d76162a` Helper） |
| Helper（Electron 小窗）接入绑定会话 | ✅ 已完成，见 04 |
| `openapi.yaml` 未补新路由 | ✅ 已补（`session-binding` GET/POST、`session-binding/sessions` GET），`check:openapi` 通过 |
| 写绑定丢 jsonc 注释 | ✅ 已修（JSONC 定点改写） |
| `docs/01` 实施状态 | ✅ 已同步（标注一期/二期均已完成） |
