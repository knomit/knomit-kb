---
type: observation
domain: [app, tls, fleet, decisions]
confidence: 0.88
sources: 1
entities: [app.Listeners.StartTLSWhenInstalled, Listeners.tlsCfg, repos.Manager.SetTLSStarter, Manager.startTLSAfterInstall, TestStartTLSWhenInstalledIsWiredOnlyByServe, TestListenersApplyHasOnlyTheSignalLoopAsCaller]
motifs: [narrow-door, decided-once-at-boot]
refs: ['src://7b4887ce51d9/internal/app/listeners_tls_start.go@d68abd8fb5684453cd4c013cbea37643d07bf462:def836d210a1d825e0c82da1aa0b4306a4a2a655', 'src://7b4887ce51d9/internal/app/listeners_tls_start_test.go@d68abd8fb5684453cd4c013cbea37643d07bf462:3ac0cea2477877edc460f5fcc01954651f89f8df', 'src://7b4887ce51d9/test/archtest/reload_callers_test.go@d68abd8fb5684453cd4c013cbea37643d07bf462:62db6685a1c297bf841191750f05c8c82b00657b', 'src://7b4887ce51d9/cmd/serve.go@d68abd8fb5684453cd4c013cbea37643d07bf462:3dcf0c903d1ff2b5708dddf24413b355119eed94', 'src://7b4887ce51d9/internal/repos/manager.go@d68abd8fb5684453cd4c013cbea37643d07bf462:b1d020a0d0b58a531c510acf3a811c3639ee0705', 'src://7b4887ce51d9/internal/repos/fleet_listener_start_test.go@d68abd8fb5684453cd4c013cbea37643d07bf462:d5a9e5ebebc67410af09db1452500d52771234b1', 'kb://3ec012f5b4d2/kb/gotchas/desktop/tls/install-does-not-start-listener/fe172776.md']
---
# A fleet create, join or host-here that succeeds starts the configured mTLS listener with no restart, through app.Listeners.StartTLSWhenInstalled: a narrow door that opens ONLY a listener down with reason no_certificate, reads no config and never closes, moves or re-binds one; Apply/Reload keep the signal loop as their one caller

DECISION (fleet PR8b item 3; PR #528 B1 hand-over). Options were: (a) call Listeners.Apply/Reload from the fleet operations (forbidden by test/archtest TestListenersApplyHasOnlyTheSignalLoopAsCaller: a reload re-binds every port and must stay off every route); (b) keep telling the user to restart or run `knomit reload`; (c) a narrow method. Chosen (c), without loosening the reload archtest.

WHAT. `StartTLSWhenInstalled()` (internal/app/listeners_tls_start.go), under applyMu: does anything only when the server is not closing, the [tls] config the listeners reflect (`tlsCfg`: the boot's, then each reload's, set in reconcileTLS) has an addr, and TLSState.Reason == no_certificate (a listening listener has no reason). Then it opens exactly that listener (OpenTLSServer, installTLS), advances server.json's generation and rewrites it. A held port (addr_in_use) or a certificate that does not load stays the operator's (restart or reload). cmd/serve.go hands it to `Manager.SetTLSStarter`; `CreateFleet`, `JoinFleet` and `HostFleetHere` call `startTLSAfterInstall` once they SUCCEED (a refusal starts nothing). The desktop spawns `knomit serve --supervised`, so it gets the same wiring.

PINS. A new archtest, TestStartTLSWhenInstalledIsWiredOnlyByServe: the method is named only in cmd/serve.go outside its defining file. Verified in the real binary: a fresh home with [tls].addr set booted no_certificate; POST /fleet/create over the unix socket answered 201 and GET /fleet then said listening, the port bound, log "mTLS listener started".

CONSEQUENCES. The Fleet pages no longer ask for a restart after a create (FleetWizard Hosting/Review); listenerNotRunning says a create/join/host-here starts it and a restart or reload is for a listener still down after them. kb fe172776 ("install never STARTS a listener") is superseded for these three operations.

MISREADING. "Any route can now start listeners": only these three local-listener-only fleet operations, and only this one listener, only from the no_certificate state.
