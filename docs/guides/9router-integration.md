# Wiring freebuff-proxy into 9router

**Goal**: make 9router (and any OpenAI client) use FreeBuff's free models through
freebuff-proxy. Your FreeBuff auth token acts as the credential behind the proxy; 9router sees
an ordinary **OpenAI-compatible custom provider**.

```
9router (localhost:20128)
   │  /v1/chat/completions  (Bearer api_key, model "freebuff/deepseek-v4-flash")
   ▼
freebuff-proxy (:3457, OpenAI-compatible surface)
   │  x-freebuff-model / x-freebuff-instance-id / codebuff_metadata envelope
   ▼
codebuff.com (FreeBuff free tier — token-bound)
```

---

## 1. Prerequisites — the proxy must be running and reachable

1. Get a token (see README → *Getting a token*: `freebuff.llm.pm` or
   `~/.config/manicode/credentials.json`).
2. Run freebuff-proxy **with the token** (any of these):
   - **Same machine as 9router (recommended):** build + run, or the systemd unit below
   - **Docker:** `docker compose up --build -d` (the compose file sets `LISTEN_ADDR=:3457`
     and publishes port 3457 on the host)
   - **Remote host / VPS:** run it there and open the firewall/NSG for port 3457
3. Verify before touching 9router:

   ```bash
   curl http://127.0.0.1:3457/healthz     # {"models":12,"tokens":[...],"uptime_seconds":N}
   curl http://127.0.0.1:3457/v1/models   # the 12-model catalog
   ```

**Base URL — which one to use in 9router:** the proxy listens on port **3457**; only the
*host* part of the URL changes:

| Where the proxy runs | Base URL in 9router |
|---|---|
| **Same machine as 9router** — binary **or** Docker Compose (default setup) | `http://127.0.0.1:3457/v1` |
| **Another machine on your LAN** (e.g. a homelab box) | `http://<that-host-ip>:3457/v1` (e.g. `http://192.168.10.3:3457/v1`) |
| **A VPS / remote server** | `http://<vps-ip-or-domain>:3457/v1` (firewall must allow 3457; if the container binds loopback, set `LISTEN_ADDR=:3457` in its env) |

**When in doubt, use `http://127.0.0.1:3457/v1`** — it is correct whenever the proxy runs on
the same machine as 9router, which is the default for both the binary and `docker compose up`.

Example systemd unit (Ubuntu/Debian, same host as 9router):

```ini
[Unit]
Description=freebuff-proxy (FreeBuff OpenAI-compatible bridge)
After=network-online.target

[Service]
Type=simple
User=<your-user>
WorkingDirectory=/home/<your-user>/freebuff-proxy   # .env auto-loads from here
ExecStart=/home/<your-user>/freebuff-proxy/freebuff-proxy
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
sudo cp freebuff-proxy.service /etc/systemd/system/ && sudo systemctl enable --now freebuff-proxy
```

---

## 2. Install 9router

**Option A — npm (quickest):**

```bash
npm install -g 9router
9router          # dashboard opens at http://localhost:20128
```

**Option B — from source:**

```bash
git clone https://github.com/decolua/9router && cd 9router
cp .env.example .env          # then set JWT_SECRET and INITIAL_PASSWORD (default is 123456!)
npm install
npm run build
PORT=20128 HOSTNAME=0.0.0.0 npm run start
```

Data lives in `~/.9router/` (SQLite). Default port: **20128** (dashboard `/dashboard`, API `/v1`).
**Set `INITIAL_PASSWORD` and `JWT_SECRET`** — the default password is a known value.

---

## 3. Add freebuff-proxy as a provider (dashboard)

1. Open **http://localhost:20128/dashboard/providers**
2. Under **Custom Providers (OpenAI/Anthropic Compatible)** click **Add OpenAI Compatible**
3. Fill the form **exactly** (verified against the 9router source, `AddCompatibleModal.js`):

   | Field | Value | Why (from 9router source) |
   |---|---|---|
   | **Name** | `freebuff` | Required. Friendly label only. |
   | **Prefix** | `freebuff` | Required. Becomes the model-id prefix: model combos are `freebuff/<model-id>` |
   | **API Type** | **Chat Completions** | Keep the default `chat`. The proxy implements `/v1/chat/completions` only — **Responses API is NOT supported**; choosing it makes every request 404 |
   | **Base URL** | **default: `http://127.0.0.1:3457/v1`** (see the host table in §1; the port is always 3457) | Must end in `/v1` — 9router appends `/models`, `/chat/completions` to it. Correct for binary **and** Docker Compose on the same machine as 9router; use `http://<host-ip>:3457/v1` only when the proxy is on a different machine |
   | **API Key (for Check)** | any non-empty value (e.g. the FreeBuff token) | Used ONLY by the **Check** button. The **Create** button does not require it. With an empty proxy `API_KEYS`, any value passes |
   | **Model ID (optional)** | **leave empty** | Only for providers *without* a `/models` endpoint (falls back to a chat-completions inference test). The proxy has `GET /v1/models` — the /models check succeeds and enumerates all 12 models |
   | **Check** button | click it → expect a green **Valid** badge | 9router POSTs `/api/provider-nodes/validate` → `GET {base}/models` with your key, 10s timeout. Green = proxy reachable |
   | **Create** button | click it | POSTs `/api/provider-nodes` `{name, prefix, apiType:"chat", baseUrl, type:"openai-compatible"}` |

4. After create, open the node and add the models from `/v1/models` (see §4); each is addressed as
   `freebuff/<model-id>` (e.g. `freebuff/deepseek-v4-flash`).

Equivalent raw config shape (for config-file or headless setups — 9router's custom provider
store is the same object it persists in its DB):

```json
{
  "freebuff": {
    "base_url": "http://127.0.0.1:3457/v1",
    "api_key": "any-placeholder-or-your-API_KEYS-value",
    "models": ["deepseek/deepseek-v4-flash"]
  }
}
```

---

## 4. Model catalog (live, 2026-08-11)

Served by `GET /v1/models` (parsed from `CodebuffAI/codebuff` TS sources, refreshed every 6h,
fallback at boot). Register any subset in 9router:

| Model ID | Notes |
|---|---|
| `deepseek/deepseek-v4-flash` | CLI default; full **and** limited access; fastest |
| `deepseek/deepseek-v4-pro` | full access; deeper reasoning |
| `openai/gpt-5.6-luna` | full access; deep reasoning + image support |
| `minimax/minimax-m3` | full access; fast + image support |
| `mimo/mimo-v2.5` | full **and** limited access |
| `z-ai/glm-5.2` | earned sessions; **rate-limited to 5 sessions / 20h** (HTTP 429 `rate_limited`) |
| `crof/kimi-k3-eco`, `anthropic/claude-fable-5`, `meta/muse-spark-1.2-contributor` | catalog additions, may be restricted per tier |
| `google/gemini-2.5/3.1/3.5-flash-lite` | specialist subagents (file finding/research) — not a general chat model |

Quota reality (see `docs/research/freebuff-limitations.md`):
- **Limited mode** (some regions / VPN / datacenter IPs): DeepSeek V4 Flash + MiMo 2.5 only,
  6 one-hour sessions/day.
- **Full mode**: all models; ~5 one-hour sessions/day for premium models (MiniMax unlimited,
  GLM 5/20h). One proxy session serves many requests across models, so a normal day burns 1–3.
- The proxy does **not** tier-filter `/v1/models` — a model that errors upstream is a
  tier/quota issue, not a proxy bug.

---

## 5. Verification

Through 9router (the model id carries the provider prefix):

```bash
curl -N http://localhost:20128/v1/chat/completions \
  -H "Authorization: Bearer <9router-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"freebuff/deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"stream":true}'
```

Also verify in the 9router dashboard chat: pick `freebuff/deepseek-v4-flash` and send a message.

**9router settings that matter:**
- **RTK token saver** — leave enabled (it compresses `tool_result` before it reaches the
  proxy; saves FreeBuff quota too).
- **max_tokens**: reasoning models think before they answer — set a generous
  `max_tokens` (≥ 4k) for `deepseek-*`, `gpt-5.6-luna`, `glm-5.2` or tool calls get truncated
  (`finish_reason: "length"`, observed live).
- **Fallback tiers**: freebuff fits a FREE tier (Tier 3) under your paid providers in 9router
  combo/fallback chains.

---

## 6. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| 9router: connection refused on base_url | Proxy not running, wrong port, or firewall. Check `systemctl status freebuff-proxy` / `docker compose ps`; `curl http://127.0.0.1:3457/healthz` |
| 401 from the proxy | `API_KEYS` is set in the proxy `.env` and 9router's api_key doesn't match one of them |
| `400 model_not_found` (proxy) | Model not in the registry catalog — check `/v1/models` |
| `404 unknown model` (9router) | The model combo isn't registered — re-add the model in the provider config |
| `503 waiting_room_queued` + Retry-After | FreeBuff waiting room (quota/hourly). Normal; 9router/opencode retry automatically |
| `429` with `rate_limited` | GLM 5/20h cap — switch model or wait |
| `502 upstream_unavailable` | Token in 30-min cooldown after a 401, or all tokens failed — check `healthz` |
| Model streams `reasoning_content` | By design (CLI-faithful). 9router handles it; don't strip it |
| 9router shows provider disconnected | Proxy restarted mid-flight; sessions recover transparently on the next request |

---

## 7. References

- This project: `docs/product/prd.md`, `docs/research/freebuff-reference-analysis.md`,
  `docs/research/freebuff-limitations.md`, README (getting a token, config table)
- 9router: github.com/decolua/9router (`npm i -g 9router`, dashboard on :20128, data in
  `~/.9router/`)
- ToS note: FreeBuff's terms prohibit third-party wrappers/endpoints — educational/personal
  use only, bans possible. Keep usage modest.
