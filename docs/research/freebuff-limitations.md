# FreeBuff / Codebuff free tier — limitations & quota research

**Date**: 2026-08-12 · **Status**: research complete (web + upstream source + commit history + 6 reference implementations + live 429)
**Sources**: freebuff.com, freebuff.com/terms-of-service (2026-07-23), npm registry `freebuff`,
CodebuffAI/codebuff GitHub (commits `7dd903b`, `793de91d9aaa`, PRs #566/#598/#657, issue #504),
live upstream constants (`free-agents.ts`, `model-config.ts`), live API behavior observed via this
proxy, and the reference implementations in `reference/` (proxy-freebuff, freebuff2api-quorinex,
freebuff2api-miladnoo, freebuff-api-kiprana, freebuff-gateway, free-buff-lol).

---

## 1. Session quotas (the "max 6 sessions" your account showed)

Sessions are **one-hour, model-bound** units. The quota depends on access tier:

| Tier | Models | Session quota | Notes |
|---|---|---|---|
| **Limited mode** (outside the 25+ full-access countries, or **VPN/datacenter IPs**) | DeepSeek V4 Flash (07/31), MiMo 2.5 | **6 one-hour sessions per day** | What your web UI showed ("max 6 session avail") |
| **Full mode** | DeepSeek V4 Flash (07/31, CLI default), DeepSeek V4 Pro, GPT-5.6 Luna, MiniMax M3, MiMo 2.5, Kimi K2.7 Code | ~5 one-hour sessions/day for premium models (issue #504, 2026-04: "5 sessions of one hour each for the premium models, and currently unlimited of MiniMax M2.7") | MiniMax exempt from the 20h window |
| **GLM 5.2** (earned sessions) | z-ai/glm-5.2 | **5 per 20 hours**, enforced with HTTP 429 `rate_limited` (commit `7dd903b`, 2026-04-22) | Response carries `rateLimit` quota: "N / 5 used in last 20h" |
| Gemini 3.1 Flash Lite | google/gemini-3.1/3.5-flash-lite | specialist subagents only (file finding/research) | not user-pickable directly |

- Queued→active transitions are audited (`free_session_admit` audit log).
- Sessions are **model-locked**: a turn resolved to a different model than the session is rejected
  with `session_model_mismatch` (observed in upstream `free-agents.ts` comments). The official
  CLI admits one session per model; switching models ends/creates a session.
- **Ads no longer grant credits** (issue #504, 2026-04, jahooma: "ads no longer give credits…
  I turned it off") — kiprana's ad-impression integration is obsolete upstream.

## 2. What this means for freebuff-proxy

1. **One token ≈ 6 sessions/day (limited) or ~5/day (full premium) in practice.** Each session
   lasts 1h. Our pool creates a session lazily per token and caches it; the quota is burned by
   session CREATES, not by requests within a session. Keep usage modest.
2. **Model-bound sessions vs our shared session.** The proxy currently caches ONE session per
   token and shares it across models. Upstream rejects model switches with
   `session_model_mismatch`; the recovery matrix now treats that as session-invalid →
   invalidate + recreate + retry once (added 2026-08-11). Cost: every model switch burns a
   fresh session from the daily quota. **Future work**: cache sessions per (token, model) so a
   working session is reused until it expires.

   **UPDATE (live, 2026-08-11)**: the mismatch rejection did NOT trigger in practice. One
   session served 9+ requests across SIX different models (deepseek-v4-flash, deepseek-v4-pro,
   minimax-m3, gpt-5.6-luna, mimo-v2.5, glm-5.2) with the same instance id throughout —
   `session_model_mismatch` is enforced at the CLI planner layer, not the chat API. The shared
   per-token session design is validated live; model switching does NOT burn session quota.
3. **429 `rate_limited`** (GLM) is passed through verbatim by the error matrix (UpstreamError
   with 429 → client sees 429 + Retry-After where present). The proxy does not track the
   "N / 5 in 20h" quota yet — surfaced only in upstream error bodies.
4. **Geo-gating is tier-based, not request-based.** A datacenter/VPN IP puts the account in
   limited mode (6 sessions, 2 models) rather than outright blocking — observed live: this
   proxy on an Azure VPS IP served DeepSeek V4 Flash successfully. The 402/403 errors in the
   reference docs predate the tier rollout. `HTTP_PROXY`/`SOCKS5_PROXY` still matter for
   unlocking full mode from blocked countries.

   **UPDATE (live, 2026-08-11)**: an Indonesian VPS IP (70.153.81.129, Azure) received FULL
   access: all 12 catalog models responded 200, including premium (deepseek-v4-pro,
   gpt-5.6-luna, minimax-m3) and GLM 5.2. Indonesia is NOT on the published 25-country list
   (US/UK/EU/CA/AU/SG/NZ/IL/NO/SE/CH/LI/LU/MT/IS/IE/PT/ES/DK/FI/FR/DE/AT/BE/NL) — the list is
   expanding monthly since launch (2026-02) and/or tier follows the account, not the IP.
   Empirical rule: if a model returns 200, the account has it; the /v1/models catalog is global
   and NOT tier-filtered by this proxy.
5. **cost_mode**: upstream `model-config.ts` defines `costModes = ['free','lite','normal',
   'max','experimental','ask']` — `"free"` is a legitimate value, which supports the
   freebuff2api/kiprana evidence (PRD §8 A/B still open, but "free" is valid syntax).

## 2b. Session & workspace semantics (live-observed)

- One session (instance id) is reused across requests AND models until it expires (1h), is
  ended/superseded, or hits a waiting room. `healthz` shows the cached session state.
- "Workspace" is a CLI-side concept (per-project directory context + session bookkeeping);
  the raw API flow used by this proxy has no workspace concept — chat completions carry
  `codebuff_metadata {run_id, client_id, freebuff_instance_id}` only.
- Waiting room: `queued` → proxy returns `503 + Retry-After`; the pool's 60s maintain ticker
  advances queued sessions in the background. Not yet observed in live traffic (2026-08-11).

## 2c. Quota accounting — exact mechanics (2026-08-12, upstream source confirmed)

- **Quota is burned by the ADMISSION transition (queued→active), not by POST /session.**
  POST is an *upsert* into the `free_session` table (PK user_id): an existing active+unexpired
  row is returned as-is without burning quota (CodebuffAI/codebuff PR #598). A 429 is a
  **pre-check rejection** — it never creates an admit row, so failed attempts while
  rate-limited do NOT deepen the limit (commit `7dd903b`).
- **The quota unit is `session_units` (numeric(3,1))**, not a raw count: 1.0 per admission,
  reduced when a session ends early. Live `recentCount: 6.6` = 6 full sessions + a ~36-min
  partial. Ending a session early partially REFUNDS quota (schema comment, commit `793de91d9aaa`).
- **Scope**: limited-tier daily quota (6) is per-**account**, shared across the 2 limited
  models (the `model` field in the 429 body identifies the requested model, not a per-model
  bucket). Full-access premium is also account-shared. Separate per-model quotas exist for
  GLM (5/20h) and Gemini Pro (1/24h); MiniMax is unlimited (PRs #566/#657).
- **Agent-runs START does NOT consume session quota**; chat requests inside an active session
  cost zero. Sessions are the only burn vector.
- **429 body carries the full reset contract**: `retryAfterMs`, `resetAt` (ISO, Pacific
  midnight for the daily bucket), `limit`, `recentCount`, `period`, `accessTier`.

## 2d. Reference-implementation consensus (proxy-freebuff, quorinex, miladnoo, kiprana,
free-buff-lol, freebuff2api-optimized, freebuff2api-wokers, freebuff-proxy-hengxin, 2026-08-12)

- **GET-before-POST is universal**: every implementation polls/reuses a cached session and
  only POSTs when none exists/ended/superseded. 5s (or 60s) expiry margin is standard.
- **None of them cooldown on 429** (only 401→30min auth cooldown) — a 429-aware cooldown
  keyed by `retryAfterMs`/`resetAt` would be the first of its kind. The wokers worker parses
  `retryAfterMs` (regex, capped 6h) and hengxin surfaces `Retry-After` to clients; the
  Go/Python implementations ignore the body entirely.
- **The quota-exhaustion failure mode on 9router** (observed live 2026-08-12): our 502
  mapping → 9router fixed 30s account lock + 3 executor retries @3s → hammering. A 429 +
  Retry-After response triggers 9router's exponential backoff (2s→5min) instead — the
  correct response for "quota gone until resetAt".
- **Our proxy's quota leak (fixed 2026-08-12)**: the 60s maintain loop called
  `EnsureSession`, which POSTs a fresh session when none exists or the cached one expired
  (1h). An idle container burned ~1 session/hour of uptime — a 6/day account was exhausted
  in ~6 hours of uptime with zero traffic. Fix: maintain advances *queued* sessions only
  (GET, zero cost); session creation is lazy on first request.
- **Hot-session-first scheduling** (hengxin/wokers): when multiple tokens exist, prefer the
  token holding a live same-model session — never create on an account that already has one.
  Future work for our multi-token pool (single-token deployments are already optimal).
- **Per-(token, model) session cache** (optimized/wokers) with a 30s verify window: avoids
  re-verifying during bursts. Future work; our shared per-token session is validated live
  (one instance served 9+ requests across 6 models).

## 3. ToS constraints (freebuff.com/terms-of-service, 2026-07-23)

- Free access is per **person**, one account per person; no multi-account farming.
- Explicitly prohibited: "Access free AI model inference except through normal use of an
  official Company product… You may not call the underlying servers or endpoints directly or
  through scripts, custom clients, wrappers, integrations, or third-party software."
- "A human must initiate each session and remain actively present while it runs."
- "Hide, spoof, or misrepresent your actual country or location, including by using a proxy,
  VPN, relay…" to obtain different models/limits.
- **Verdict**: a community proxy like this one violates the letter of the ToS (undocumented
  endpoints + wrapper access). This project is explicitly educational/personal (PRD §8.6,
  README Terms of use). Account bans are possible; keep usage modest.

## 4. Open items (still worth a live A/B)

- [ ] Confirm current full-mode quota per model with a real account (5/day for premium; MiniMax
      truly unlimited? GLM 5/20h confirmed by commit).
- [ ] `COST_MODE` omit vs `"free"` — 'free' is valid upstream syntax; A/B with a real token.
- [ ] Buffy system-prompt preamble — kiprana-only claim; verify against current CLI HAR.
- [ ] Does the session create endpoint bind the model via `x-freebuff-model` header? (Our
      CreateSession sends `{}` with no model header; if binding happens, the shared-session
      design must become per-(token,model) — see §2.2.)
