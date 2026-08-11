# FreeBuff Proxy Bridge - Reference Analysis

**Date**: 2026-08-11 · **Status**: research complete - design input
**Goal**: build a *better* FreeBuff proxy, wired into 9router as a provider. This doc inventories what the reference repos do, what is worth stealing, and where their gaps are (our opportunities).

All reference clones live under `reference/` (gitignored, shallow clones).

---

## 1. The reference repos at a glance

| Repo | Stack | License | What it is | Why it matters |
|---|---|---|---|---|
| `reference/freebuff` (CodebuffAI/freebuff) | TypeScript/Bun monorepo | Apache-2.0 | The actual FreeBuff product + CLI (upstream source of truth, ~8.8k stars) | Protocol truth: `common/src/constants/*` model registry, CLI request envelope |
| `reference/proxy-freebuff` (nasrulhadi) | Node 18+, zero-dep | MIT | Protocol-grade OpenAI+Anthropic proxy: session/run lifecycle, error-recovery matrix, live model registry | **Best protocol replication** among the community proxies |
| `reference/freebuff2api-quorinex` (Quorinex) | Go 1.23 | MIT | OpenAI-compatible proxy, stealth fingerprinting, multi-token pool | **Multi-token rotation**, real tiktoken tokenizer, prewarm, Docker multi-arch |
| `reference/freebuff2api-miladnoo` (miladnoo) | Go | MIT | Near-identical fork of Quorinex | Skip; track Quorinex upstream instead |
| `reference/freebuff-api-kiprana` (Kiprana) | Python 3.11 / FastAPI | none (no LICENSE) | OpenAI-compatible adapter with account pooling + ad handling | **The 403-gate key**: Buffy system prompt, UA spoofing, ads (gravity/zeroclick), model-locked fallback |
| `reference/freebuff-gateway` (Haadesx) | Python 3.10 / FastAPI | MIT | General free-LLM gateway: FreeBuff → OpenRouter → Ollama failover | **Rate limiting, SQLite usage DB, cost tracking, HTML dashboard, CLI wizard** |
| `reference/9router` (decolua) | Next.js (ESM JS) + `open-sse` engine | free OSS | The AI router our proxy wires into (port 20128, `/v1`) | Provider model: our proxy = one more OpenAI-compatible provider inside 9router |

Integration shape (from proxy-freebuff README) - 9router treats the proxy as a provider:

```json
{ "freebuff": { "base_url": "http://localhost:3457/v1", "api_key": "user_...", "models": ["deepseek/deepseek-v4-flash"] } }
```

---

## 2. How FreeBuff's free tier actually works (the protocol)

FreeBuff is only usable through the official CLI - the backend **fingerprints CLI traffic** and rejects direct API calls:

- Direct `/api/v1/chat/completions` without the CLI envelope → `403 free_mode_cli_required`
- `/provider/v1/*` (public provider API) → blocked for free agents

A working proxy must replicate (sources: proxy-freebuff `lib/upstream.js`, `lib/convert.js`; kiprana `codebuff.py`, `openai_compat.py`):

**Endpoints** (base `https://codebuff.com` - Go repo rewrites to `www.codebuff.com`):
- `POST /api/v1/freebuff/session` → create (returns `active | queued | ended | superseded | disabled`)
- `GET /api/v1/freebuff/session` → poll (header `x-freebuff-instance-id`)
- `DELETE /api/v1/freebuff/session` → end
- `POST /api/v1/agent-runs` `{action:"START", agentId}` → `runId`; afterwards `{action:"FINISH", runId, status:"completed", totalSteps, directCredits:0, totalCredits:0}`
- `POST /api/v1/chat/completions` with CLI envelope
- `POST /api/v1/ads` (ad impressions - see kiprana)

**The CLI envelope (fingerprint)**, per request:
- Headers: `x-freebuff-model: <model>`, `x-freebuff-instance-id: <sessionInstanceID>`, `Authorization: Bearer <token>`, `Accept: application/json, text/event-stream`, spoofed `User-Agent`
- Body: `codebuff_metadata {run_id, client_id, freebuff_instance_id}`; `provider: {data_collection:"deny"}`; `stream: true` (always; non-stream is a drained stream); `stop: ["cb_easp"]` sentinel

**Session semantics**: sessions are 1-hour, model-bound, limited per account per Pacific day. `queued` = waiting room → surface as `503 + Retry-After`. `ended/superseded` → transparently recreate. `disabled` → proceed without instance id.

**Run semantics**: one run per agent, reused across requests, rotated every 6h, `FINISH`ed on rotation/shutdown.

**Model catalog**: agent→model mapping parsed live from `CodebuffAI/codebuff` TS sources (`common/src/constants/{free-agents,freebuff-model-ids,freebuff-models,gemini,model-config}.ts`) with hardcoded fallback.

---

## 3. The 403 gate - conflicting evidence (RESOLVE EMPIRICALLY with a live token)

| Repo | `cost_mode` | Result claimed |
|---|---|---|
| proxy-freebuff (Node, most current - 2026-08-09 docs) | **omitted** | presence of `cost_mode` trips `403 free_mode_cli_required` |
| freebuff2api-quorinex (Go) | `cost_mode: "free"` | works (older code) |
| freebuff-api-kiprana (Python) | `cost_mode: "free"` | works |

Additional 403-avoidance layers (kiprana only): system message must open with the **Buffy identity preamble** (`"You are Buffy, the strategic coding assistant..."`, verified against a HAR of the official CLI), `developer → system` conversion, `cache_control: {type:"ephemeral"}` on system messages, UA spoofing (`ai-sdk/openai-compatible/.../codebuff`, `Bun/1.3.11`, `Freebuff-CLI/0.0.105`).

`client_id` format: official SDK is 13-char base36 (`Math.random().toString(36).substring(2,15)`) - Go repo matches this; Node uses 32-hex; kiprana uses uuid4 hex[:11]. Prefer base36.

**Action item**: build with envelope configurable per-mode, then A/B a live token to settle `cost_mode` + Buffy preamble + client_id format in one shot.

---

## 4. Ideas to steal (ranked by value for "a better proxy")

### Tier 1 - protocol correctness (without this nothing works)
1. **Session lifecycle state machine** - single-flight session refresh, readiness/expiry checks, 5-iteration status loop, `queued` poll delay clamped [1s, 5s] from `estimatedWaitMs` (proxy-freebuff `lib/sessions.js`, quorinex `free_session.go`)
2. **Waiting-room surfacing** - typed `WaitingRoomError` with position/queueDepth/retryAfter → `503` + `Retry-After` header + OpenAI/Claude-specific error bodies (both Node+Go)
3. **Run lifecycle** - lazy create + 60s maintain ticker + 6h rotation + FINISH with `totalSteps` + shutdown drain (proxy-freebuff `lib/runs.js`); **upgrade: Go's background prewarm of every agent at boot**
4. **Error-recovery matrix** - retry-once semantics: `session_expired/superseded/waiting_room_*` → refresh session; `runId not found/not running` → rotate run; `401` → 30-min token cooldown; else surface verbatim (both)
5. **Live model registry** - parse agent→model maps from `CodebuffAI/codebuff` TS sources every 6h with hardcoded fallback (proxy-freebuff `lib/registry.js` - richest: 5 files, alias/object/Set resolution, root-agent map wins)
6. **CLI-envelope injection** - headers + `codebuff_metadata` + `client_id` (see §3)

### Tier 2 - resilience & scale
7. **Multi-token pool with round-robin + failover + "best waiting-room position" selection** (quorinex `run_manager.go:183-217`) - the single biggest missing feature in the Node proxy
8. **Account pooling with busy-flag lease** - asyncio.Condition serialization per account, release-notify (kiprana `codebuff.py:730-805`); **upgrade: add per-token health/cooldown** (kiprana lacks it - dead tokens keep round-robining)
9. **`model_locked` recovery** - delete mismatched session, retry; fall back to default model (kiprana `codebuff.py:598-608`)
10. **Client-disconnect abort propagation** - AbortController → upstream abort, checked at every await point (proxy-freebuff)
11. **Outbound proxy** - HTTP CONNECT + **SOCKS5** (Node hand-rolled SOCKS5 handshake; Go is HTTP-only) for geo-bypass (`402 Out of credits` on blocked country IPs, `403 country_blocked` on VPN/datacenter IPs)
12. **Ads integration** - `POST /api/v1/ads` gravity + zeroclick impression reporting, once per session, fail-open (kiprana `codebuff.py:306-412`); contributes to quota/dwell

### Tier 3 - client-facing polish
13. **Complete Anthropic↔OpenAI conversion** - thinking blocks ↔ reasoning_content, tool_use/tool_calls with per-index args accumulation, images, usage/cache fields, stop-reason mapping (proxy-freebuff `lib/convert.js` is complete; Go is a faithful port)
14. **Tool schema normalization** - resolve `$ref`/`$defs`, simplify nullable anyOf/oneOf, drop null-typed enums (proxy-freebuff `lib/convert.js:40-154`)
15. **`reasoning_content` preservation end-to-end** - sanitize stream chunks, re-add reasoning_content, fragment-stitch tool_calls into final message (kiprana `openai_compat.py`)
16. **Real token counting** - tiktoken-go codec selection for `count_tokens` instead of chars/4 heuristic (quorinex `token_count.go`; note: model-prefix matching always falls back to o200k for freebuff IDs)
17. **Model-name mapping** (gpt-4 → minimax-minimax-m2.7 etc.) with config overrides (freebuff-gateway)
18. **Sliding-window rate limiting** per-IP/per-key with OpenAI-style 429 (freebuff-gateway) - **upgrade: DB-backed + XFF only when behind trusted proxy** (their XFF trust is spoofable)
19. **SQLite usage DB + daily rollups + cost-avoidance + HTML dashboard** (freebuff-gateway `db.py`/`dashboard.py` - fix their model-cost bug: always uses default rates)
20. **Multi-provider failover chain** FreeBuff → OpenRouter → Ollama (freebuff-gateway) - **caution: their streaming failover is broken by design** (generator returned unconsumed); ours must consume/validate before commit
21. **Ops**: Docker multi-arch CI, `/healthz` with full token/session/queue/cooldown snapshot, debug stream dumps, CLI wizard with credential auto-detect (`~/.config/manicode/credentials.json`) (quorinex + gateway)

### Tier 4 - 9router-aware integration
22. **RTK-style token saver** - compress `tool_result` payloads (git diff/grep/ls) fail-open before sending upstream; 9router already does this at ITS layer, but a proxy-side variant saves upstream quota too (9router `open-sse/rtk/`)
23. **Format-translation pivot**: 9router pivots through OpenAI as intermediate format; keep our proxy's public surface OpenAI + Anthropic so 9router/claude-code/opencode all work directly
24. **Align with 9router provider conventions** - `/v1/models` live list, `x-api-key`/Bearer auth, model IDs `provider/model` (9router expects this shape for custom providers)

---

## 5. Gaps in existing implementations = our "better" opportunities

1. No repo has: multi-token pool + model-bound session auto-switching (all pin 1 session = 1 model; switching needs session end)
2. No real background session poller - a `queued` session with no traffic is never advanced (both Node+Go poll only on request/maintain tick)
3. Streaming failover broken in freebuff-gateway; naive retry-once elsewhere (no exponential backoff on session-create storms)
4. Node proxy: single token, chars/4 token counting, regex registry parser (fragile vs upstream TS changes), `client_id` format off-SDK
5. Go proxy: no `x-freebuff-model`/`x-freebuff-instance-id` headers on chat, static UA (README overclaims "randomized fingerprints"), weak session/run-call timeouts (inherits 15m), dumb 32KB copy for OpenAI streaming (no mid-stream error synthesis)
6. kiprana: dead finalize path (run accounting written but unreachable), no per-token health, synchronous ads in session-create path, unknown models silently mapped to default
7. freebuff-gateway: no real auth (any key accepted, log-only), in-memory rate limits, `len//4` token guesses, cost bug (default rates only)
8. Environment specific: Windows - stale-node port cleanup hack in proxy-freebuff (kill only node processes)

---

## 6. Recommended architecture for "our" proxy

Shape (aligns with 9router provider expectations):

```
client (9router / claude-code / opencode / any OpenAI+Anthropic client)
   │  /v1/chat/completions · /v1/messages · /v1/models · /v1/messages/count_tokens
   ▼
our-proxy  (one binary; Node/TS or Go - see decision below)
   ├─ auth layer (x-api-key / Bearer, optional)
   ├─ rate limiter (sliding window, DB-backed)
   ├─ model mapper + live registry (upstream TS parse + fallback)
   ├─ session manager (per token: single-flight refresh, waiting-room, model binding, auto-recreate)
   ├─ run manager (per agent: lazy/prewarm, 6h rotation, FINISH drain, cooldown)
   ├─ token pool (round-robin + failover + best-queue-position, per-token health)
   ├─ conversion (OpenAI↔Anthropic complete both directions, tool-schema normalize, reasoning_content)
   ├─ upstream client (CLI envelope, UA rotate, x-freebuff headers, SOCKS5/HTTP proxy, abort propagation)
   ├─ error-recovery matrix (retry-once semantics w/ backoff)
   ├─ usage DB (SQLite: requests, daily_stats, cost-avoidance) + dashboard
   └─ observability (color logs, proxy.log, debug dumps, /healthz snapshot)
   ▼
codebuff.com (free session + agent-run + chat completions)
```

Stack decision (all three implementations exist and work):
- **Node/TS zero-dep** (proven by proxy-freebuff; best protocol fidelity; easy Windows) 
- **Go** (concurrency-safe pool/session state, static binary, tiktoken-go, best for multi-token)
- **Python/FastAPI** (fast to build, but heaviest runtime + GIL serialization)

Recommended: **Go** if we lead with multi-token pooling + concurrency; **Node/TS** if we want the fastest path to a feature-complete proxy (port the Node repo's protocol layer, graft Go's pool + gateway's DB/dashboard). All upstream protocol knowledge ported here in §2-§3.

## 7. Open questions / unknowns

- ❓ `cost_mode` free-vs-absent: conflicting across repos - needs a live-token A/B (§3)
- ❓ Buffy system-prompt preamble: mandatory? (kiprana only); verify against current CLI HAR
- ❓ Current model catalog + session limits (daily quota, waiting-room depth) - refresh at build time
- ❓ `www.codebuff.com` vs `codebuff.com` normalization (Go rewrites; Node follows `www.` redirect)
- ⚠️ ToS risk: all community proxies use undocumented endpoints; account bans possible
- ⚠️ Geo-gating: blocked-country IPs get `402 Out of credits`; VPN/datacenter IPs get `403 country_blocked` - needs clean residential egress

## 8. Sources

Local (shallow clones): `reference/freebuff`, `reference/proxy-freebuff`, `reference/freebuff2api-quorinex`, `reference/freebuff2api-miladnoo`, `reference/freebuff-api-kiprana`, `reference/freebuff-gateway`, `reference/9router`

Remote: github.com/CodebuffAI/freebuff (Apache-2.0), github.com/nasrulhadi/proxy-freebuff (MIT), github.com/Quorinex/Freebuff2API (MIT), github.com/miladnoo/Freebuff2API (MIT), github.com/Kiprana/Freebuff-api (no license), github.com/Haadesx/freebuff-gateway (MIT), github.com/decolua/9router (OSS)