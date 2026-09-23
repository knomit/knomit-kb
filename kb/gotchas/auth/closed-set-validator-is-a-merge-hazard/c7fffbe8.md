---
type: observation
domain: [auth, windows, cli, process]
confidence: 0.95
sources: 1
entities: [auth.ParsePrincipal, auth.ViaPipe, auth.ViaSocket, cmd/grants.go, parseGrantArgs, grantsList, TestParsePrincipal_RoundTripsThisPlatformsLocalPrincipal, TestParsePrincipal_AcceptsBothLocalVias]
motifs: [correct-alone-wrong-together, enumeration-drifts-from-type]
refs: ['src://7b4887ce51d9/internal/auth/principal.go@e58f08a34a481fa682f798d1b62f84a54c34c058:0fe505786c81fec004c7610cc91070dc86519e22', 'src://7b4887ce51d9/internal/auth/principal_test.go@e58f08a34a481fa682f798d1b62f84a54c34c058:4a407b9b3b234cc3216ef0b786ef3718533f08e1', 'src://7b4887ce51d9/cmd/grants.go@e58f08a34a481fa682f798d1b62f84a54c34c058:c865b01a10fe10152f4782bf28210b401606ef0a', 'https://github.com/knomit/knomit/pull/251', 'https://github.com/knomit/knomit/pull/252']
---
# A closed-set validator over an enumerated type is a MERGE HAZARD — ParsePrincipal's via switch was correct on dev and wrong the moment ViaPipe merged in, and neither branch's tests could have caught it

auth.ParsePrincipal (added by F19 phase 2, PR #252) validates Via against a closed list:

    case ViaCert, ViaToken, ViaSocket, ViaNone:

That list was CORRECT on dev and became wrong the instant PR #251's ViaPipe merged into it. Neither side was wrong alone; the PAIR was, which is why no test on either branch could have caught it — only the merge can.

CONSEQUENCE: on Windows the local principal is ALWAYS `bridge:sid:<SID>@pipe`, and cmd/grants.go parses through ParsePrincipal for `add`, `revoke` AND `list`. So the entire grants CLI would have rejected the only local principal that platform has, with `unknown via "pipe"`.

THE GENERAL SHAPE: adding a constant and adding a validator over that constant's type are INDEPENDENT changes that each look complete. Whenever a switch or slice enumerates every member of a type, ask what happens when someone adds a member on another branch.

TWO TESTS NOW HOLD IT, and both are needed for the same reason the oversight happened: TestParsePrincipal_RoundTripsThisPlatformsLocalPrincipal goes through the REAL auth.LocalPrincipal() so it cannot rot when either half changes, but it only ever exercises ONE via per platform — TestParsePrincipal_AcceptsBothLocalVias covers @socket and @pipe on every platform, which is the property that was actually violated.

The rest of the parser was already correct for pipe principals: kind cuts at the FIRST ':' and via starts after the LAST '@', and a SID contains neither character, so `sid:S-1-5-21-...` survives whole.
