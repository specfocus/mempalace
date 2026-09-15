# MemPalace DigitalOcean Droplet

Created manually in the DigitalOcean UI (not via API).

## Droplet

| Field | Value |
| --- | --- |
| Name | `ubuntu-s-1vcpu-2gb-nyc1` |
| Status | Active |
| Image | Ubuntu 24.04 (LTS) x64 |
| Plan | Basic — 1 vCPU, 2 GB RAM, 50 GB SSD (~$12.00/mo, $0.018/hr) |
| Region | NYC1 (New York 1) |
| Project | `first-project` |
| VPC | `default-nyc1` |
| Public IPv4 | `64.227.5.215` |
| Private IPv4 | `10.116.0.2` |

## SSH

Key that works: `C:\Users\lucas\.ssh\id_ed25519_droplet`

`ash
ssh mempalace
# or
ssh -i ~/.ssh/id_ed25519_droplet root@64.227.5.215
`

SSH config alias `mempalace` / `64.227.5.215` points at that key (`IdentitiesOnly yes`). Resize later in the DO control panel if 2 GB gets tight.

## Deploy target

Stack: Docker Compose team server from `deploy/docker-compose.server.yml` (MemPalace + Qdrant on port `8765`), with a bearer token in `deploy/.env`. Prefer TLS via a reverse proxy if exposed beyond a private network.

## Notes

- Only Lucas's agents write/read memory on this host.
- Upstream image default: `ghcr.io/mempalace/mempalace:latest` (fork is `specfocus/mempalace` on `develop` if a custom image is needed later).


## Runtime (deployed)

| Field | Value |
| --- | --- |
| Host path | `/opt/mempalace` |
| Compose file | `deploy/docker-compose.server.yml` |
| Env file | `/opt/mempalace/deploy/.env` (mode 600; also local `deploy/.env`, gitignored) |
| MCP URL | `http://64.227.5.215:8765/mcp` |
| Health | `http://64.227.5.215:8765/healthz` |
| Image | `ghcr.io/mempalace/mempalace:latest` + `qdrant/qdrant:latest` |
| Swap | 2 GB `/swapfile` |
| Docker | 29.8.0 + Compose v5.5.1 |

### Useful commands

```bash
ssh mempalace
cd /opt/mempalace
docker compose -f deploy/docker-compose.server.yml ps
docker compose -f deploy/docker-compose.server.yml logs -f
curl http://127.0.0.1:8765/healthz
```

Bearer token lives only in `deploy/.env` (`MEMPALACE_MCP_HTTP_TOKEN`). Do not commit it. TLS/Caddy still TODO if exposing beyond a trusted network.

## Migration (WSL palace -> Droplet)

Copied WSL `/home/lucas/.mempalace` (Chroma) onto the Droplet on 2026-09-10.

| Field | Value |
| --- | --- |
| Backend now | Chroma (replaced empty Qdrant stack) |
| Compose file | `/opt/mempalace/deploy/docker-compose.chroma.yml` |
| Data bind-mount | `/opt/mempalace/data` → container `/data` |
| Palace path in container | `/data/.mempalace/palace` |
| Drawer count verified | 1597 (matches WSL) |
| Archive kept | `/opt/mempalace/mempalace-home.tgz` |

Local WSL palace was restarted after the tar; both sides independently hold the same snapshot from migrate time.

## Ops fixes (2026-09-10)

### Droplet write failure (`/data/.cache` EACCES)
Container runs as uid ``1000`` (``mempalace``). Bind-mount parent ``/opt/mempalace/data`` was ``root:root``, so creating ``/data/.cache`` failed while ``/healthz`` still returned ``ok``.

Fix applied:
```bash
chown -R 1000:1000 /opt/mempalace/data
```
Verified: ``mempalace_checkpoint`` succeeded after restart.

After any future extract/tar into ``/opt/mempalace/data``, re-run that chown before ``compose up``.

### Integrity
``PRAGMA integrity_check`` on Droplet ``chroma`` / ``knowledge_graph`` / ``logstream``: all ``ok``.
Local WSL same three DBs: all ``ok``. Local write probe also succeeded after re-check (earlier -32002 may have been transient around the migrate stop/start).

## Design note (next agent)

**Write-readiness / `/readyz` probe (and honest `-32002` messages):** see [`docs/readyz-write-probe.md`](docs/readyz-write-probe.md).

Measured 2026-09-10: Droplet `Errno 13` on `/data/.cache` and local `-32002` while `/healthz` stayed green. No GitHub issue filed (`gh` not installed here); this file is the canonical handoff until a PR lands after on-demand usage is enabled.
