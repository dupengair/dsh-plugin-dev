# AGENT.md — 本目录工作约定

## 身份与目标

本目录用于 DSH 插件分析与实现。当前主线：dsh-pet 桌宠插件与 DSH 指定会话的打通改造。

## 硬性约束

1. **文档沉淀**：分析结论与方案写入 `docs/`，命名 `<序号>_<插件名称>_<事务>.md`（如 `00_dsh-xiaojing_实现分析.md`），序号递增不复用。
2. **先确认后落笔**：需要与需求方确认的问题一一当面确认，全部形成结论后一次性写入文档，避免反复更新；文档里只允许确定性结论，不得出现「待确认/❓/需要对方回答」类内容，未定事项留在对话里推进、定了再写。
3. **临时文件**放 `.tmp/`，最终交付物放本目录根或 `docs/`。
4. **上游仓库只读**：`dsh-fish/`、`dsh-pet/` 是 clone 下来的源码；对 dsh-pet 的改动先在 `docs/` 记方案再动代码，改动尽量小、可单独成 patch。
5. dsh-pet 的设计边界（README.md:45）：只做桌宠本身、不碰 DSH 内核（只读 session/event + 宿主注入服务）——打通方案优先在该边界内实现，越界需先在文档中说明理由。
6. 宿主半侧（dsh-pet/src/host）改动需重开 DSH 生效；客户端半侧刷新页面即可。
7. **本目录仓库只收文档**：`AGENT.md`、`README.md`、`docs/`；其余目录与文件由 `.gitignore` 排除（各插件源码在自己的仓库里）。
8. git 推送 GitHub：代理 127.0.0.1:7897 不稳时用 `git -c http.proxy= -c https.proxy= push` 直连；本机无 SSH 密钥，远程一律 HTTPS。
9. 沙箱注意：pwsh 里 Node/Git msys 子进程 stdio:'pipe' 会 EPERM、credential helper 会崩——git push 等需要完整进程能力的命令用 sandbox_permissions 放行执行。

## 环境备忘

- 会话目录：`D:\Work\DSH\DSH使用\插件开发`
- DSH_HOME=`C:\Users\dupengair\.dsh`（dsh-pet 的 memory.json、main-config.jsonc 在其下 dsh-pet/ 子目录）
- DSH 实现在 `D:\Software\DSH\resources\app.asar`（单文件，读取法见 .tmp/ 的 asar 提取脚本）
- dsh-pet 自带 .codegraph 索引（实体在 dsh-pet/dsh-pet/），查代码用 codegraph MCP 带 projectPath
