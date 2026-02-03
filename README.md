# Public Gateway

Isolated Docker Compose stack for public-facing web applications. This stack is completely separate from the private AI services in `~/vps`.

## Architecture

```
Internet → Cloudflare Edge → cloudflared → nginx:80 (internal) → Containers
                                              ↑
            Direct access: ${TAILSCALE_IP}:8880 (Tailnet only)
```

**Security:** Nginx is only exposed on the Tailscale IP (${TAILSCALE_IP}:8880). Public access comes through Cloudflare Tunnel, which connects to nginx via Docker's internal network.

## Services

| Service | Domain | Description |
|---------|--------|-------------|
| cd-player | cd.ilgailu.com | Vinyl record visualizer |
| cigarspot | cigarspot.ilgailu.com | Cigar inventory app |
| be-mine | kiki.ilgailu.com | Valentine's meme site |

## Setup

1. Copy the example environment file:
   ```bash
   cp .env.example .env
   ```
2. Edit `.env` and set your values:
   - `TAILSCALE_IP` - Your VPS Tailscale IP (e.g., 100.x.x.x)
   - `CLOUDFLARE_TUNNEL_TOKEN` - Your Cloudflare tunnel token

## Commands

### Start services
```bash
cd ~/public-gateway
docker compose up -d
```

### Stop services
```bash
docker compose down
```

### View logs
```bash
docker logs cd-player
docker logs cigarspot
docker logs be-mine
docker logs public-nginx
docker logs public-cloudflared
```

### Rebuild after code changes
```bash
docker compose build --no-cache
docker compose up -d
```

### Test via Tailscale (direct)
```bash
curl -H "Host: cd.ilgailu.com" http://${TAILSCALE_IP}:8880
curl -H "Host: cigarspot.ilgailu.com" http://${TAILSCALE_IP}:8880
curl -H "Host: kiki.ilgailu.com" http://${TAILSCALE_IP}:8880
```

### Test via Cloudflare (public)
```bash
curl https://cd.ilgailu.com
curl https://cigarspot.ilgailu.com
curl https://kiki.ilgailu.com
```

## Adding a New Public App

1. Add service definition to `docker-compose.yml`
2. Add server block to `nginx.conf`
3. Configure Cloudflare Tunnel hostname in Cloudflare Zero Trust dashboard
4. Create DNS CNAME record pointing to tunnel

## Security Notes

- Nginx bound to Tailscale IP only (${TAILSCALE_IP})
- No public internet exposure except via Cloudflare Tunnel
- Static or self-contained apps only (no shared databases)
- Separate Docker network from VPS AI stack
- Minimal resource allocations

## Cloudflare Tunnel Configuration

The tunnel token is stored in `.env`. In Cloudflare Zero Trust dashboard:

1. Go to Networks → Tunnels
2. Select your tunnel
3. Add public hostnames with these settings:
   - `cd.ilgailu.com` → Service: `http://nginx:80`
   - `cigarspot.ilgailu.com` → Service: `http://nginx:80`
   - `kiki.ilgailu.com` → Service: `http://nginx:80`

Note: The tunnel connects to nginx via Docker's internal network using the container name `nginx`.
