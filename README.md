# Public Gateway

Isolated Docker Compose stack for public-facing web applications. This stack is completely separate from the private AI services in `~/vps`.

## Architecture

```
Internet → Cloudflare Edge → cloudflared → Caddy :80 → Containers
```

## Services

| Service | Domain | Description |
|---------|--------|-------------|
| cd-player | cd.ilgailu.com | Vinyl record visualizer |
| cigarspot | cigarspot.ilgailu.com | Cigar inventory app |
| be-mine | kiki.ilgailu.com | Valentine's meme site |

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
docker logs public-caddy
docker logs public-cloudflared
```

### Rebuild after code changes
```bash
docker compose build --no-cache
docker compose up -d
```

## Adding a New Public App

1. Add service definition to `docker-compose.yml`
2. Add route to `Caddyfile`
3. Configure Cloudflare Tunnel hostname in Cloudflare Zero Trust dashboard
4. Create DNS CNAME record pointing to tunnel

## Security Notes

- No Tailscale restrictions - these are public sites
- No database connections - static or self-contained apps only
- Separate Docker network from VPS AI stack
- Minimal resource allocations

## Cloudflare Tunnel Configuration

The tunnel token is shared with the VPS stack. In Cloudflare Zero Trust dashboard:

1. Go to Networks → Tunnels
2. Select your tunnel
3. Add public hostnames:
   - `cd.ilgailu.com` → `http://localhost:8880`
   - `cigarspot.ilgailu.com` → `http://localhost:8880`
   - `kiki.ilgailu.com` → `http://localhost:8880`

Note: Port 8880 is the public gateway Caddy instance.
