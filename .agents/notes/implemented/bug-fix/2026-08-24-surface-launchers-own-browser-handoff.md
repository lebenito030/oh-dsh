# Agent Note: keep the browser handoff owned by the surface launchers

Status: implemented

English | [中文](2026-08-24-surface-launchers-own-browser-handoff.zh.md)

## Problem

DSH runtime 0.1.1-rc.2 (pulled in with PR #122) gives `@deepseek-ai/dsh-web-app` a new `openBrowser` config field defaulting to `true`. The desktop Electron shell spawns the runtime with only `--profile desktop` and loads the served URL into its own window, so nothing overrode that default. Every desktop launch printed `dsh web: opening the default browser` and handed the same URL to the system browser. That browser tab has no Electron preload, so `window.dshDesktop` is undefined there and the `@oh-dsh/desktop` client entry fails to apply with "preload bridge is unavailable outside Oh-DSH Desktop", surfaced to the user as a "Failed to load plugins" dialog.

The Oh-DSH Web launcher has the mirror problem: it already owns the handoff via `--open/--no-open` (default: open on an interactive terminal), so the bundle default would open a second tab on top of the launcher's own.

## Decision

Both surface patch layers (`cordis.patch.yml` for desktop, `web/cordis.patch.yml` for web) pin `openBrowser: false` on the `web-runtime` entry. The desktop patch already overrode the sibling fields (`printUrl`, `surfaceContext`, `trustedHosts`) of the same config object, so the fix stays in the existing override block. Browser opening remains a launcher decision: the Electron shell loads the URL into its window, and the web launcher keeps its interactive-terminal default.

## Alternatives considered

**Pass `--no-open` on the runtime command line.** The staged runtime's `bin.js` has no such flag; the message text advertising `--no-open` refers to the web-app's standalone surface, not the profile-boot CLI the shell uses.

**Set `printUrl: false` too.** The URL line is useful in `desktop.log` diagnostics and was already intentionally enabled; only the browser handoff is wrong.

**Fix it in `src/main.ts` via an env override.** The config comes from the cordis patch layer, and inventing an env side channel would add a second configuration path beside the existing one.

## Consequences

Desktop launches no longer open a system-browser tab, which removes the misleading "Failed to load plugins" dialog seen there. Web launches open exactly one tab under the launcher's control. If a future runtime revision renames or removes `openBrowser`, the pinned override either no-ops or fails the profile composition loudly at startup instead of silently regressing. Verified with `bin.js --profile desktop|web --dump-config`: the composed `web-runtime` config carries `openBrowser: false` on both profiles.
