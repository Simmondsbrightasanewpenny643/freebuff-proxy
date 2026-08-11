# PRD — FreeBuff Proxy Bridge

**Status**: v1.2 — MVP complete · **Date**: 2026-08-11 · **Stack**: Go 1.26+ (single binary)
**Source of truth for protocol knowledge**: `docs/research/freebuff-reference-analysis.md`

**Implementation status**: Slices 1–5 of `docs/delivery/tasks.md` are implemented and verified: `go build`/`go vet` green, 116 unit/integration tests passing locally, `go test -race ./...` green in GitHub Actions CI (Linux). The proxy serves `/v1/chat/completions` (stream + non-stream), `/v1/models`, `/healthz` with graceful SIGINT/SIGTERM drain, client auth, retry-once recovery, and the full PRD §6 error matrix. Six defects found during validation (2026-08-11) were fixed — including a production data race in the session single-flight and a pool shutdown deadlock (details in tasks.md). Environment caveats: Kaspersky AV false positive on one test binary (`docs/security/av-kaspersky-false-positive.md`), `-race` is CI-only (no C toolchain locally), Docker build not yet smoke-tested (no Docker host), live verification blocked on a real FreeBuff token.

## 1. Problem

FreeBuff (Codebuff's free coding agent) exposes free models only through its official CLI. The backend fingerprints CLI traffic and rejects direct API calls with `403 free_mode_cli_required`. Existing community proxies (5 known, all cloned under `reference/`) each solve a slice of this — but none combines multi-token resilience, protocol fidelity, and operator visibility.

We build a **better** one: a Go proxy that exposes an OpenAI-compatible API (which 9router — and any OpenAI client — can wire to as a provider) while managing the full FreeBuff free-session and agent-run lifecycle upstream.

## 2. Users & journeys

- **Primary**: the user running 9router / opencode / claude-code through 9router. Journey: `9router → our-proxy /v1 → codebuff.com` with zero manual session/run management.
- **Secondary**: multiple FreeBuff tokens (multi-account) for higher throughput / failover.

## 3. Goals (Must)

- OpenAI-compatible surface: `POST /v1/chat/completions` (streaming + non-streaming), `GET /v1/models`, `GET /healthz`.
- Free session lifecycle: create/poll/end, single-flight refresh, model-bound, auto-recreate on `ended/superseded/expired`, waiting room surfaced as `503 + Retry-After`.
- Agent run lifecycle: START / FINISH, prewarm at boot, maintain ticker (60s), 6h rotation, FINISH drain on rotation/shutdown, 30-min token cooldown on 401.
- CLI-compatible request envelope: `x-freebuff-model` + `x-freebuff-instance-id` headers, `codebuff_metadata {run_id, client_id, freebuff_instance_id}`, spoofed UA. `cost_mode` handling **configurable** (default per empirical A/B — see §8).
- Multi-token pool: round-robin start + linear failover + best-waiting-room-position selection; per-token health/cooldown.
- Model registry: live agent→model map parsed from `CodebuffAI/codebuff` TS sources (5 files), refreshed every 6h, hardcoded fallback.
- Error-recovery matrix: session-invalid → refresh+retry once; run-invalid → rotate+retry once; 401 → cooldown; other → surface verbatim.
- Client-disconnect abort propagation to upstream.
- Optional client auth (`x-api-key`/Bearer, `API_KEYS`), default bind `127.0.0.1` (configurable for container).
- Outbound proxy: HTTP CONNECT **and** SOCKS5 (geo-bypass path for blocked countries).
- Structured logging (color terminal + file), debug dump mode, `/healthz` snapshot (tokens/sessions/queue/cooldown).

## 4. Non-goals (post-MVP)

- Anthropic `/v1/messages` surface — 9router translates formats itself; single OpenAI surface is sufficient for the wired-to-9router goal.
- Ad impressions (gravity/zeroclick) — kiprana-only optimization, ToS-adjacent, risky.
- SQLite usage DB + cost-avoidance dashboard — nice-to-have telemetry, defer.
- Multi-provider failover (OpenRouter/Ollama) — out of scope; that is 9router's job.
- Inbound rate limiting — defer to post-MVP (9router has its own).

## 5. MVP acceptance criteria

1. `AUTH_TOKENS=a,b` + `LISTEN_ADDR=:3457` → proxy serves `/v1/chat/completions` (stream + non-stream) against a **mock upstream** (httptest) exercising the full state machine, all green with `go test ./...`.
2. Waiting-room queue returns `503` with `Retry-After` and OpenAI-shaped error body; client retry succeeds.
3. `401` triggers 30-min token cooldown; other tokens still serve.
4. Pool failover: dead token's run/session errors → next token serves; exhausted → 502 `upstream run expired twice in a row`.
5. Model list served from registry fallback (offline) and from parsed upstream TS (online).
6. `cost_mode` toggle present and covered by tests; live A/B documented in §8 checklist.
7. Client disconnect mid-stream aborts upstream request (tested via mock).
8. `go vet` clean; no external deps beyond stdlib + tiktoken-go (token counting later — see §9).

## 6. Error / empty / edge states

| State | Behavior |
|---|---|
| `queued` session (waiting room) | `503` + `Retry-After: ceil(pollAt-now)/1000`; body `server_error`/`waiting_room_queued` (OpenAI shape) |
| `disabled` session | proceed without instance id |
| `ended/superseded/expired` | transparent recreate, retry once |
| `runId not found / not running` | rotate run, retry once |
| 401 upstream | cooldown token 30 min, surface error |
| 402 (out of credits / geo) | surface verbatim; healthz shows state |
| 429/503/timeout | surface with `Retry-After` where available |
| unknown model | 400, model list from registry |
| no tokens configured | refuse startup with clear error |
| client gone | abort upstream, log |

## 7. Architecture

```
client (9router/opencode/any OpenAI client)
  → server.go (routes, auth, error mapping)
      → convert (param whitelist, developer→system, tool-schema normalization, SSE sanitize, accumulator)
      → pool (multi-token: round-robin start + failover + best-queue-position)
          → runs (per-agent run lifecycle + cooldown)
              → session (per-token single-flight free-session state machine)
                  → upstream (CLI envelope, UA, x-freebuff headers, SOCKS5/HTTP proxy, abort)
                      → codebuff.com
  → registry (live TS source parse + fallback, 6h refresh)
```

Package layout (Go, stdlib-first):
`cmd/freebuff-proxy/main.go` · `internal/config` · `internal/registry` · `internal/upstream` · `internal/session` · `internal/runs` · `internal/pool` · `internal/convert` · `internal/server` · `internal/telemetry` · `internal/testutil` (mock upstream)

## 8. Open questions (must resolve empirically)

1. ❓ **`cost_mode`**: omit (proxy-freebuff, 2026-08 evidence) vs `"free"` (Go+Python repos). → CLI flag/env `COST_MODE`; live A/B with real token; bake winner as default.
2. ❓ Buffy system-prompt preamble + `cache_control` — kiprana-only. A/B with cost_mode test.
3. ❓ `client_id` format: 13-char base36 (SDK-faithful, Go) — default this; A/B only if 403s appear.
4. ❓ `www.codebuff.com` vs `codebuff.com` — follow `www.` redirect, normalize in config.
5. ⚠️ Live verification needs: a FreeBuff token (user-supplied) + clean residential egress (geo-gating).
6. ⚠️ ToS risk: undocumented endpoints; bans possible. Educational use.

## 9. Out of scope this iteration (recorded)

- Real tiktoken-go token counting for `/v1/messages/count_tokens` — surface not in MVP (Anthropic). Token estimate for usage logging: chars/4 heuristic is fine.
- `x-freebuff-model` on GET session poll (headers on poll = `x-freebuff-instance-id` only, per references).