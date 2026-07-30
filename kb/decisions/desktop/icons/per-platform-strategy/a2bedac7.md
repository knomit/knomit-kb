---
type: observation
domain: [tray, desktop, ui, build, platform]
confidence: 0.95
sources: 0
entities: [tools/desktop/trayicon_darwin.go, tools/desktop/trayicon_darwin_test.go, tools/desktop/trayicon_others.go, tools/desktop/trayicon.go, tools/desktop/app.go, tools/desktop/icon-tray-light.svg, tools/desktop/icon-tray-dark.svg, events.Common.ApplicationStarted, events.Common.ThemeChanged, application.App.Env.IsDarkMode, newTrayIconState, SetTemplateIcon]
refs: ['src://knomit/tools/desktop/app.go@bc20b84', 'src://knomit/tools/desktop/trayicon_darwin.go@bc20b84', 'src://knomit/tools/desktop/trayicon_others.go@bc20b84', 'src://knomit/Makefile@bc20b84', 'src://knomit/tools/desktop/trayicon.go@b1cb963c']
---
# Desktop icons are per-platform: macOS tray = monochrome two-tone swap (NOT a template image), Linux window icon via Options.Icon + .desktop launcher

Decision for the Wails v3 desktop app (tools/desktop) on how to give it icons across platforms.

**macOS tray icon — theme-aware two-tone swap (final).** macOS menu-bar icons should adapt to the bar's light/dark state. Constraint in Wails v3-alpha.98: (1) `SetTemplateIcon` auto-tints but is ALPHA-ONLY/single-tone — a filled diamond + distinct K collapses to a blob; (2) `SetDarkModeIcon` is a NO-OP on macOS (systemtray_darwin.go setDarkModeIcon just calls setIcon; only Windows implements real switching). So we swap the icon OURSELVES: trayicon_darwin.go embeds icon-tray-light.png (dark glyph for a light bar) and icon-tray-dark.png (white diamond + dark graph for a dark bar = 'appicon without the green'); `trayIconFor(darkMode bool)` picks one; `applyTrayIcon(app, tray)` calls `tray.SetIcon` from `app.Env.IsDarkMode()`.

**GOTCHA (boot timing) — load-bearing:** do the INITIAL pick on `events.Common.ApplicationStarted` (== applicationDidFinishLaunching on macOS), NOT synchronously during boot. macOS isDarkMode() reads `AppleInterfaceStyle` from NSUserDefaults (application_darwin.go C func), which is nil/unreliable before the app finishes launching — it reports LIGHT even in dark mode. A synchronous set therefore sticks on the wrong (light-bar) icon until the next live theme change. First shipped version had exactly this bug (dark-mode users saw the light-bar dark-diamond icon). Fix: register on both ApplicationStarted (initial) and Common.ThemeChanged (live).

**SUPERSEDED in part (2026-07-30): there IS now a synchronous set, and it must stay.** macosSystemTray.run() installs the tray image only under `if s.icon != nil`, so a SystemTray whose first SetIcon lands on ApplicationStarted has its status item built EMPTY — a blank slot on the menu bar until the hook fires. newTrayIconState (trayicon.go) therefore calls apply() synchronously before returning; pre-Run that is just a field write (SetIcon with a nil impl does not dispatch to the main thread), so nothing races. The AppleInterfaceStyle unreliability above is UNCHANGED and still real — the synchronous pick may be mistoned — but ApplicationStarted re-applies and corrects it. A briefly wrong tone beats a blank menu bar. Do NOT re-remove the synchronous apply to 'fix' the mistoning. trayIconFor is unit-tested (trayicon_darwin_test.go) because the dark↔light mapping is easy to invert.

**NOT an NSImage template — and colored badges depend on that.** The glyph is monochrome ART (two single-hue assets); the status item is NOT a template image. Nothing in tools/desktop calls SetTemplateIcon, so Wails leaves isTemplateIcon false and systemtray_darwin.m never reaches `[image setTemplate:YES]`. This is load-bearing: AppKit renders a template image as an alpha MASK and discards color, so the amber boot dot and the green update dot (trayicon.go) would both collapse to the same silhouette and become indistinguishable. "Monochrome" describes the art, NOT the rendering mode — anyone who reads it as "use SetTemplateIcon" will silently break the badges.

Rejected en route (looked bad at ~22px): full-color logo; SetTemplateIcon variants (black diamond OUTLINE+graph; inverted solid-diamond-with-K-punched-out that read as a blob/octagon — the rotate(45) rounded-rect filled poorly). Lesson: preview tray glyphs at TRUE 22px in BOTH light and dark BEFORE building (ImageMagick: composite the PNG; recolor-to-white for the dark-bar sim).

**Linux/Windows tray:** stays the COLORED icon.png (64px) via `tray.SetIcon` in trayicon_others.go (`//go:build desktop && !darwin`).

**Linux app/window icon + launcher.** Root cause of 'Linux app has no icon': `application.New(application.Options{...})` set no Icon. On Linux Wails derives the window/taskbar/alt-tab icon SOLELY from `Options.Icon` (application_linux.go setIcon + webview_window_linux.go `w.setIcon(app.icon)`); no .app bundle fallback. Fix: embed appicon.png (256px colored), pass as `Options.Icon`. For the app grid, `make desktop-install` installs the binary to ~/.local/bin, appicon.png as the hicolor 256x256 icon, and tools/desktop/linux/knomit-desktop.desktop to ~/.local/share/applications.

**Asset generation:** all committed PNGs regenerated by `make desktop-icons` (rsvg-convert + iconutil), //go:embed-ed into the binary.
