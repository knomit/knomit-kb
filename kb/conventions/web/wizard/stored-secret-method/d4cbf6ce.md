---
type: pattern
domain: [web, ui, onboarding, secrets]
confidence: 0.85
sources: 1
entities: [authFor, credentialFor, SecretPicker, secretFits, SET_SECRET, FleetAuthMethod, StepAccess, RemoteConnectWizard, CredentialFields]
refs: ['src://7b4887ce51d9/web/src/wizardState.ts@a20039dadd0429e64abbfc70373ca27836fb8cf4:8c1de1d31cb8345f361aaacc837616ffb1e50c54', 'src://7b4887ce51d9/web/src/fleetWizardState.ts@a20039dadd0429e64abbfc70373ca27836fb8cf4:a8987e376f8dc5c6f592d8c401095db719c8e41e', 'src://7b4887ce51d9/web/src/secrets.ts@a20039dadd0429e64abbfc70373ca27836fb8cf4:33625301a26847476b5381f070b7325b2a410c25', 'src://7b4887ce51d9/web/src/SecretPicker.tsx@a20039dadd0429e64abbfc70373ca27836fb8cf4:ee9d25ec7be2f7b764317d6b83909ae51a90d20c']
---
# In the create, connect and fleet wizards a stored secret is the UI-only auth method 'secret' plus a secretId; authFor / credentialFor turn it into secret_id with NO pasted fields, and SecretPicker offers only secrets whose type fits the URL scheme and whose hosts allow its host

wizardState.ts (`authMethod: 'secret'`, `secretId`, action `SET_SECRET`), fleetWizardState.ts (`FleetAuthMethod` includes 'secret'; `credentialFor(url, method, user, token, secretId)`) and RemoteConnectWizard (local state) all model a stored secret the same way: the method never reaches the wire; `authFor` returns `{auth_method: '', auth_token: '', secret_id}` and `credentialFor` returns `{secret_id}`, because the server refuses secret_id together with a pasted credential (400) and derives the method from the secret. `remoteKey` includes secret_id, so choosing another secret invalidates a probe like a new token does. SecretPicker (web/src/SecretPicker.tsx) filters with secrets.ts `secretFits` (token or basic over http(s), ssh_key over SSH; host matched as the server's HostAllowed does, a host entry with a port matches only that port) and its 'Create a new secret…' opens SecretForm with the remote's host pre-filled and selects the new secret.

Consequence: a new credential surface should reuse SecretPicker and the same 'secret' method shape rather than sending auth_method alongside secret_id. Not meant: the client filter is not the check; the server re-checks host and type when linking and at every git operation.
