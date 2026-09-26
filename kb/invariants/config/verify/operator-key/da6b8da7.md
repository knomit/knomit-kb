---
kind: pragmatic
type: policy
domain: [config, verify, security, app]
confidence: 0.9
sources: 1
entities: [VerifyConfig.OperatorKey, KNOMIT_VERIFY_OPERATOR_KEY, store.NewStaticRoot, repos.Deps.VerifyRoot, Manager.RootOfTrust, Service.SetRootOfTrust]
motifs: [fail-closed-default, no-implicit-authority]
refs: ['src://7b4887ce51d9/internal/app/app.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:0805863d7f3628ef325a08c3ece70b36349bff75', 'src://7b4887ce51d9/internal/config/config.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:319054900051d2dc19649f448b8307767e0e52f3', 'src://7b4887ce51d9/internal/store/verify_fold.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:8687e784f78514aba7a7195b677bcc068f7c2fd0', 'src://7b4887ce51d9/internal/store/verify_first_contact_test.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:06d8c1769804507553bbd3897043c87c8c8128b5']
---
# [verify].operator_key is F09's root of trust: an ssh-ed25519 line parsed at boot (malformed fails boot), empty = unrooted = closed before any enable, and it NEVER defaults to the instance's own key

`[verify].operator_key` (env KNOMIT_VERIFY_OPERATOR_KEY) is parsed in app.New by store.NewStaticRoot: ssh.ParseAuthorizedKey, ssh-ed25519 only, compared by full fingerprint. A malformed value FAILS BOOT with a named error rather than leaving every enabled repo unrooted for a reason nobody would guess. Empty is the unconfigured root.

It travels as repos.Deps.VerifyRoot into every Service that verifies: the builder, SwapStore's rewire, the lifecycle stores (clone, initialize, subscribe) and the origin wizard's clone store (Manager.RootOfTrust()). A Service with none behaves as unrooted.

It NEVER defaults to the instance's own key, even when they are equal (the user's own setup): a default would make every instance an operator, and first-enabler-wins would return. TestFirstContact_OwnKeyIsNeverTheRoot pins this: an instance whose own key IS the operator's but with no operator_key configured is unrooted.

The F19 fleet root replaces it later as another RootOfTrust implementation: a configuration change, not a code change in the fold.
