# 05 · dsh-xiaojing 发布记录（npm + Git tag）

日期：2026-11-06。记录首次发布到 npm 的完整流程、踩到的坑与后续发版规则。

## 一、结论

1. **发布载体**：npm 公开包 `dsh-xiaojing`（账号 `dupengair`），`0.4.1` 是首个发布版本；Git 侧同名 tag `0.4.1` 打在同一个 commit 上。
2. **装法变化**：别人不再需要本地路径或 GitHub 地址，插件市场里填 `dsh-xiaojing` 即可（`dsh plugin add dsh-xiaojing` 同理）。
3. **为什么之前不能 git 直装**：仓库根没有 `package.json`（包实体在 `dsh-pet/` 子目录），且 `dsh-pet/lib/`、`runtime/electron-helper/shared-core.js` 是构建产物、不入库。npm 这条路不需要改这两点——`prepack` 在打包时构建，`files` 白名单把产物一起带走。

## 二、发版步骤（本机流程）

```
cd D:\Work\DSH\DSH使用\插件开发\dsh-xiaojing\dsh-pet
# 1. 改版本号（npm 不允许同版本重复发布）
#    dsh-pet/package.json 的 version +1，commit 后再发布，保证 git 与 npm 对齐
# 2. 登录一次即可（token 会存在 ~/.npmrc）
npm login
# 3. 发布（prepack 自动跑 build + typecheck + lint + 门禁）
npm publish
# 4. Git 侧打同名 tag 并推送
cd ..
git tag -a 0.4.1 -m "0.4.1: …"
git push origin main && git push origin 0.4.1
```

`prepack` 依次执行：`npm run build`、`typecheck`、`lint`、`check:cycles`、`check:runtime`、`format:check`、`scripts/prepack-check.js`、`check:openapi`。任何一项红都会阻止发布——所以「发布成功」本身等价于过一次全量门禁。

## 三、踩过的坑

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `npm publish` 报 `E403 … Two-factor authentication or granular access token with bypass 2fa enabled is required` | 账号开了「写入需 2FA」，CLI 没拿到验证码 | `npm publish --otp=<6 位验证码>`；或在 npm 站点建 Granular Access Token 并勾选 Bypass 2FA，`npm config set //registry.npmjs.org/:_authToken=<token>` |
| 沙箱里 `npm pack` 报 `EPERM … npm-cache\_cacache\tmp` | npm 默认缓存在 `%LOCALAPPDATA%`，沙箱外不可写 | 加 `--cache <工作区路径>` 即可（正式发布在普通终端跑，不受影响） |
| `format:check` 一开始就是红的（17 个文件） | 仓库基线未统一格式 | `npx prettier --write` 那些文件；否则 `prepack` 直接拦住发布 |
| `tsc -p tsconfig.test.json` 报错 | 测试文件类型没跟上（`SessionLike.title` 必填、`MenuNode` 无 `sessionId`） | 修测试代码，不要放宽 tsconfig |
| 包体积 65.5 MB / 1100 文件 | `assets/webm`（109 个动画）+ 字体/表情/音效 | 属正常预期；`prepack-check` 有体积上限检查，当前通过 |

## 四、发布前已改的元数据

- `dsh-pet/package.json`：`repository`/`homepage`/`bugs` 指到 `dupengair/dsh-xiaojing`（含 `"directory": "dsh-pet"`），补 `author`。
- `dsh-pet/README.md`：npm 页面显示的就是这份，补了安装方式（市场填包名 / `dsh plugin add`）、功能、环境要求、许可。
- 包名 `dsh-xiaojing` 在 npm 上可用（无 scope）；若将来被占用，改用 `@dupengair/dsh-xiaojing` 并 `npm publish --access public`。

## 五、备用分发方式

`npm pack` 产出 `dsh-xiaojing-0.4.1.tgz`（65.5 MB），可以挂到 GitHub Releases，别人在插件市场用「压缩包」安装。npm 不通时的兜底，内容与 npm 包一致。

## 六、后续发版清单

1. `version` +1（如 0.4.2），commit；
2. `npm publish`（或带 `--otp`）；
3. `git tag -a <版本号>` + 推 branch/tag；
4. 更新仓库 README 的更新说明与 `docs/`（如需）。
