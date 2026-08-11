# FreeBuff Proxy Bridge — Delivery Tasks

Ordered risk-first; each slice is independently verifiable with `go test ./...` against the mock upstream in `internal/testutil`. Slices 2–4 build on the previous one.

**Status snapshot (2026-08-11)**: **Slices 1–5 COMPLETE.** The proxy builds, serves `/v1/chat/completions` + `/v1/models` + `/healthz`, drains gracefully on SIGINT/SIGTERM, and passes `go build` / `go vet` / `go test ./...` locally (116 tests) plus `go test -race ./...` in CI (GitHub Actions, Linux). Remaining: live verification with a real FreeBuff token (blocked on user) and a docker-build smoke on a machine with Docker.

---

## ✅ Slice 1 — Skeleton, config, model registry (DONE, verified)

- [x] `go.mod` (module `freebuff-proxy`, Go 1.26), `cmd/freebuff-proxy/main.go` (flags: `-config`, `-v`), `.env.example`, `.gitignore` (`.env`, `dump/`, `*.log`)
- [x] `internal/config`: env + JSON config (keys mirror env names, env wins), defaults: `LISTEN_ADDR 127.0.0.1:3457` (loopback by default, PRD §3), `UPSTREAM_BASE_URL https://codebuff.com` (normalized to `www.`), `AUTH_TOKENS`, `ROTATION_INTERVAL 6h`, `REQUEST_TIMEOUT 15m`, `SESSION_CALL_TIMEOUT 30s`, `API_KEYS`, `HTTP_PROXY` + `SOCKS5_PROXY`, `COST_MODE`, `REGISTRY_REFRESH 6h`, `DEBUG_DUMP`, `LOG_FILE`
- [x] `internal/registry`: model→agent map; parse 5 upstream TS files with const/alias/Set/object resolution, depth-capped; root-agent map wins, then first-seen; hardcoded fallback; 6h refresh keeping previous state; `AgentForModel(model)`, `Models()`
- [x] Tests: config defaults/overrides/token-splitting; registry fallback parse + synthetic TS fixture parse + refresh failure keeps state

## ✅ Slice 2 — Upstream client + session lifecycle (DONE, verified)

- [x] `internal/upstream`: `Client` (per token) — `ChatCompletions` streaming with the CLI envelope (`x-freebuff-model`, `x-freebuff-instance-id`, `Authorization: Bearer`, `Accept: application/json, text/event-stream`, rotating UA list incl. `codebuff-cli/1.x`, `freebuff/0.2.4`, `ai-sdk/openai-compatible/.../codebuff`), `codebuff_metadata {run_id, client_id (13-char base36), freebuff_instance_id}`, `cost_mode` per config, `provider.data_collection=deny`, `stop:["cb_easp"]`, forced `stream:true`; CreateSession/GetSession/EndSession/StartRun/FinishRun with 30s call timeout; HTTP CONNECT + SOCKS5 outbound proxy; abort propagation
- [x] `internal/session`: per-token single-flight state machine — readiness (active w/ 5s expiry margin, disabled → ""), waiting room (typed `WaitingRoomError{position, queueDepth, retryAfter}`), refresh loop (GET if queued-with-instance else POST, 5-iteration status loop, poll delay clamped [1s,5s]), `ended/superseded/none` → recreate; EndSession
- [x] `internal/testutil`: mock codebuff.com (httptest) — session create/poll/end, agent-runs START/FINISH, chat SSE (configurable: queued/active/disabled, session errors, run errors, 401), abort detection
- [x] Tests: session state machine transitions; waiting room error + retryAfter; single-flight; envelope exactness

## ✅ Slice 3 — Run manager + multi-token pool (DONE, verified)

- [x] `internal/runs`: per-agent run lifecycle — acquire (cooldown check, 6h rotation), START, FINISH, draining list, finishIfReady, boot prewarm, 60s maintain ticker, shutdown drain + session end
- [x] `internal/pool`: round-robin start + linear failover; all-waiting-room → lowest queue position; per-token cooldown (401 → 30 min); `Acquire(model) → (lease, agentID)`; recovery helpers (`InvalidateSession`, `InvalidateRun`, `CooldownToken`, `Chat`); healthz snapshot
- [x] Tests: rotation; FINISH on rotation/shutdown; prewarm; round-robin; failover; cooldown; best-position; concurrent hammer; invalidation/cooldown/chat helpers

**Defects found + fixed during validation (2026-08-11)**:
1. **Pool shutdown deadlock** — `prewarm`/`maintainLoop` never called `wg.Done()`; `Pool.Shutdown` hung forever (would have deadlocked production SIGTERM). Fixed with `defer p.wg.Done()`.
2. **Pool test constants wrong** — expected `minimax/minimax-m3 → base2-free-minimax-m3`, but the JS-faithful fallback assigns the five base2-free models to generic `base2-free`. Tests now use exclusively-owned models (`z-ai/glm-5.2`, `poolside/laguna-s-2.1`).
3. **Flaky request-count undercount** — `Snapshot()` summed only active+draining runs; FINISHed rotated runs took their counts with them. Fixed with a cumulative `totalRequests` counter.
4. **Session single-flight DATA RACE** (CI `-race`, production) — `EnsureSession` read `m.refreshCh` outside the lock in its select while the refresher set it to nil under the lock (could also deadlock waiters on a nil channel). Fixed by capturing the channel under the mutex.
5. **Pool inflight leak** — when `Acquire` got a run but `EnsureSession` errored (e.g. waiting room), the run's inflight counter was never released → draining-list leak on rotation. Fixed with `tok.runs.Release(run)` in the error path.
6. **Default bind** — `LISTEN_ADDR` default changed from `:3457` (all interfaces) to `127.0.0.1:3457` (PRD §3; the proxy holds tokens). Compose sets `LISTEN_ADDR=:3457` explicitly.

## ✅ Slice 4 — HTTP server + OpenAI surface (DONE, verified)

- [x] `internal/convert`: request normalize (param whitelist, `developer→system`, tool-schema normalization, reasoning_effort passthrough); SSE re-encode (stable ids, drop empty chunks, preserve `reasoning_content`, error event + `[DONE]`); non-stream accumulator (fragment-stitch tool_calls, finish_reason, usage)
- [x] `internal/server`: routes `/v1/chat/completions`, `/v1/models`, `/healthz`; optional client auth (Bearer/x-api-key exact match, constant-time; healthz exempt); error mapping (503+Retry-After waiting room, 502 exhausted, 400 unknown model, 401→cooldown, verbatim upstream, 402/409/429 passthrough); streaming relay (sanitize → SSE → `[DONE]`, mid-stream error chunk); non-stream = drained stream via `convert.Accumulator`; retry-once recovery (session-invalid / run-invalid); client-gone abort propagation
- [x] `internal/telemetry`: colorized text slog handler (stderr, 4-color), file append via MultiWriter (color off with file), `RedactHeaders` (auth/cookies), `DumpRequest` helper
- [x] `cmd/freebuff-proxy/main.go`: per-token clients + session managers → pool → server; `http.Server` (ReadHeaderTimeout 15s); graceful drain (stop accepting → finish runs/sessions, 10s bound); `slog.SetDefault` routes pool logs
- [x] Integration tests (mock upstream): stream + non-stream happy path; waiting room → 503 → retry success; 401 cooldown; run-invalid rotation; client-abort; model list; auth; unknown model; healthz; all-tokens-dead 502; 405
- [x] Smoke (dummy token): `/healthz` 200, `/v1/models` 200 (12 models), chat → sane 502, Ctrl+C → exit 0 with clean shutdown logs

## ✅ Slice 5 — Hardening, packaging, docs (DONE; docker build pending a Docker host)

- [x] Shutdown sequence (SIGINT/SIGTERM → server stop → FINISH runs → end sessions, 10s force bound); server ReadHeaderTimeout 15s
- [x] Dockerfile (multi-stage: golang:1.26-alpine → alpine:3.20, static binary, non-root user) + docker-compose example (env_file, port publish, wget healthcheck on /healthz) + .dockerignore
- [x] README.md: quick start (binary + Docker), config table, 9router provider snippet, mock-upstream testing, troubleshooting (incl. Kaspersky false positive), ToS disclaimer, CI badge
- [x] CI via GitHub Actions (`.github/workflows/ci.yml`): build, vet, `go test -race ./...`, `go mod verify` on ubuntu-latest — **green**, and it caught 5 race-detector findings on its first run (see Slice 3 defects)
- [x] Docker build smoke — **verified on vps-01** (Ubuntu 24.04, Docker 29.7.2): `docker compose up --build` → container healthy, `/healthz` 200, `/v1/models` serving the live-parsed registry (27 agents / 12 models from Codebuff TS sources), full chat path exercised (dummy token → upstream 404 → clean 502 mapping). Image: 26MB (7.18MB uncompressed layers).

## Live verification (in progress — token provided 2026-08-11)

- [x] User provides FreeBuff token — done (local `.env` + VPS `.env`, both gitignored)
- [x] Real upstream smoke — **PASSED**: non-stream ✅, streaming SSE ✅ (reasoning_content preserved, [DONE]), model list ✅, live registry refresh (27 agents / 12 models) ✅, waiting room not yet observed
- [x] Deployment — VPS (vps-01) Docker container running with the real token; Azure datacenter IP gets **limited mode** (6 sessions/day, 2 models), not a hard block
- [x] **Defect found by live testing**: upstream `do()` deferred-cancel killed streamed response bodies → 502 "context canceled" right after a 200. Fixed (cancel ownership moved to callers, `cancelBody` wrapper, regression test). This bug was invisible to mock tests — live traffic exposed it.
- [ ] A/B: `COST_MODE` omit vs `free` (upstream `costModes` includes 'free' — valid syntax), Buffy preamble on/off, `client_id` formats → settle defaults (PRD §8)
- [ ] Wait: observe a waiting-room (`503 + Retry-After`) and a session rotation in real traffic
- [ ] Research complete → `docs/research/freebuff-limitations.md` (session quotas, model-bound sessions, GLM 5/20h, ToS, ads-dead). Follow-up: per-(token, model) session cache (model switches burn quota today).

---

## Tooling & environment notes (Windows dev machine, 2026-08-11)

1. **Kaspersky false positive** — Kaspersky quarantines freshly built test binaries from the go-build cache → `go test ./internal/convert/` fails with `fork/exec ... Access is denied`. NOT an infection (validated; see `docs/security/av-kaspersky-false-positive.md`). Workarounds: Kaspersky exclusion for `Temp\go-build*`, or `go test -c -o out\convert.test.exe ./internal/convert` + run directly, or build/test in WSL2/Docker. CI on Linux is unaffected.
2. **`rtk` CLI wrapper breaks `-race`** — use the raw `go` binary for race runs (or CI).
3. **`-race` needs a C toolchain** — no gcc on the dev machine; race runs happen in CI (Linux).
4. **`GOTMPDIR`** — pointing it at `%TEMP%\opencode\gotmp` reduces AV interference on local test runs (not a full fix; the `-c` workaround above is the reliable one).
5. **Subagent note** — the `implementor`/`build` agent types returned empty results in this environment; the `general` agent type worked reliably for all build slices.
6. **Observability pass (2026-08-12)** — `LOG_LEVEL` config (debug/info/warn/error, wins over `-v`), telemetry `New`/`ParseLevel`, server access-log middleware (Info: method/path/status/ms/remote), upstream `do()` Debug lines, session/runs/pool lifecycle Debug lines, `.env.example` + README config table/troubleshooting updated.
7. **Quota & upstream hardening (2026-08-12)** — 429 rate-limit handling (`retryAfterMs`/`resetAt`/`limit`/`recentCount` → typed `RateLimitError`, token cooldown for the exact window, 429+Retry-After surfaced); quota-safe maintain loop (queued-only advancement, lazy session create — fixed idle burn of ~1 session/hour); ban detection (`403 status banned` + `resumes_at` → quarantine + 403 account_banned); explicit session statuses (`banned`/`country_blocked`/`rate_limited`/`model_locked`); tier/country diagnostics on lease + logs. Research: 32 reference repos cloned under `reference/` (gitignored), findings in `docs/research/freebuff-limitations.md` §2c–§2g. Opt-in `TLS_FINGERPRINT` (JA3 utls impersonation + browser headers) for future anti-fingerprint hardening.
