# dsh-pet 实现分析

> 分析对象：`插件开发/dsh-pet`（git clone，插件实体在嵌套子目录 `dsh-pet/dsh-pet/`，npm 包 dsh-pet v0.4.1）
> 分析目的：为「桌宠对话 ↔ DSH 指定会话打通」改造做现状摸底。本文记录截至 2026-11-05 的仓库状态（HEAD 5861a42）。

## 1. 架构总览

```
DSH 宿主进程 (cordis)
 └─ 宿主半侧插件 dsh-pet/src/host/index.ts (name='pet')
     ├─ inject: webServer / agentDefaultModel / credentials / llm / commands / connection
     ├─ HTTP 路由 /dsh-pet-7340/* 挂在 DSH webServer 上（registerHttpRoute, host/index.ts:114）
     ├─ ctx.on('session/event') ×2：工作状态联动 + 系统通知（plugin/session-watch.ts:155,266）
     ├─ ctx.on('agent/error')（session-watch.ts:297）
     ├─ ctx.provide('petApi7340')：balance / voice / speech 注册（plugin/setup.ts:193-209）
     ├─ 斜杠命令 /pet /chat /balance
     └─ 桌面 Helper：Electron 独立进程跑同样路由（plugin/desktop-helper.ts, spawn.ts）
浏览器（DSH web 页面）
 └─ 客户端半侧 src/client/index.ts → window.__ModuleLoader__.load('dsh-pet')
     ├─ inject: slots / locale / connection / remote / remote.commands / commandUi（client/app.ts:69）
     ├─ ctx.slots.inject('shell.overlay') 渲染宠物本体
     ├─ ctx.slots.inject('sidebar.right.pane.tab') 右侧栏「桌宠对话」面板（chat/register.ts:114）
     └─ ctx.provide('petApi7340') 客户端半侧：只读 getState / menuEvents / pose / progress
独立模式：src/standalone/cli.ts，伪 ctx 上跑宿主半边，不依赖 DSH
```

双端同步：HTTP 轮询 `/dsh-pet-7340/state`（1s，PollStateStore，setup.ts:184）+ 各自叶子 counter。

## 2. 记忆机制

| 内容 | 位置 | 格式 | 生命周期 |
|---|---|---|---|
| 对话记忆 | `<DSH_HOME>/dsh-pet/memory.json`（chat-memory.ts:54） | `{ 桶(assetRoot\|petId): { 实例id: { messages: ChatMemoryMessage[] } } }`，1 轮=1问1答带 ts | 持久；按 chatMemoryRounds 截尾（默认 5 轮，chat-memory.ts:113-116）；withMemoryLock 串行队列防交错；删宠物有孤儿清理（memory-orphans.ts） |
| 配置 | `<DSH_HOME>/dsh-pet/main-config.jsonc` + pet/ 目录（setup.ts:146-158） | 合并包内默认 assets/config.jsonc | 持久 |
| 待办 | TodoStore，落 userRoot（host/index.ts:84） | — | 持久 |
| 工作状态/播放表 | 进程内存（setup.ts:214-220） | — | 重启即空 |

**关键结论：记忆只存宠物自身对话，与 DSH 会话内容完全无关。**

## 3. DSH 联动现状

- 事件订阅：`ctx.on('session/event')`（session-watch.ts:155、266），消费 turn/start、user/message、tool/call、tool/result、approval/asked、turn/end、todo/write、assistant/message（:276）；`ctx.on('agent/error')`（:297）。sessionId 取 `session.header.id ?? session.id`（:158-161）。
- **纯只读监听，无任何写回会话的调用。**
- 其他注入服务：`ctx.llm.stream`（对话生成）、`ctx.commands.register`、`ctx.agentDefaultModel.currentSelection`、`ctx.credentials`、speechToText（可选）。
- README「只做桌宠本身，不碰 DSH 内核」的边界（README.md:45）= 只读订阅 session/event + 使用宿主注入服务；控制面全部收敛在自己的 `/dsh-pet-7340` 路由与同名 Cordis 服务，不写 DSH 会话、不改 DSH 状态。
- 客户端侧用到 DSH 服务：
  - `ctx.get('sessions').list.getSnapshot().byId[sessionId].title`（app.ts:97，读会话标题）
  - `ctx.get('uiWorkspace').openSession(sessionId)`（app.ts:93，跳转会话）
  - `ctx.remote.commands.execute(sessionId, '/pet id', [])`（app.ts:129，**向指定会话提交斜杠命令**）

## 4. 聊天现状

能对话，走 LLM。链路：

```
右侧栏对话面板（slots 'sidebar.right.pane.tab', register.ts:114）
  → sendMessage() HTTP POST /chat?pet=&surface=any（chat/api.ts:258-267）
  → 宿主 chatWithPet（chat-memory.ts:222）
  → generateChat 用 ctx.llm.stream（@deepseek-ai/dsh-llm 的 createUserMessage/BlockAssembler, chat.ts:123-257）
  → 回复写 S 的 pets.<id>.say 叶子 + 落 memory.json
```

另有 `/chat` 斜杠命令入口（command-chat.ts:30）与碎碎念/梦话（同 llm、不同提示词）。
模型选择：条目配置 chatModel 优先，失败回落当前对话模型（chat.ts:130-153）。支持 `[图:名]`、`[动作:名]` 标记解析。

## 5. 可用的 DSH 会话 API 面（本仓库证据内）

| 能力 | API | 位置 | 备注 |
|---|---|---|---|
| 读会话列表 | 客户端 `ctx.get('sessions').list.getSnapshot().byId` | app.ts:97 | 含 title |
| 跳转会话 | 客户端 `ctx.get('uiWorkspace').openSession(sessionId)` | app.ts:93 | UI 行为 |
| 向指定会话发消息 | 客户端 `ctx.remote.commands.execute(sessionId, text, args)` | app.ts:129 | 目前只用于斜杠命令；**发普通文本未验证** |
| 读会话事件流 | 宿主 `ctx.on('session/event')`，含 assistant/message 正文 | session-watch.ts:276 | 「DSH 回复显示到桌宠」的现成数据源 |

注意：CORDIS-API.md / API.md / openapi.yaml 全是 dsh-pet 自己对外的 38 个 HTTP 端点（openapi.yaml:101-3032），**不是** DSH 会话 API。宿主侧完整的会话写 API 面需另查 DSH 源码（app.asar）。

## 6. 技术约束

- TypeScript + ESM，Node ≥22.12，pnpm，构建 tsdown（lib/index.js 宿主侧 / lib/client.js 浏览器侧 / lib/standalone.js CLI）。
- 运行时依赖仅 5 个（edge-tts、anthropic sdk、openai、zod、electron 工具）；cordis / dsh-* / react 均为 peerDependencies 由宿主提供（package.json:129-142）。
- 测试 `node --test` + strip-types；prepack 有 openapi 一致性检查。
- 浏览器半侧禁顶层 ESM import（client/index.ts:5）。
- 宿主半侧改动需重开 DSH，客户端改动刷新页面即可（README.md:318）。

## 7. 关键文件清单

| 文件:行 | 说明 |
|---|---|
| dsh-pet/src/host/index.ts:32 | 宿主入口，inject 六服务 |
| dsh-pet/src/host/plugin/setup.ts:90,193 | userRoot=`<DSH_HOME>/dsh-pet`；provide petApi7340 |
| dsh-pet/src/host/plugin/session-watch.ts:155,266 | session/event 双监听（打通的核心数据源） |
| dsh-pet/src/host/plugin/chat-memory.ts:54,222 | memory.json 结构与 chatWithPet |
| dsh-pet/src/host/chat.ts:123 | generateChat，ctx.llm.stream 调用 |
| dsh-pet/src/host/routes/session.ts:46 | focusSession 跳会话模式（绑定信号可仿做） |
| dsh-pet/src/client/app.ts:93,97,129 | openSession / sessions.list / remote.commands.execute |
| dsh-pet/src/client/chat/register.ts:114 | 右侧栏对话 tab |
| dsh-pet/src/client/chat/api.ts:258 | 面板发送 HTTP 路径 |
| dsh-pet/package.json:122-127 | dsh.client 注入声明 |
