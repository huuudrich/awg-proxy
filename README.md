# AWG Proxy (Portable Alpine)

Containerized VPN gateway that establishes an AmneziaWG tunnel and exposes a SOCKS5 proxy.

Traffic flow:
- Client -> SOCKS5 proxy (`microsocks`)
- Proxy process -> container network stack
- Container routing policy -> AWG tunnel (`awg-quick` + `amneziawg-go` userspace fallback)

This project is designed to work on Windows Docker Desktop and Linux.

## What is included

- Base image: Alpine (portable variant)
- AWG userspace backend: `amneziawg-go`
- AWG tooling: `awg`, `awg-quick`
- Proxy: `microsocks`
- Entrypoint orchestration: `entrypoint.sh`

## Requirements

- Docker Engine / Docker Desktop
- Docker Compose v2
- `NET_ADMIN` capability
- `/dev/net/tun` device mapping
- AWG client config mounted to `/config/amnezia.conf`

## Quick start

1. Put your AWG config into `amnezia.conf` in project root.
2. Start service:

```powershell
docker compose up --build -d
```

3. Check status:

```powershell
docker compose ps
docker compose logs --tail=120 awg-proxy
```

4. Use SOCKS5 proxy on host:

- Address: `127.0.0.1`
- Port: `1080` (default)

Example:

```powershell
curl.exe --socks5-hostname 127.0.0.1:1080 https://api.ipify.org
```

## Configuration

Compose publishes a configurable port:

- `PROXY_PORT` (default `1080`)

Supported environment variables:

- `AWG_CONFIG_FILE` (default `/config/amnezia.conf`)
- `WG_QUICK_USERSPACE_IMPLEMENTATION` (default `amneziawg-go`)
- `LOG_LEVEL` (default `info`)
- `PROXY_LISTEN_HOST` (default `0.0.0.0`)
- `PROXY_PORT` (default `1080`)
- `PROXY_USER`, `PROXY_PASSWORD` (optional auth, must be set together)
- `MICROSOCKS_BIND_ADDRESS` (optional)
- `MICROSOCKS_WHITELIST` (optional)
- `MICROSOCKS_AUTH_ONCE` (`0` or `1`)
- `MICROSOCKS_QUIET` (`0` or `1`)
- `MICROSOCKS_OPTS` (extra flags)

## Notes about AWG config

- File name must end with `.conf`.
- `AllowedIPs` should include default routes if you want all proxy traffic to go through VPN:
  - `0.0.0.0/0`
  - `::/0`
- Empty assignments like `I2 =` are sanitized at runtime by `entrypoint.sh` into a temporary config.

## Platform behavior

- Windows Docker Desktop: expected to use userspace fallback (`amneziawg-go`).
- Linux with kernel module installed: `awg-quick` may use kernel path first.

## Current tested status

Validated in this repository session:

- `docker compose up --build -d` succeeds
- Container is `Up` and port mapping is active
- AWG interface is configured and proxy accepts outbound connections
- Alpine image size: about `26.5MB`

Note on IP checks:
- If direct and proxied external IP are identical, your host may already egress through the same provider/VPN path.
- In that case rely on container logs (`client[...] connected to ...`) and AWG counters (`awg show`) to verify tunnel activity.

## Troubleshooting

- `/dev/net/tun is missing`
  - Ensure `devices: - /dev/net/tun:/dev/net/tun` is present in compose.

- `Line unrecognized: I2=`
  - Fixed by runtime sanitization in `entrypoint.sh`. Use the current image.

- `sysctl: permission denied on key net.ipv4.conf.all.src_valid_mark`
  - Expected in some Docker Desktop environments.
  - Current image tolerates this and continues startup.

- Proxy port busy
  - Override host/container port via `PROXY_PORT`.

## Files

- `Dockerfile` - multi-stage Alpine portable build
- `entrypoint.sh` - AWG startup and proxy orchestration
- `docker-compose.yml` - capabilities, tun mapping, env defaults
