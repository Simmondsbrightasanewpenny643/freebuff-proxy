# FreeBuff Proxy Bridge — Delivery Tasks

Ordered risk-first; each slice is independently verifiable with `go test ./...` against the mock upstream in `internal/testutil`. Slices 2–4 build on the previous one.

**Status snapshot (2026-08-11)**: Slices 1–3 complete and green; Slice 4 half-done (convert ✅, server + telemetry ❌); Slice 5 not started. Three defects found during validation are FIXED (see below). Blocked items: Kaspersky AV interference with test binaries, `-race` needs a C toolchain, live verification needs a real token.

---

## ✅ Slice 1 — Skeleton, config, model registry (DONE, verified)

- [x] `go.mod` (module `freebuff-proxy`, Go 1.26), `cmd/freebuff-proxy/main.go` (flags: `-config`, `-v`), `.env.example`, `.gitignore` (`.env`, `dump/`, `*.log`)
- [x] `internal/config`: env + JSON config (keys mirror env names, env wins), defaults: `LISTEN_ADDR :3457`, `UPSTREAM_BASE_URL https://codebuff.com` (normalized to `www.`), `AUTH_TOKENS`, `ROTATION_INTERVAL 6h`, `REQUEST_TIMEOUT 15m`, `SESSION_CALL_TIMEOUT 30s`, `API_KEYS`, `HTTP_PROXY` + `SOCKS5_PROXY`, `COST_MODE`, `REGISTRY_REFRESH 6h`, `DEBUG_DUMP`, `LOG_FILE`
- [x] `internal/registry`: model→agent map; parse 5 upstream TS files with const/alias/Set/object resolution, depth-capped; root-agent map wins, then first-seen; hardcoded fallback; 6h refresh keeping previous state; `AgentForModel(model)`, `Models()`
- [x] Tests: config defaults/overrides/token-splitting; registry fallback parse + synthetic TS fixture parse + refresh failure keeps state

**Accept (verified)**: `go build ./...`, `go vet ./...` green.

## ✅ Slice 2 — Upstream client + session lifecycle (DONE, verified)

- [x] `internal/upstream`: `Client` (per token) — `ChatCompletions` streaming with the CLI envelope (`x-freebuff-model`, `x-freebuff-instance-id`, `Authorization: Bearer`, `Accept: application/json, text/event-stream`, rotating UA list incl. `codebuff-cli/1.x`, `freebuff/0.2.4`, `ai-sdk/openai-compatible/.../codebuff`), `codebuff_metadata {run_id, client_id (13-char base36), freebuff_instance_id}`, `cost_mode` per config, `provider.data_collection=deny`, `stop:["cb_easp"]`, forced `stream:true`; CreateSession/GetSession/EndSession/StartRun/FinishRun with 30s call timeout; HTTP CONNECT + SOCKS5 outbound proxy; abort propagation
- [x] `internal/session`: per-token single-flight state machine — readiness (active w/ 5s expiry margin, disabled → ""), waiting room (typed `WaitingRoomError{position, queueDepth, retryAfter}`), refresh loop (GET if queued-with-instance else POST, 5-iteration status loop, poll delay clamped [1s,5s]), `ended/superseded/none` → recreate; EndSession
- [x] `internal/testutil`: mock codebuff.com (httptest) — session create/poll/end, agent-runs START/FINISH, chat SSE (configurable: queued/active/disabled, session errors, run errors, 401), abort detection
- [x] Tests: session state machine transitions; waiting room error + retryAfter; single-flight; envelope exactness (headers/body/UA/client_id format)

**Accept (verified)**: session unit tests green; envelope asserted in test.

## ✅ Slice 3 — Run manager + multi-token pool (DONE, verified; 2 defects fixed)

- [x] `internal/runs`: per-agent run lifecycle — acquire (cooldown check, 6h rotation), START, FINISH (`status:"completed"`, `totalSteps`, credits 0), draining list, finishIfReady, boot prewarm, 60s maintain ticker, shutdown drain + session end
- [x] `internal/pool`: `Pool` over tokens — round-robin start + linear failover; all-waiting-room → lowest queue position; per-token cooldown (401 → 30 min); `Acquire(model) → (lease, agentID)`; healthz snapshot
- [x] Tests: rotation at interval; FINISH on rotation/shutdown; prewarm; round-robin; failover; cooldown; best-position selection; concurrent hammer

**Defects found + fixed during validation (2026-08-11)**:
1. **Pool shutdown deadlock** (`internal/pool/pool.go`) — `prewarm`/`maintainLoop` never called `wg.Done()`, so `Pool.Shutdown` → `wg.Wait()` hung forever. Would have deadlocked production SIGTERM shutdown. Fixed with `defer p.wg.Done()` in both goroutines.
2. **Pool test constants wrong** (`internal/pool/pool_test.go`) — `minimax/minimax-m3` was expected to map to `base2-free-minimax-m3`, but the JS-faithful fallback assigns the five base2-free models to the generic `base2-free` agent (documented in `registry_test.go` `expectedFallback`). Tests now use `z-ai/glm-5.2 → base2-free-glm` and `poolside/laguna-s-2.1 → base2-free-laguna-s-2-1` (exclusive owners in both fallback and live maps).
3. **Flaky request-count undercount** (`internal/runs/runs.go`) — `Snapshot()` summed only active + draining runs; runs FINISHed during rotation races left the sets and took their request counts with them (observed 280 vs 320). Fixed with a cumulative `totalRequests` counter; healthz `Requests` is now exact.

**Accept (verified)**: `go test ./...` green (see tooling notes for the Kaspersky/`-race` caveats).

## ◐ Slice 4 — HTTP server + OpenAI surface (HALF DONE)

- [x] `internal/convert`: request normalize (param whitelist, `developer→system`, tool-schema `$ref/$defs`/nullable anyOf/oneOf normalization, reasoning_effort passthrough); SSE re-encode (stable ids, drop empty chunks, preserve `reasoning_content`, error event + `[DONE]`); non-stream accumulator (fragment-stitch tool_calls, finish_reason, usage) — tests pass (verify via `go test -c` + exec, see tooling notes)
- [ ] `internal/server` — **NOT STARTED (biggest remaining build item)**: routes `/v1/chat/completions`, `/v1/models`, `/healthz`; optional client auth (Bearer/x-api-key exact match); error mapping (503+Retry-After waiting room, 502 exhausted, 400 unknown model, 401, verbatim upstream); streaming relay (sanitize chunks → SSE to client, `[DONE]`); non-streaming = drained stream via `convert.Accumulator`; client-gone abort propagation to upstream
- [ ] `internal/telemetry` — **NOT STARTED**: leveled color/file logger (main.go already has a basic `newLogger`), redacted headers, optional raw dump to `dump/` (upstream client already writes dumps when `DEBUG_DUMP`)
- [ ] **Wire main.go**: build `[]*upstream.Client` (one per token) + `[]*session.Manager`, `pool.New`, `pool.Start(ctx)`, `http.Server` with server timeouts, SIGTERM → `pool.Shutdown` + server graceful shutdown (the `wg.Done` fix makes this safe now)
- [ ] Integration tests (mock upstream): stream + non-stream happy path; waiting room → 503 → retry success; 401 cooldown; run-invalid rotation; client-abort aborts upstream; model list; auth required/rejected

**Accept**: full `go test ./...` green incl. integration; manual curl smoke documented in README.

## Slice 5 — Hardening, packaging, docs

- [ ] Shutdown sequence polish (SIGINT/SIGTERM → FINISH all runs → end sessions → 10s force-exit); server timeouts (ReadHeader 15s, per-request 15m, keep-alive tuning)
- [ ] Dockerfile (multi-stage, distroless/alpine) + docker-compose example; Windows notes
- [ ] README.md: quick start, config table, 9router integration snippet, mock-upstream testing, troubleshooting table (incl. Kaspersky false-positive section), ToS disclaimer
- [ ] CI: `go vet` + `go test ./...` + `go test -race ./...` on Linux (this machine has no C toolchain — race detector is CI-only)

**Accept**: `go test -race ./...` green in CI; README complete; Docker build works.

## Live verification (blocked on user)

- [ ] User provides FreeBuff token (+ clean egress IP if geo-blocked)
- [ ] A/B: `COST_MODE` omit vs `free`, Buffy preamble on/off, `client_id` formats → settle defaults (PRD §8)
- [ ] Real upstream smoke: stream, non-stream, waiting room, model list

---

## Tooling & environment notes (Windows, 2026-08-11)

1. **Kaspersky false positive** — Kaspersky on-access quarantines freshly built `convert.test.exe` from the go-build cache → `go test ./internal/convert/` fails with `fork/exec ... Access is denied`. NOT an infection; the code and toolchain are clean (see `docs/security/av-kaspersky-false-positive.md`). Workarounds:
   - `go test -c -o out\convert.test.exe ./internal/convert && out\convert.test.exe -test.v` (binary runs fine once written)
   - Or add exclusions in Kaspersky → Settings → Exclusions: `C:\Users\<user>\AppData\Local\Temp\go-build*` and the repo `dist/`/`bin/` output dirs
   - Or build/test in WSL2/Docker (no Kaspersky on-access there)
2. **`rtk` CLI wrapper breaks `-race`** — `rtk go test -race ...` reports "No tests found". Use the raw `go` binary (absolute path) for race runs.
3. **`-race` needs a C compiler** — no gcc on this machine; `CGO_ENABLED=1` fails with `cgo: C compiler "gcc" not found`. Options: install mingw-w64, or run race tests in CI on Linux.
4. **`config.example.json` `COST_MODE`** — was `"free"` while `.env.example` documents `""` (omit). Aligned to `""`; live A/B pending (PRD §8).
