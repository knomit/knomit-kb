---
type: reference
domain: [architecture, cli, documentation]
confidence: 0.9
sources: 1
entities: [knomit serve, knomit reset, knomit verify, knomit warm-models, knomit version]
refs: ['src://knomit/cmd/root.go', 'src://knomit/cmd/serve.go', 'src://knomit/cmd/verify.go']
---
# knomit CLI has 5 subcommands: serve, reset, verify, warm-models, version

The `knomit` binary (cobra, cmd/) exposes five subcommands. **serve** starts the HTTP server (flags: --port [19278], --host [localhost], --log-file, --log-max-size [10MB], --log-max-backups [3], --log-max-age [7d]; KNOMIT_PPROF_ADDR enables pprof). **reset** wipes and reinitializes a repo (--name [trunk]). **verify** runs an integrity check across the store-integrity categories (--repo [trunk], --all, --deep; exit 0 clean / 1 issues / 2 error; read-only but takes per-branch locks). **warm-models** pre-downloads the embedding model into KNOMIT_HOME/models/<id> without booting the server (--model [embeddinggemma]). **version** prints `<semver>.<sha>`. The knomit-bridge and knomit-desktop binaries also answer `version`/`--version`.
