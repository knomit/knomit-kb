---
kind: pragmatic
type: policy
domain: [desktop, startup, wails]
confidence: 0.9
sources: 1
entities: [tools/desktop/app.go, tools/desktop/serverboot.go, run, bootKnomit, startServerBoot]
refs: ['src://knomit/tools/desktop/app.go', 'src://knomit/tools/desktop/serverboot.go']
---
# knomit-desktop must not boot the server before application.Run

In tools/desktop/app.go, run() must NOT call bootKnomit (embedder init + repos.Manager.Start + bootServer) inline before wapp.Run(). That work takes ~5-7s on a warm cache, and until Wails' application.Run() executes there is no NSApplication, so no status item can exist — inline booting kept the menu-bar icon off screen for the entire boot. The server is now started via startServerBoot (tools/desktop/serverboot.go), which runs bootKnomit on a goroutine so it overlaps Wails' own startup. Measured on 2026-07-30: tray up in the same second as process start, vs the server coming up 7s later.

Consequence for a consumer: anything new that must run before the UI appears has to be genuinely cheap (config.Load, webui.FS). If it can block on disk, network, or model loading, it belongs inside bootKnomit, not in run()'s prologue. The log line "knomit-desktop UI starting (tray up, server still booting)" marks the boundary — if it stops appearing BEFORE "server up (API-only)", the ordering has regressed.

What this does NOT mean: it is not a blanket rule that startup work must be asynchronous. singleinstance.Acquire deliberately stays synchronous — a second instance must fail before it builds any UI.
