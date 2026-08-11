# HANDOFF — freebuff-proxy (start here for a new session)

**Repo**: D:\github_repo\freebuff-proxy · **GitHub**: github.com/trefeon/freebuff-proxy (public, gh authed as trefeon)
**Project docs**: docs/product/prd.md · docs/delivery/tasks.md · docs/research/freebuff-limitations.md · docs/guides/9router-integration.md

## Where the project stands (2026-08-11)

The MVP is COMPLETE and LIVE: Go OpenAI-compatible proxy bridge for the FreeBuff free tier, wired into 9router on acerblue-local. Real chats verified end-to-end through 9router (stream + non-stream + tool calls). CI green (GitHub Actions: build, vet, `go test -race ./...` on Linux).

**Deployments**:
- **vps-01** (70.153.81.129): Docker container, real token in `~/freebuff-proxy/.env`, healthy, full model access from Indonesian IP (confirmed empirically)
- **acerblue-local** (192.168.10.3): proxy in Docker (`~/freebuff-proxy`, compose, port 3457 published), 9router container at :20128 consumes it as custom provider — **Base URL `http://172.17.0.1:3457/v1`** (9router container is on docker `bridge`; gateway 172.17.0.1). AgentRouter node in 9router should use `http://172.17.0.1:8318`.
- Systemd unit was replaced by Docker on acerblue (disabled). Networking cleaned: removed the old `9router-net` override (backed up at `~/projects/agentrouter-spoof-proxy/docker-compose.override.yml.bak`); both compose projects now use auto-created per-project networks.

## IN PROGRESS — observability pass (uncommitted, compiles clean)

Modified, NOT committed: `.gitignore` (full update), `cmd/freebuff-proxy/main.go`, `docs/product/prd.md`, `internal/config/config.go`, `internal/server/server.go`, `internal/telemetry/telemetry.go` (`git diff --stat` = 206+/56-).

Done so far:
- config: `LOG_LEVEL` field + rawConfig + Load/dotenv wiring + validation (slog.UnmarshalText) — **tests NOT yet written**
- telemetry: `New(level, logFile)` + `ParseLevel(s)`; `NewLogger` delegates — **tests NOT yet written**
- main.go: effective level (LOG_LEVEL > -v > info), `log_level` in startup summary
- server: access-log middleware (Info: method/path/status/ms/remote) with `statusWriter` (forwards Flusher/Hijacker/Pusher), `remoteHost` helper, `chat request` Debug, `chat done` Info with `relayStats{chunks,bytes}`, `writeError` restructured to single Warn log + switch mapping

Remaining (in order):
1. `internal/upstream/client.go` `do()`: Debug `"upstream ok"/"upstream error"` (method/path/status/ms/err) via slog.Default()
2. `internal/session`, `internal/runs`, `internal/pool`: lifecycle Debug lines (session created/queued/reused/invalidated/ended; run started/finished + Warn on finish failure; token cooldown; token skipped; lease acquired; waiting room surfaced)
3. `.env.example`: LOG_LEVEL section
4. Tests: config LOG_LEVEL (default empty, debug set, invalid errors, .env source); telemetry New/ParseLevel — follow config_test.go conventions (clearEnv does t.Chdir(t.TempDir()))
5. **Full user-friendly README rewrite** (user asked: "full update readme for making user easier") — sections: what/why, getting a token (scripts exist: scripts/get-freebuff-token.ps1/.sh), quick start binary + Docker (scripts/setup-proxy-docker.sh prints 9router config incl. gateway), config table (+LOG_LEVEL), 9router integration (link guide), testing, troubleshooting (+LOG_LEVEL/-v/DEBUG_DUMP/LOG_FILE), ToS. README currently exists (good base) — update, don't throw away.
6. **Repo About** (user asked): `gh api repos/trefeon/freebuff-proxy -X PATCH -f description="..." -f 'topics[]=go' -f 'topics[]=proxy' ...` (description + topics; no website)
7. tasks.md: one line for the observability pass in tooling notes
8. Verify: build/vet, tests (convert via `go test -c` workaround), smoke with LOG_LEVEL=debug
9. Commit + push (CI watches), then deploy: `ssh acerblue-local "cd ~/freebuff-proxy && git pull && docker compose up --build -d"`

## Environment quirks (critical)

- **Kaspersky AV** blocks freshly-linked test binaries: `go test ./internal/convert/` fails "Access is denied" (false positive, docs/security/av-kaspersky-false-positive.md). Workaround: `go test -c -o C:\Users\trefeon\AppData\Local\Temp\opencode\gotool-test\convert.test.exe ./internal/convert` + run that exe with `-test.count=1`. Set `$env:GOTMPDIR = "C:\Users\trefeon\AppData\Local\Temp\opencode\gotmp"` before go commands.
- **rtk wrapper** breaks `-race` and some args — use raw `go` (or `& "C:\Program Files\Go\bin\go.exe"`).
- **No gcc** → `-race` is CI-only.
- **Subagents**: `general` agent type WORKED (all build slices); `implementor`/`build`/`fixer` return empty results consistently (read but never write). If `general` is rejected as unknown, implement directly — do not waste dispatches on implementor/build/fixer.
- SSH quoting from PowerShell mangles JSON/`$` — use base64-encoded scripts (`[Convert]::ToBase64String` + `echo <b64> | base64 -d | bash`).

## Secrets (do NOT print/log — they live in gitignored .env files)

- Real FreeBuff token: `C:\Users\trefeon\.config\manicode\credentials.json` → `default.authToken` (36-char UUID); also in repo `.env` (local) + both servers' `.env`. Scripts/README examples use fake values — never confuse them.
- gh token in keyring (repo+workflow scopes).

## Suggested skills

- `claude-handoff` — this file's format (for future handoffs)
- `engram` — mem_search/mem_context before deep work; mem_session_summary at end
- `source-driven-development` — if touching upstream protocol behavior
- `design-standards` — only if the README gets visual treatment (keep it plain markdown)

## Test/demo recipes (no code changes needed)

```bash
# live proxy check (any host): healthz + models
curl -s http://127.0.0.1:3457/healthz
curl -s http://127.0.0.1:3457/v1/models
# chat through the proxy (base64 body to avoid quoting)
echo <b64-json> | base64 -d > /tmp/c.json && curl -s -N http://127.0.0.1:3457/v1/chat/completions -H 'Content-Type: application/json' -d @/tmp/c.json
```
