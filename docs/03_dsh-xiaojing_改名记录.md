# dsh-pet → dsh-xiaojing 改名与独立仓库记录

日期：2026-11-05。本文只记录已确定的结论与改动，供后续开发与回溯。

## 结论

1. 插件由 `dsh-pet` 更名为 `dsh-xiaojing`，独立仓库 https://github.com/dupengair/dsh-xiaojing （基于 PC2005-cloud/dsh-pet v0.4.1 fork，不再跟随上游）。
2. 更名范围 = **名称层**（npm 包名、插件 id、显示名、日志前缀、文档指路、仓库目录名——目录于 2026-11-06 由 `插件开发/dsh-pet` 改为 `插件开发/dsh-xiaojing`）；**契约层不动**：数据目录 `~/.dsh/dsh-pet/`、HTTP 路由前缀 `/dsh-pet-7340/`、`dsh-pet-standalone` bin、`%APPDATA%\dsh-pet-electron-helper`、`.dsh-pet-*` CSS、`x-dsh-pet-token` 头、`dsh-pet-bridge://` 协议、`DSH_PET_*` 环境变量。用户数据（配置/记忆）零迁移。
3. 中文显示名「小鲸桌宠」（en "Xiaojing Pet"）。

## 代码与配置改动（改名本体）

- 内层 `dsh-pet/package.json` name → `dsh-xiaojing`；`cordis.patch.yml` id/name 同步。
- 客户端注册 id、9 个 locale（显示名 + 卸载命令）。
- 63 个 src/runtime 文件：`[dsh-pet]` → `[dsh-xiaojing]` 日志前缀、`dsh-pet: ` 错误前缀（测试断言同步）。
- openapi.yaml 5 处错误 description；cordis-probe OWNER_NAMES 追加 `dsh-xiaojing`。
- 文档层补漏（README/INSTALL/MAKE-YOUR-OWN-PET/CORDIS 系列）：插件名指代与仓库链接改到新仓库；上游素材下载链接与「二创署名」（指向 PC2005-cloud/dsh-pet）按约定保留。
- 验证：tsc 0 错、eslint 0 问题、`node scripts/prepare.js` 全过（产物 dsh-xiaojing@0.4.1）；197 测试 191 过 6 败（均为沙箱 spawn EPERM 环境性失败，与改动无关）。

## DSH 安装侧（本机 desktop profile）

- `C:\Users\dupengair\.dsh\profiles\desktop\package.json`：dependencies 键与 bundles 条目 `dsh-pet` → `dsh-xiaojing`；link 值指向包实体目录（目录改名后为 `link:D:/Work/DSH/DSH使用/插件开发/dsh-xiaojing/dsh-pet`）。
- `node_modules\dsh-pet` Junction 删除，新建 `node_modules\dsh-xiaojing` → 同一目标。
- 重开 DSH 验证通过：设置页插件名显示「小鲸桌宠」，功能正常。

## 独立仓库

- 旧克隆 .git（在外层 dsh-pet 根）已删，原地 `git init -b main`；`.gitignore` 补 `.codegraph/`。
- 首个 commit e9c16cb（含会话打通一期全部改动），已推 `main -> main`。此后改动（含本次改名）另行 commit。

## 目录改名（2026-11-06）

- 外层仓库目录 `插件开发/dsh-pet` → `插件开发/dsh-xiaojing`；**内层包实体目录仍名 `dsh-pet`**（`插件开发/dsh-xiaojing/dsh-pet`），避免额外改动构建脚本、profile link 与文档中的包内相对路径。
- 同步改动：profile `package.json` 的 link 值 → `link:D:/Work/DSH/DSH使用/插件开发/dsh-xiaojing/dsh-pet`；`node_modules\dsh-xiaojing` Junction 重建指向新路径（目标已不存在时先 `rmdir` 再 `mklink /J`）。
- 文档同步：`插件开发/README.md`、`插件开发/AGENT.md`、`docs/00`、`docs/02`、本文中的仓库路径引用。
- 契约层仍不动：数据目录 `~/.dsh/dsh-pet/`、路由 `/dsh-pet-7340/` 等一律未变。

## 保留原名的原因（备忘）

路由前缀、数据目录、bin 名属于对外契约与用户已有数据/脚本的锚点：改它们需要数据迁移与已装链接重装，收益仅是好看，风险不成比例，故按用户确认保留。
