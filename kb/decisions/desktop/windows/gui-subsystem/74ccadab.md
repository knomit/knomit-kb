---
kind: pragmatic
type: policy
domain: []
confidence: 0.95
sources: 1
entities: [Makefile, DESKTOP_LDFLAGS, DESKTOP_BUILD, desktop, desktop-winres, knomit-desktop.exe, kb, knomit-okf, -H windowsgui, tools/desktop/rsrc_windows_amd64.syso, go-winres, setupDPIAwareness]
refs: ['src://7b4887ce51d9/Makefile@31d63ccaba7f1997ea01e79fd4bdef8f84901aff:5cf706d0a5cfa20902a37da33b0c9ac13cb84e44', 'src://7b4887ce51d9/tools/desktop/README.md@31d63ccaba7f1997ea01e79fd4bdef8f84901aff:e9373cc28d401831c178811c6088d271ad960a3f']
---
# knomit-desktop.exe is linked into the Windows GUI subsystem (-H windowsgui) and the CLIs deliberately are not — a CONSOLE-subsystem tray app is owned by the terminal that launched it

Go's default PE subsystem is CONSOLE (3). A CONSOLE-subsystem binary launched from a terminal is attached to that terminal's console, so closing the terminal window delivers CTRL_CLOSE_EVENT and KILLS the process — a tray app that dies when you close the shell you started it from. It also pops up a console window when launched any other way.

`DESKTOP_LDFLAGS` in the Makefile adds `-H windowsgui` under `ifeq ($(GOOS),windows)`, giving subsystem 2 (WINDOWS GUI). Measured before/after on dist/windows-amd64/knomit-desktop.exe: **3 -> 2**. Wails' own Windows task files pass the same flag.

SCOPED TO THE DESKTOP BINARY ONLY. `knomit`, `kb` and `knomit-okf` stay at subsystem 3 and must: they are CLIs whose entire job is writing to the terminal that ran them. Verified after the change — the three CLIs still read 3.

CONSEQUENCE, and the part that is easy to miss: a GUI-subsystem process gets no console, so `knomit-desktop.exe version` / `--version` would print nowhere. `tools/desktop/console_windows.go` repairs that on the version path only, and it must guard per stream rather than attaching unconditionally — see the AttachConsole gotcha, which is where the non-obvious part lives.

WHAT THIS DOES NOT MEAN: this is not a Wails requirement and does not change DPI handling. Wails v3 sets per-monitor-v2 DPI awareness itself at runtime (`setupDPIAwareness` in pkg/application/application_windows.go), and Windows permits DPI awareness to be set only ONCE per process — which is why the committed `rsrc_windows_*.syso` carries an icon and NO manifest. Go already embeds a default manifest (asInvoker + supportedOS).
