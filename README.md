# 插件开发（本会话工作区）

DSH（DeepSeek Harness / cordis 平台）插件的分析、开发与验证工作目录。

## 三个插件现状

| 目录 | 性质 | 当前状态 | 远程仓库 |
| --- | --- | --- | --- |
| `codegraph-mcp/` | config-only bundle（纯配置，无源码无依赖） | 把 codegraph MCP 接入 DSH，提供 `codegraph_explore / search / node / callers / callees / impact / files / status` 八个工具；**本机已安装并验证可用** | [dupengair/dsh-codegraph-mcp](https://github.com/dupengair/dsh-codegraph-mcp)（v0.1.0） |
| `dsh-fish/` | 第三方插件源码（clone，**只读参考**） | 用于学习插件结构、事件流与边界约束；不做改动 | 上游 `ssamurias/dsh-fish`（0.8.6） |
| `dsh-pet/` | 桌宠插件源码（clone 后独立开发） | 已改造成 **dsh-xiaojing（小鲸桌宠）**，新增「DSH 会话绑定」：网页端面板/弹窗与桌面 Helper 小窗都能绑定到指定 DSH 会话，消息汇入该会话、回复回流气泡 | [dupengair/dsh-xiaojing](https://github.com/dupengair/dsh-xiaojing) |

### codegraph-mcp

- 结构：`package.json` + `cordis.patch.yml` + `locale/{zh,en}.json` + `icon.svg`，通过 `@deepseek-ai/dsh-mcp-client` 起 `codegraph serve --mcp`
- 安装：DSH 侧栏「插件 → 添加插件」，填本目录绝对路径即可（link 式安装，零依赖、无需 pnpm install）
- 注意：默认不写死项目路径，调用工具时传 `projectPath`；目标项目需先 `codegraph init`
- 文档：见 `codegraph-mcp/README.md`

### dsh-pet → dsh-xiaojing

- 结构：`dsh-pet/dsh-pet/` 是 TS 包实体（宿主半侧 + 浏览器半侧 + Electron Helper 运行时），外层放文档与构建脚本
- 已实现的会话打通：
  - 绑定是**全局单值**，存在 `~/.dsh/dsh-pet/main-config.jsonc` 的 `sessionBinding`，两端共用同一条宿主路由
  - 网页端：右侧栏面板 + 宠物右键「对话」弹窗都能绑定；消息经 `remote.session.prompt` 或宿主 `chatInSession` 投进会话
  - Helper 小窗：右键菜单 `DSH 会话 ▸ 工作区 ▸ 会话` 绑定，发送复用宿主 `/chat`
  - 记忆 `memory.json` 随直聊链路退役（文件保留，不再写入）
- 构建：`node scripts/prepare.js`（本机依赖用 `pnpm install --node-linker=hoisted`）；宿主半侧改动需重开 DSH
- 改名说明：包名/插件 id/显示名/日志前缀已是 `dsh-xiaojing`；**数据目录 `~/.dsh/dsh-pet` 与路由前缀 `/dsh-pet-7340` 保留原名**（用户数据零迁移）

## 目录结构

```
插件开发/
├── AGENT.md          # 本目录工作约定（改动前先看）
├── README.md         # 本文
├── docs/             # 分析与实施文档
├── codegraph-mcp/    # 插件①：codegraph MCP bundle
├── dsh-fish/         # 插件②：第三方源码（只读）
├── dsh-pet/          # 插件③：桌宠源码（dsh-xiaojing）
└── .tmp/             # 临时文件（脚本、探针、提取产物；不入库）
```

## 文档索引

| 文档 | 内容 |
| --- | --- |
| `docs/00_dsh-xiaojing_实现分析.md` | 桌宠原始架构分析（宿主/客户端半侧、记忆、DSH 联动） |
| `docs/01_dsh-xiaojing_会话打通方案.md` | 会话打通方案与实施状态（一/二期均已完成） |
| `docs/02_dsh-xiaojing_网页版实施记录.md` | 网页端（面板 + 弹窗 + 右键菜单）实施与踩坑 |
| `docs/03_dsh-xiaojing_改名记录.md` | dsh-pet → dsh-xiaojing 改名范围与契约层保留项 |
| `docs/04_dsh-xiaojing_Helper实施记录.md` | Helper 小窗实施、菜单稳定性修复、JSONC 定点改写入库 |

## 约定

- 结论与方案一律沉淀到 `docs/`，命名 `<序号>_<插件名称>_<事务>.md`；文档只留确定性结论
- 临时文件放 `.tmp/`，不落本目录根
- 三个插件各自是独立 git 仓库（`dsh-fish` 为上游 clone），推送远程时若本机代理不稳用 `git -c http.proxy= -c https.proxy= push` 直连
- 本目录仓库只收录 `AGENT.md`、`README.md`、`docs/`，其余目录与文件由 `.gitignore` 排除（源码各自在自己的仓库里）
