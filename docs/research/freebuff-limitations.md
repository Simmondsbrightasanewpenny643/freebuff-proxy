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

## 2e. The "3 projects per day" composer limit (2026-08-12) — SEPARATE from the API quota

- The Freebuff **Web composer** (freebuff.com/web, formerly "Vly") and **Cloud** (freebuff.com/cloud)
  are separate products from the CLI. The web app-builder shows *"your region is limited to
  3 projects per day — 0/3"* for accounts in limited-tier regions (observed live, Indonesia).
- This is a **project-creation quota for the app-builder only** — it does NOT gate the CLI
  chat/session/agent-run endpoints our proxy uses, and it does NOT share the 6-session/day
  budget. The CLI shows its own counter ("6.6 of 6 sessions used") while the web shows 0/3
  projects — independent counters, same account.
- Not documented publicly (no "3 projects" constant in CodebuffAI/codebuff; the app-builder
  backend is not open-sourced). Closest official statement: Freebuff Cloud blog (2026-07-01)
  — *"some regions are temporarily limited to one active project and one new project per day"*.
  The "3" may be a newer value or a Web-composer-specific tier.
- Tier detection for the composer is IP-based (Cloudflare/GeoIP) like the session tier, but the
  project limit is enforced per-account after browser login. A SOCKS5 proxy may lift the *tier*
  (more models), but the project-creation cap likely follows the account.
- Referral/earn (freebuff.com/earn) unlocks extra CLI **sessions** (e.g. GLM 5.2), not composer
  projects. ToS still says one account per person; no farming.
- **Proxy impact: none.** Our proxy consumes only the CLI session quota. No code changes
  needed; this is a third quota dimension users hit only when using the web/cloud products
  directly.

## 2f. Upstream intelligence from GitHub issues/PRs (2026-08-12, lib-3)

- **Ban responses**: upstream returns `403 {"status":"banned"}` with a `resumes_at` timestamp
  (Quorinex/Freebuff2API#8, XxxXTeam/freebuff2api#11). Heavy continuous usage (esp. V4 Flash
  left running) triggers temporary bans; multiple accounts compound the risk. Bans are
  temporary but have no documented duration beyond `resumes_at`. Our proxy does NOT detect
  this yet — it falls through as a generic upstream error. **TODO: `ErrBanned` sentinel +
  cooldown until `resumes_at`.**
- **Country/tier gating**: `FREE_MODE_ALLOWED_COUNTRIES` is a hardcoded allowlist (US, CA, UK,
  select EU) — Indonesia is NOT on it, which is why our account is `limited`. Detection is
  Cloudflare `cf-ipcountry` + x-forwarded-for + ipinfo privacy signals. Blocked signals:
  `anonymous`, `vpn`, `proxy`, `tor`, `relay`, `res_proxy` (403 `country_blocked` /
  `anonymous_network`); `hosting`/`service` are NOT blocked post-commit b4367ac. A later
  commit (0f331ed) made VPN traffic land in `limited` mode instead of hard block. The session
  response carries `accessTier`, `countryCode`, `countryBlockReason`, `ipPrivacySignals` —
  our `SessionState` ignores them. **TODO: parse + log for diagnostics.**
- **Session statuses**: the type surface includes `banned`, `country_blocked`, `rate_limited`,
  `model_locked`, `superseded` in addition to the ones we handle. `model_locked` = active
  session bound to a different model. **TODO: handle these explicitly in the session machine.**
- **Session fights**: two instances sharing one auth token fight over the session — the last
  POST supersedes the previous (`status: "superseded"`); the loser gets session-invalid until
  it re-creates (CodebuffAI/freebuff#897). Running the official CLI + our proxy on the same
  token will thrash sessions. Constraint to document, not a bug.
- **Waiting room is permanent**: PR #667 removes the waiting-room env-var toggle; the session
  gate is always on, and the `disabled` status is becoming a 404-compat fallback. Our 503 +
  Retry-After handling is the right shape.
- **`developer` role**: upstream accepts only `system/user/assistant/tool/latest_reminder`;
  `developer` → 400 (XxxXTeam/freebuff2api#3). Our `convert` normalizes `developer`→`system`
  — confirmed correct.
- **Quota transparency**: session responses carry `rateLimitsByModel` (`model`, `limit`,
  `period` pacific_day|pacific_week, `resetAt`, `recentCount`) and `entitlementBreakdown`
  (`base`, `referral`, `streak`, `promo`). GLM 5.2 has its own referral-gated pool; premium
  and limited models have separate pools. **Idea: surface remaining quota in healthz.**
- **Envelope injection is the critical anti-block measure** — without the CLI envelope
  (`codebuff_metadata`, `x-freebuff-model`, `x-freebuff-instance-id`, `data_collection:deny`,
  `stream:true`, `cb_easp`), upstream returns `403 free_mode_cli_required`. Confirmed our
  implementation matches; if it ever drifts this error names it explicitly.
- **No headless CLI**: freebuff CLI is TUI-only (no `-p`/`--print`/`--json`); the `authToken`
  is a device-OAuth token (`~/.config/manicode/credentials.json`), and `@codebuff/sdk`
  expects a paid `CODEBUFF_API_KEY` instead. Our token+HTTP approach is the only programmatic
  path; `fingerprintId`/`fingerprintHash` exist in credentials but are not yet validated by
  the chat API.

## 2g. Anti-block hardening techniques (2026-08-12, 16 newly cloned repos)

Surveyed marktantongco/freebuff-proxy, freebuff-reverse, freebuff-bridge,
freebuff2api-rs, freebuff2api-lza6, freebuff2cloudflare-api, codebuff-fork, and 9 more.
Our proxy ALREADY has: CLI envelope injection, GET-before-POST session reuse, quota-safe
maintenance, 429/ban/cooldown handling, Retry-After surfacing, waiting-room 503, tier
diagnostics. Techniques we do NOT have (ranked by value/risk):

1. **JA3/TLS fingerprint impersonation** (HIGH value, HIGH risk) — marktantongco uses
   `github.com/refraction-networking/utls` (Chrome120/Safari17/Firefox120 profiles +
   `HelloRandomized`), freebuff-reverse uses `github.com/bogdanfinn/tls-client` (100+
   profiles, random TLS extension order, protocol racing). Plain Go `net/http` has a unique
   ClientHello fingerprint. We currently pass the gate WITHOUT it — adds a dependency and
   transport complexity; only worth it if upstream starts fingerprinting TLS.
2. **Header sanitization + browser headers** (MEDIUM, LOW risk) — marktantongco strips 27
   proxy-identifying headers (X-Forwarded-For, Via, CF-Connecting-IP, True-Client-IP) and
   injects 11 browser-typical ones (Sec-CH-UA, Sec-Fetch-Site/Mode/Dest,
   Upgrade-Insecure-Requests). We construct upstream requests fresh (nothing forwarded), so
   stripping is moot; injecting Sec-* realism is a cheap optional add.
3. **Session create gate** (LOW, LOW) — freebuff-reverse limits concurrent session creates
   (global 128 / per-key 32 / per-model 32 / per-group 96) to prevent thundering herds. We
   have single-flight refresh per token already; multi-token herds are bounded by token count.
4. **Per-session concurrency semaphore** (LOW) — freebuff-reverse caps concurrent chats per
   session with wait-or-reject backpressure. We already allow concurrent leases on one
   session (validated live: 9+ requests on one instance).
5. **Synthetic device fingerprint** (MEDIUM, UNKNOWN risk) — fatmuh generates
   `enhanced-{sha256}` fingerprints with real device catalogs (MacBook M4, ThinkPad X1),
   real OUI MACs, locally-administered bit, seeded per account. The CLI sends these; we
   don't. Upstream doesn't currently reject us for their absence — adding unproven fields
   could just as easily trigger the gate.
6. **Desktop orchestrator protocol** (N/A) — freebuff-bridge reverse-engineered the Desktop
   app's local REST+SSE API (thread messages, sessionSlots `{limit,used,holders}`,
   quotaByModel, accessTier). Insight only: the desktop app itself shows session slot limits
   and per-model quotas in its state.
7. **Timing jitter** (LOW) — 50-300ms inter-request jitter + "reading time" simulation.
   Adds latency for no measured benefit on a session-quota-gated service.
8. **ResponseClass typed classification** (N/A) — freebuff-reverse classifies responses
   Ok/RateLimited/AuthExpired/Retryable/Fatal at the transport layer. Equivalent to our
   typed sentinels (RateLimitError/BanError/UpstreamError).

**Official constants spotted** (codebuff-fork → wokers): `FREEBUFF_PREMIUM_SESSION_LIMIT=6`,
`FREEBUFF_WEB_STANDARD_SESSION_LIMIT=6`, `FREEBUFF_DESKTOP_SESSION_LIMITS`
(premium:1, unlimited:3 slots), `FREEBUFF_ROOT_AGENT_ID_BY_MODEL` (source of truth for the
model→agent map our registry parses).

**Verdict**: nothing in this batch is a must-have — our proxy already implements the
quota-critical patterns, and the remaining techniques are either risky (TLS/fingerprint
spoofing), low-value (gates/semaphores/jitter), or N/A (desktop protocol). Revisit JA3 if
upstream ever starts rejecting our plain-Go ClientHello.

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
