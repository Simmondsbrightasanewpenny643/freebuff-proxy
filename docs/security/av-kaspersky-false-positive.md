# Kaspersky false positive on Go build artifacts

**Date**: 2026-08-11 · **Status**: validated — not an infection · **Applies to**: Windows dev machine

## Symptom

`go test ./internal/convert/` fails on this machine:

```
fork/exec C:\Users\trefeon\AppData\Local\Temp\go-buildXXXX\b001\convert.test.exe: Access is denied.
```

or, on retry:

```
open ...\convert.test.exe: Access is denied.
```

Kaspersky (the active AV; Windows Defender is disabled on this machine) quarantines the freshly
linked test binary out of the go-build cache the moment `go` writes it. The file is gone from disk
immediately after (verified: `Get-ChildItem` on the cache dir shows no `convert.test.exe`).

## What was validated (and found clean)

| Check | Result |
|---|---|
| Go toolchain integrity (`C:\Program Files\Go`) | Clean — all files carry the original install/extract timestamps (2026-07-01/09); no strays, no modified binaries |
| `go.exe` signature | Not signed — **normal for official golang.org Windows builds**; not evidence of tampering |
| `go mod verify` | `all modules verified` — go.sum matches the single dep (`golang.org/x/net`) |
| Hello-world Go binary | Builds **and runs** — Kaspersky does not flag Go binaries in general |
| Main deliverable `freebuff-proxy.exe` (9.5 MB) | Builds and exists on disk — not quarantined |
| `convert.test.exe` written via `go test -c -o <explicit path>` | Survives and **executes fine**; the same test code passes 100% when run directly |
| Source review (all of `internal/`, `cmd/`, docs) | Read in full — no malware, no obfuscation, no embedded payloads, no suspicious data blobs (grep for base64/hex blobs: none) |

Conclusion: **there is no trojan in the Go runtime, the toolchain, or this codebase.** The only
thing Kaspersky flags is one freshly-linked test binary, and it flags it inconsistently (same
content executes fine when written to a different path) — textbook heuristic false-positive
behavior, not a signature match.

## Why Kaspersky heuristics fire on this project

1. **Go binaries trip AV heuristics at a high rate** — the Go runtime is a large, distinctive
   blob (embedded runtime strings, panic handlers, syscall patterns), and Go is a popular
   language for real malware (botnets/ransomware), so heuristics are tuned suspicious of it.
2. **This project *behaves* like an evasive tool** — by design it spoofs user agents, rotates
   fingerprints, injects a CLI-envelope to pass a 403 anti-bot gate, and manages sessions/runs
   to defeat server-side rate state. AV behavioral heuristics see "proxy + fingerprint
   evasion + session management" and raise the suspicion score. That is a legitimate design
   tension to be aware of: the more "stealth" layers the proxy adds, the more AVs will flag it.
3. Test binaries additionally embed the full test source + runtime, so they can trip heuristics
   that the production binary does not.

## Workarounds (pick one)

1. **Kaspersky exclusion** (recommended for local dev): Kaspersky → Settings → Threats and
   exclusions → Exclusions → add:
   - `C:\Users\trefeon\AppData\Local\Temp\go-build*` (go build/test cache)
   - your output dirs (`dist/`, `bin/`) if you build there
2. **Bypass the blocked path**: `go test -c -o convert.test.exe ./internal/convert` then run
   `.\convert.test.exe -test.v` directly — verified working.
3. **Build/test in WSL2 or Docker** — no Kaspersky on-access scanner there; this also matches
   the planned CI setup.

## Long-term hardening (when the server slice lands)

- Add `-ldflags "-s -w"` to release builds (strips symbol/string noise that heuristics key on).
- Optionally code-sign release binaries (reduces AV suspicion further).
- If the flag persists after the project is public, submit a false-positive report via the
  Kaspersky VirusDesk / whitelisting portal with the binary hash.
- **Do not** strip or obfuscate the source to "avoid detection" — that makes the AV situation
  worse (packed/obfuscated Go is flagged far more) and hurts maintainability. The UA-rotation /
  envelope code exists for protocol fidelity (the official CLI does it), not stealth; keep it
  documented as such.
