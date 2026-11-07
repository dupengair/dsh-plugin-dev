# dsh-codegraph-mcp 可移植化与首次发布

日期：2026-11-07。记录 codegraph-mcp 从「作者机器配置」变成可发布 npm 包的过程与结论。

## 结论

1. 包已发布：`dsh-codegraph-mcp@0.1.0`（npm 公开包），仓库 https://github.com/dupengair/dsh-codegraph-mcp ，tag `v0.1.0` 指向 `6f2b5da`。
2. 仓库里**不再包含任何机器路径**：`command` 与 `args` 改成 `!!js` 表达式，加载时探测 codegraph 的安装位置。
3. 仍是**纯配置 bundle**（无源码、无依赖、无构建）：`package.json` + `cordis.patch.yml` + `locale/{zh,en}.json` + `icon.svg` + `LICENSE`，tarball 7 个文件、9.3 kB。

## 为什么要改（问题）

原来 `cordis.patch.yml` 里写死两行：

```
C:/Users/dupengair/AppData/Roaming/npm/node_modules/@colbymchenry/codegraph/node_modules/@colbymchenry/codegraph-win32-x64/node.exe
.../codegraph-win32-x64/lib/dist/bin/codegraph.js
```

换电脑 / 换 npm 安装位置（pnpm、nvm、自定义 prefix）就装不起来，而 `failOnStartupError: true` 会让整个 MCP 启动失败。

## 改法（运行时探测）

DSH 的补丁由 Cordis Loader 解析，支持 `!!js` 表达式（`new Function("ctx","expr",'with(ctx){return eval(expr)}')`，asar 内在 `@deepseek-ai/cordis-plugin-loader/lib/index.js`；`dsh-app-boot` 用 `yaml.JSON_SCHEMA.extend(JsExpr)` 让 `!!js` 成为合法标签）。所以把两条路径改成加载时求值：

探测顺序：
1. 环境变量 `CODEGRAPH_NODE`（命令）与 `CODEGRAPH_JS`（CLI 入口）——手动指定优先；
2. 常见全局 npm 根下的 `@colbymchenry/codegraph/node_modules/@colbymchenry/codegraph-<platform>-<arch>`：
   `%APPDATA%\npm\node_modules`、`<dirname(execPath)>/../../lib/node_modules`、`/usr/local/lib/node_modules`、
   `/opt/homebrew/lib/node_modules`、`/usr/lib/node_modules`、`~/.npm-global/lib/node_modules`；
   命中后 Windows 用该包自带的 `node.exe`，其他平台用 `process.execPath`；
3. 都没有 → 返回空串，MCP 启动失败并在插件页/日志报 spawn 错误。

`command` 与 `args[2]` 各写一条独立表达式（同一套探测逻辑），`args` 其余项仍是 `--liftoff-only`、`--disable-warning=ExperimentalWarning`、`serve`、`--mcp`；`env` 保持 `CODEGRAPH_MCP_TOOLS` 打开全部 8 个工具。

## 元数据（可发布）

- `name`: `dsh-codegraph-mcp`（发布前 `npm view` 确认未占用）、`version` 0.1.0、去掉 `private`
- 补 `description` / `keywords` / `repository` / `homepage` / `bugs` / `author` / `license: MIT`
- `files`: `cordis.patch.yml`、`locale/*.json`、`icon.svg`、`LICENSE`、`README.md`
- 新增 `LICENSE`（MIT）；README 增「npm 装法」与「找不到 codegraph 时怎么办」

## 验证

- 用 loader 同款求值方式跑 patch：`command` 解析到 `.../codegraph-win32-x64/node.exe`，`args[2]` 解析到 `.../lib/dist/bin/codegraph.js`（脚本 `插件开发/.tmp/probe-patch-eval.mjs`）。
- 本机卸载旧 `@local/codegraph-mcp`、按新目录安装并重启 DSH 后实测：
  - `codegraph_status`（`projectPath`=dsh-fish）：190 文件 / 2870 节点 / 12287 边 / 13.10 MB，WAL 正常；
  - `codegraph_explore`（「guard Write 拦截秘密外发」）：返回 44 符号 / 5 文件，附完整源码与体积半径。
- `npm pack --dry-run`：7 文件、9.3 kB、无本机路径。
- 发布后 `npm view dsh-codegraph-mcp version` → `0.1.0`。

## 发版步骤（下次照做）

1. 改版本号 → `npm pack --dry-run` 核对清单 → `npm publish`（Bypass 2FA 的 granular token；否则加 `--otp=<6位码>`）。
2. `git add -A && git commit && git -c http.proxy= -c https.proxy= push origin main`。
3. `git tag -a v<版本> -m "..." && git push origin v<版本>`（tag 带 `v` 前缀）。

## 已知取舍 / 后续（v0.2.0 计划）

- 探测是启发式：pnpm 全局安装或自定义 prefix 可能探不到 → 目前靠 `CODEGRAPH_NODE` / `CODEGRAPH_JS` 手改 patch 兜底。
- v0.2.0 计划做成**真插件**（host + client）：host 侧启动时探测或读用户配置动态创建 mcp-client 实例；client 侧注册 `settings.section` 设置页，用目录选择器（`ctx.uiWorkspace.pickDirectory()`）选 codegraph 安装目录与默认项目（写进 `--path`），带「检测」按钮；配置存插件自己的文件，重启生效。届时先卸载 v0.1.0（同 `serverName: codegraph`，不能并存）。
- 设置页能力已核对：DSH 客户端有 `settings.section` / `plugins.item` 插槽，目录选择器服务为 `ctx.uiWorkspace.pickDirectory()`；**没有**选单个文件的公开 API，所以「手动选 exe」要退化为「选目录 + 校验」。
