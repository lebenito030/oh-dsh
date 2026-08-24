# Agent Note: 浏览器移交由各 surface 启动器独占持有

Status: implemented

[English](2026-08-24-surface-launchers-own-browser-handoff.md) | 中文

## 问题

DSH runtime 0.1.1-rc.2（随 PR #122 引入）为 `@deepseek-ai/dsh-web-app` 新增了 `openBrowser` 配置字段，默认 `true`。Desktop Electron 外壳只以 `--profile desktop` 启动 runtime，并自己把服务 URL 加载进窗口，没有任何地方覆盖这个默认值。于是每次 desktop 启动都会打印 `dsh web: opening the default browser`，把同一 URL 交给系统默认浏览器。浏览器标签页没有 Electron preload，`window.dshDesktop` 不存在，`@oh-dsh/desktop` 的 client 入口 apply 失败并抛出 "preload bridge is unavailable outside Oh-DSH Desktop"，用户看到的就是「Failed to load plugins」弹窗。

Oh-DSH Web 启动器存在镜像问题：它已经通过 `--open/--no-open`（默认：交互终端才打开）持有移交逻辑，bundle 的默认值会在它之上再开第二个标签页。

## 决策

两个 surface 的 patch 层（desktop 的 `cordis.patch.yml`、web 的 `web/cordis.patch.yml`）都在 `web-runtime` 条目上固定 `openBrowser: false`。Desktop patch 本就覆盖着同一 config 对象的兄弟字段（`printUrl`、`surfaceContext`、`trustedHosts`），修复停留在既有覆盖块内。打开浏览器保持为启动器决策：Electron 外壳把 URL 加载进自己的窗口，web 启动器保留其交互终端默认。

## 考虑过的替代方案

**在 runtime 命令行上传 `--no-open`。** staged runtime 的 `bin.js` 没有这个旗标；宣传 `--no-open` 的消息文本属于 web-app 的独立 surface，不属于外壳使用的 profile-boot CLI。

**顺手设 `printUrl: false`。** URL 行对 `desktop.log` 诊断有用，且已被有意启用；错的只是浏览器移交。

**在 `src/main.ts` 里用环境变量修。** 配置来自 cordis patch 层，另造环境变量旁路会在既有路径之外增加第二条配置通道。

## 后果

desktop 启动不再打开系统浏览器标签页，消除了在那里出现的误导性「Failed to load plugins」弹窗。web 启动在启动器控制下只开一个标签页。若未来 runtime 版本重命名或移除 `openBrowser`，固定的覆盖要么不再生效，要么在启动时的 profile 组合中大声失败，而不是静默回归。已用 `bin.js --profile desktop|web --dump-config` 验证：两个 profile 组合出的 `web-runtime` 配置都带有 `openBrowser: false`。
