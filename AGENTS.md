# AGENTS.md

## Project summary

This repository contains a portable Dockerized AWG gateway with SOCKS5 proxy.

Main runtime path:
- `entrypoint.sh` calls `awg-quick up` with mounted config (`/config/amnezia.conf`)
- If kernel AWG interface type is unavailable, `awg-quick` falls back to `amneziawg-go`
- `microsocks` listens on all interfaces and serves SOCKS5 traffic

## Current architecture

- Runtime base: Alpine
- AWG userspace backend: `amneziawg-go` (required for Windows Docker Desktop portability)
- AWG tools: `awg`, `awg-quick`
- Proxy: `microsocks`
- Init process: `tini`

## Known design decisions

1. Keep portable mode first
- Do not remove `amneziawg-go` from default image.
- This is necessary to keep Windows/macOS Docker Desktop working.

2. Runtime config sanitization
- `entrypoint.sh` writes a temporary runtime config in `/tmp/<iface>.conf`.
- Empty assignments (for example `I2 =`) are removed before `awg setconf`.

3. DNS helper shim
- Runtime includes a minimal `resolvconf` shim script to satisfy `awg-quick` DNS hook.

4. Desktop sysctl tolerance
- `awg-quick` is patched in image so `src_valid_mark` sysctl failure does not crash startup.

## Operational defaults

- Mounted config: `/config/amnezia.conf`
- Compose default proxy port: `1081`
- Proxy bind host: `0.0.0.0`

## Verification checklist (quick)

1. Build and run:
- `docker compose up --build -d`

2. Container health:
- `docker compose ps`
- Status should be `Up` and expose `1081` by default.

3. Runtime logs:
- `docker compose logs --tail=120 awg-proxy`
- Expect AWG startup lines and `Starting microsocks`.

4. Proxy smoke test:
- `curl.exe --socks5-hostname 127.0.0.1:1081 https://api.ipify.org`

5. Optional tunnel evidence:
- `docker exec awg-proxy awg show`

## Guardrails for future agents

- Prefer minimal, incremental edits.
- Preserve portable behavior unless user explicitly asks for Linux-only profile.
- If introducing kernel-only optimization, implement as separate profile/target, not replacement.
- Re-run compose startup and logs checks after changing Dockerfile or entrypoint.
