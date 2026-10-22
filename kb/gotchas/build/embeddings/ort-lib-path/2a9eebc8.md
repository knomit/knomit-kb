---
type: observation
domain: [embeddings, config, build, operations]
confidence: 0.95
sources: 1
entities: [ORT_LIB_PATH, ONNXRUNTIME_SHARED_LIBRARY, config.ONNXLibPath, onnx_lib_path, initORT, libCandidates, internal/engine/embeddings/embedder.go, internal/foundation/config/config.go, Dockerfile]
motifs: [dead-setting-looks-live, parallel-implementations-diverge]
refs: ['src://7b4887ce51d9/internal/engine/embeddings/embedder.go@0610c20083fcef9e84f42ab0311a14708bcb6193:c7ab4cd5e1fe07aa28f030af6913d625090026bf', 'src://7b4887ce51d9/internal/foundation/config/config.go@0610c20083fcef9e84f42ab0311a14708bcb6193:726e04a2e41773a709be888892cecd12a77bd06f']
---
# The ONNX runtime library path is read from the ORT_LIB_PATH env var directly; the config key onnx_lib_path / ONNXRUNTIME_SHARED_LIBRARY is parsed into config.ONNXLibPath and then read by NOTHING

`initORT` (internal/engine/embeddings/embedder.go) resolves the shared library in exactly two ways: `os.Getenv("ORT_LIB_PATH")`, else a probe of `libCandidates(exeDir)` — `<exe dir>/lib/libonnxruntime.{so,dylib}` and a couple of platform fallbacks. It never consults the loaded config.

Meanwhile `internal/foundation/config/config.go` declares `ONNXLibPath` with the TOML key `onnx_lib_path`, overlays `ONNXRUNTIME_SHARED_LIBRARY` onto it, and tilde-expands it. Nothing reads the field. Setting that env var or TOML key has NO effect.

Consequence for anyone running a built binary from outside `dist/<platform>/` (a scratch build, a test harness, a one-off `go build -o /tmp/knomit .`): the candidate probe fails, and unless `ORT_LIB_PATH` is set the boot dies with `embedder init failed … Error loading ONNX shared library "onnxruntime.so"` — embeddings are mandatory, so this is a hard boot failure, not a degraded mode. The Dockerfile, the Makefile's run targets and tools/calibrate all set `ORT_LIB_PATH`; they are the right models to copy.

The config field is dead weight, not an alternative spelling. Either wire it into `initORT` or delete it — but do not document it as a knob.
