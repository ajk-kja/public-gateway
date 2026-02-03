# Public Gateway

Isolated Docker Compose stack for public-facing web applications. This stack integrates with the VPS Caddy reverse proxy for Cloudflare Tunnel access.

## Architecture

```
Internet → Cloudflare Edge → cloudflared (VPS) → Caddy:80 → public-nginx:8585 → Containers
                                                    ↓
                          (ai/flowise/litellm/nextcloud handled directly by Caddy)
```

**How it works:**
1. The VPS `cloudflared` container (in `~/vps` stack) handles the Cloudflare Tunnel
2. VPS Caddy receives requests on port 80 and routes based on Host header
3. Requests for `cd/cigarspot/kiki.ilgailu.com` are proxied to `public-nginx:8585`
4. Public-nginx routes to the appropriate app container

**Network Integration:**
- `public-nginx` joins both `public-gateway` and `localai_default` networks
- This allows VPS Caddy to reach nginx while keeping app containers isolated

## Services

| Service | Domain | Description |
|---------|--------|-------------|
| cd-player | cd.ilgailu.com | Vinyl record visualizer |
| cigarspot | cigarspot.ilgailu.com | Cigar inventory app |
| be-mine | kiki.ilgailu.com | Valentine's meme site |

## Prerequisites

The VPS stack (`~/vps`) must be running with:
- `caddy` container on `localai_default` network
- `cloudflared` container connected to Cloudflare Tunnel
- Caddyfile configured with routes for public-gateway domains

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
```

### Rebuild after code changes
```bash
docker compose build --no-cache
docker compose up -d
```

### Test via Tailscale (simulating Cloudflare)
```bash
# Test through VPS Caddy (port 80)
curl -H "Host: cd.ilgailu.com" -H "CF-Ray: test" http://100.123.123.5:80
curl -H "Host: cigarspot.ilgailu.com" -H "CF-Ray: test" http://100.123.123.5:80
curl -H "Host: kiki.ilgailu.com" -H "CF-Ray: test" http://100.123.123.5:80
```

### Test via Cloudflare (public)
```bash
curl https://cd.ilgailu.com
curl https://cigarspot.ilgailu.com
curl https://kiki.ilgailu.com
```

## Adding a New Public App

1. Add service definition to `docker-compose.yml` (on `public-gateway` network)
2. Add server block to `nginx.conf` (listening on port 8585)
3. Add route to VPS `Caddyfile` in the `:80` block:
   ```caddy
   @newapp host newapp.ilgailu.com
   handle @newapp {
       import cloudflare_or_tailscale
       reverse_proxy public-nginx:8585
   }
   ```
4. Reload Caddy: `docker exec caddy caddy reload --config /etc/caddy/Caddyfile`
5. Configure Cloudflare DNS CNAME pointing to tunnel

## Security Notes

- Apps are only accessible via Cloudflare Tunnel or Tailscale (enforced by Caddy)
- No direct port exposure to public internet
- Static or self-contained apps only (no shared databases with VPS stack)
- Separate Docker network from VPS AI services
- Minimal resource allocations per container

## Cloudflare DNS Configuration

In Cloudflare DNS dashboard, add CNAME records pointing to your tunnel:

| Name | Type | Target |
|------|------|--------|
| cd | CNAME | `<tunnel-id>.cfargotunnel.com` |
| cigarspot | CNAME | `<tunnel-id>.cfargotunnel.com` |
| kiki | CNAME | `<tunnel-id>.cfargotunnel.com` |

The VPS cloudflared tunnel routes all `*.ilgailu.com` traffic to Caddy:80, which then routes based on Host header.

## File Structure

```
public-gateway/
├── docker-compose.yml    # Service definitions
├── nginx.conf            # Nginx reverse proxy config (port 8585)
├── README.md             # This file
└── .gitignore            # Git ignore patterns
```

App source code is in sibling directories (`../cd_player`, `../cigarspot`, `../be-mine`).
