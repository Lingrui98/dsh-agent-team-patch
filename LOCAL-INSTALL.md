# 本机安装记录（dsh-agent-team-patch）

安装时间：2026-10-17
目标：DSH CLI 0.2.0-rc.2，npm 全局安装

```
DSH=/nfs/home/goulingrui/.nvm/versions/node/v24.4.1/lib/node_modules/@deepseek-ai/dsh
```

## 装了什么

热补丁方式，改了 3 个包共 7 个文件：

- `dsh-experimental-agent-team`：`lib/index.js`、`lib/invariant.js`、`lib/types/index.js`、`lib/types/projection.js`、`lib/types/roster.js`
- `dsh-experimental-client-ui-agent-team`：`lib/client.js`
- `dsh-experimental-tool-agent-team`：`lib/index.js`

6 个文件按 `manifest.json` 的 `patched` 字节精确覆盖，sha256 与 manifest 一致。
`client.js` 本机是 npm 包构建，与仓库 `baseline` 只差生成型 CSS 骨架，所以改用 `patches/dsh-experimental-client-ui-agent-team.patch` 打到本机文件上；归一化比对确认除 CSS 骨架外与仓库 `patched/` 逐字节相同。

## 与上游仓库的差异（本地两处改动）

1. `manifest.json`：`dsh-experimental-client-ui-agent-team/lib/client.js` 的 `patched` 哈希与 `bytes` 换成本机派生版的 `2f1d4e6e9b4cb629f6277d8c929ab33e6ac8f91b9d5df67a9786960d3fad3df5`（48092 字节，含下一节的闸口摘除）。
2. `patched/dsh-experimental-client-ui-agent-team/lib/client.js`：换成同一份本机派生版。

这两处改动只为让 `node apply.mjs --check` 与本机状态自洽（现在返回 `PATCHED patched=7 pristine=0 drift=0 missing=0` + `WIRED`，exit 0）。上游原版 `patched/client.js` 存档在 `../backups/20261017083919/client.js.repo-upstream-patched`。

## 本地额外改动：摘掉「本地模型闸口」面板

热补丁那一半的客户端带上了作者本机专用的「本地模型闸口」面板。它每 5 秒轮询 `http://127.0.0.1:11499/gate/state`，本机没有这个服务，所以面板常驻一行红字「改不动闸口：Failed to fetch」，浏览器控制台也持续报连接失败。

改动：删掉渲染调用 `(0, react_jsx_runtime.jsx)(LocalModelGate, { t })`。`LocalModelGate` 组件定义与 `GATE_BASE = "http://127.0.0.1:11499"` 保留为未引用的死代码；不渲染就不会轮询。改后 `client.js` 的 `patched` 哈希为 `2f1d4e6e9b4cb629f6277d8c929ab33e6ac8f91b9d5df67a9786960d3fad3df5`（48092 字节），`manifest.json` 与 `patched/` 已同步，`apply.mjs --check` 仍为 `PATCHED patched=7` + `WIRED`。

副作用：`checks/team-ui-check.mjs` 有 1 项断言预期失败——它要求渲染遍历同时到达休息开关与闸口面板，现在只到 `TeamRestSwitch`；其余断言与其余 8 个检查不受影响。

回滚这一段：把 `../backups/20261017083919/client.js.patched-with-gate` 覆盖回 `$DSH/node_modules/@deepseek-ai/dsh-experimental-client-ui-agent-team/lib/client.js`，再按新文件重算 `manifest.json` 与 `patched/` 的哈希。

## 校验

- `node apply.mjs --check` → `PATCHED` + `WIRED`，exit 0
- 9 个离线检查：`retire-check`、`retire-event-check`、`reuse-check`、`rest-check`、`team-command-check`、`remote-namespace-check`、`profile-wiring-check`、`bundle-check` 全部 exit 0；`team-ui-check` 因上面摘掉闸口面板而有 1 项预期失败
- 在线冒烟（2026-10-17）：`spawn_teammate` 指定 `provider/model` 成功、名册可见、`retire_teammate` 生效且让出席位、退休成员对 `retire_teammate` / `send_message` 均报 `active teammate … not found`、同名重建成功
- `dsh --profile web --dump-config` exit 0，组合树 195 条入口、190 个包名全部可解析

运行检查需要 `DSH_APP_ROOT` 指向一个「`node_modules/@deepseek-ai` 可解析、且带 react」的根。本机用的是隔离根 `/tmp/dsh-verify`（`react` 副本 + `@deepseek-ai` 软链到真实安装）。`team-ui-check` 必须能 `require('react')`，而 npm 全局安装的 DSH 里没有 react。

## 备份与卸载

- 全量备份：`/nfs/home/goulingrui/project/dsh-infra/backups/20261017083919/`
- 每个被改文件旁有 `<文件名>.dsh-retire.bak`
- 卸载：`node apply.mjs --revert --root "$DSH" --profile ~/.dsh/profiles/web`，然后重启 DSH

## 坑：不要在仓库里跑 npm

`checks/bundle-check.mjs` 会建 `node_modules/@deepseek-ai` 指向 DSH 安装的链接。npm 遇到这种链接会顺着它把全局安装里「多余」的包删掉。2026-10-17 已发生过一次：一次 `npm install react` 删掉 `$DSH/node_modules/@deepseek-ai` 下 279 个包，之后用 `cd "$DSH" && npm install` 回填、再单独补 `@deepseek-ai/dsh-util-workspace-path@0.2.0-rc.2` 修好。

结论：在仓库里跑 npm 之前先确认 `node_modules` 不存在；需要跑检查时用隔离根设 `DSH_APP_ROOT`，不要对带链接的树跑 npm。

## 生效

服务端插件无热重载：完全退出 `dsh web --port 9999` 进程后重新启动才生效。
