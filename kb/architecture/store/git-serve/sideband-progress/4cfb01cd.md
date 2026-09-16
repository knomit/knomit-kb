---
type: observation
domain: [store, git, git-serve, protocol, progress]
confidence: 0.95
sources: 1
entities: [sideband.NewMuxer, sideband.Sideband64k, sideband.ProgressMessage, capability.Sideband64k, NewUploadRequestFromCapabilities, demux.go, buildAdvRefs, negotiation.objects]
motifs: [advertisement-changes-all-clients, protocol-framing-boundary]
refs: ['src://7b4887ce51d9/internal/store/httphandler.go@745a2768900d4fc2d8fba104f7016d4eb36be2ed:e65b9150ce4f9780d0507b2644f55836eb1d1800', 'src://7b4887ce51d9/internal/store/httphandler_advrefs.go@745a2768900d4fc2d8fba104f7016d4eb36be2ed:d93e23ba97cbd9c237a026366e664fa48ba20a43', 'kb://3ec012f5b4d2/kb/decisions/repos/git-serve/curated-advertisement/e4eebef9.md', 'kb://3ec012f5b4d2/kb/gotchas/store/git-serve/shallow-protocol/f1cf2d23.md', 'kb://3ec012f5b4d2/kb/gotchas/store/git-serve/pack-nondeterminism/05b2e0c6.md', 'https://github.com/knomit/knomit/pull/208']
---
# knomit's git endpoint advertises side-band-64k and streams progress lines (`knomit: sending N objects`, `knomit: sent X MiB`, `knomit: done`) on band 2 while the pack goes on band 1 — and because go-git requests sideband on EVERY fetch the server advertises it for, every go-git consumer of a knomit origin now takes the muxed path

PR #208. The upload-pack handler advertises side-band-64k (alongside the curated advertisement of e4eebef9). When the request carries it, the packfile is written through sideband.NewMuxer(sideband.Sideband64k, w) on band 1, and progress lines go to band 2 via WriteChannel(sideband.ProgressMessage, …): `knomit: sending N objects` at the start (the object set is built once per fetch, so N is known), `knomit: sent X MiB` every ~1 MiB, `knomit: done`, then a terminating flush-pkt. Band-2 write errors are ignored. Real git prints these as `remote: …`; go-git delivers them to the caller's Progress writer.

Framing rules that the review measured against a pre-branch dev baseline: muxing starts only AFTER the ACK/NAK section; the shallow section, the deepen probe round and the bare NAK round stay RAW even when sideband was requested (see f1cf2d23, judgement 2); a muxed response ends in a flush-pkt while a non-sideband one legitimately ends in raw pack bytes. One unbanded byte inside the muxed region makes go-git's demuxer fail with "unknown channel" for every subscriber, and a missing terminating flush hangs or errors them.

Client-side fact that surprised two people: go-git requests side-band-64k on FETCH whenever the server advertises it, Progress writer or not (v5.19.2 ulreq.go:84 NewUploadRequestFromCapabilities). The Progress-gated rule at remote.go:317-322 is PUSH only (newReferenceUpdateRequest). With no writer the demuxer silently discards band 2 (demux.go:125). Consequence: subscription, clone, the depth-1 wizard probe and the sync loop ALL run the muxed path now, so the existing store/repos/web suites staying green is real protocol coverage, and the non-sideband path is reachable only by a hand-built request.

Testing note: never compare raw pack bytes (05b2e0c6); compare framing and the delivered object set. Object-set equality between a sideband and a non-sideband request for the same wants is also what proves the muxer is conditional.
