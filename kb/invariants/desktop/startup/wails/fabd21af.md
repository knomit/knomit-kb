---
kind: pragmatic
type: policy
domain: [desktop, startup, wails]
confidence: 0.9
sources: 1
entities: [tools/desktop/app.go, tools/desktop/serverboot.go, run, bootKnomit, startServerBoot]
refs: ['src://knomit/tools/desktop/app.go', 'src://knomit/tools/desktop/serverboot.go', 'src://7b4887ce51d9/tools/desktop/app.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:2cbe8b46047278147296beac368d97e0967a2148', 'src://7b4887ce51d9/tools/desktop/uiproxy.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:6ded8f9875a4f55a81b30e335072b6c57653d03a']
---
# knomit-desktop must not boot the server before application.Run

In tools/desktop/app.go, run() must NOT call bootKnomit (embedder init + repos.Manager.Start + bootServer) inline before wapp.Run(). That work takes ~5-7s on a warm cache, and until Wails' application.Run() executes there is no NSApplication, so no status item can exist — inline booting kept the menu-bar icon off screen for the entire boot. The server is now started via startServerBoot (tools/desktop/serverboot.go), which runs bootKnomit on a goroutine so it overlaps Wails' own startup. Measured on 2026-07-30: tray up in the same second as process start, vs the server coming up 7s later.

Consequence for a consumer: anything new that must run before the UI appears has to be genuinely cheap (config.Load, webui.FS). If it can block on disk, network, or model loading, it belongs inside bootKnomit, not in run()'s prologue. The log line "knomit-desktop UI starting (tray up, server still booting)" marks the boundary — if it stops appearing BEFORE "server up (API-only)", the ordering has regressed.

What this does NOT mean: it is not a blanket rule that startup work must be asynchronous. singleinstance.Acquire deliberately stays synchronous — a second instance must fail before it builds any UI.

RE-CHECKED at 2104fd6d (PR #488): the rule holds and nothing was reordered. run()'s prologue now also constructs `uiUp := &uiUpstream{}`, wraps the asset handler in apiProxy(...) and calls registerSSEStream(wapp, uiUp): all of it is allocation and registration with no server dependency, which is why it belongs in the prologue. The dependency on the booting server is filled in LATER and only inside bootKnomit's goroutine (uiUp.set(srv.local) after bootServer), and until then apiProxy answers /api/ with 503 + Retry-After: 1. New consequence for the 'genuinely cheap' rule: bootKnomit can now also FAIL when its server could not bind the local socket (kb/gotchas/desktop/boot, held-socket-fails-the-boot); that failure surfaces through the same boot-status/fatal-dialog path, so it does not touch Wails from the boot goroutine.
