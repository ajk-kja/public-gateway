# Public Gateway

A Docker Compose stack for hosting public-facing static sites on a VPS while keeping private authenticated applications isolated.

## Why Use This?

If you run both **public static sites** and **private authenticated applications** (behind Cloudflare Zero Trust, Tailscale, etc.) on the same VPS, you may want to:

- Keep public and private services on separate Docker networks
- Route public traffic through a dedicated nginx proxy with minimal attack surface
- Avoid exposing ports directly to the internet
- Integrate with Cloudflare Tunnel for SSL and DDoS protection

This stack provides that separation layer.

## Architecture

```
Internet → Cloudflare Edge → cloudflared (VPS) → Main Reverse Proxy → public-nginx → App Containers
                                                        ↓
                                     (private apps handled directly by main proxy)
```

**How it works:**
1. A `cloudflared` container handles the Cloudflare Tunnel connection
2. Your main reverse proxy (e.g., Caddy) routes traffic based on Host header
3. Public domains are proxied to `public-nginx:8585`
4. `public-nginx` routes to isolated app containers

**Network Isolation:**
- App containers run on an isolated `public-gateway` network
- `public-nginx` bridges to your main proxy's network for routing
- Public apps cannot access private services directly

## Quick Start

### Prerequisites

- Docker and Docker Compose
- An existing reverse proxy (Caddy, Nginx, Traefik) with Cloudflare Tunnel
- A network your main proxy uses (e.g., `localai_default`)

### Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/ajk-kja/public-gateway.git
   cd public-gateway
   ```

2. Copy environment template:
   ```bash
   cp .env.example .env
   ```

3. Update `docker-compose.yml`:
   - Replace the external network name with yours
   - Modify service definitions for your apps

4. Update `nginx.conf`:
   - Add server blocks for each public domain
   - Point to your app containers

5. Add routes to your main reverse proxy (example for Caddy):
   ```caddy
   @myapp host myapp.example.com
   handle @myapp {
       reverse_proxy public-nginx:8585
   }
   ```

6. Start the stack:
   ```bash
   docker compose up -d
   ```

## Adding New Apps

1. Add service definition to `docker-compose.yml` (on `public-gateway` network)
2. Add server block to `nginx.conf` (listening on port 8585)
3. Add route to your main reverse proxy
4. Reload proxies:
   ```bash
   docker compose restart nginx
   # Reload your main proxy as well
   ```
5. Configure DNS (CNAME pointing to your Cloudflare Tunnel)

## Commands

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker logs public-nginx
docker logs <app-container-name>

# Rebuild after code changes
docker compose build --no-cache
docker compose up -d
```

## Security Considerations

- **Network isolation**: Apps cannot reach private services
- **No direct port exposure**: Traffic flows only through Cloudflare Tunnel
- **Minimal images**: Use `nginx:alpine` or similar lightweight bases
- **Resource limits**: Configure memory and CPU limits per container
- **Health checks**: All containers should have health checks configured

## File Structure

```
public-gateway/
├── docker-compose.yml    # Service definitions
├── nginx.conf            # Nginx reverse proxy config (port 8585)
├── .env.example          # Environment template
├── .gitignore            # Git ignore patterns
└── README.md             # This file
```

## Example Service Definition

```yaml
services:
  my-app:
    build:
      context: ../my-app
      dockerfile: Dockerfile
    restart: unless-stopped
    expose:
      - 80/tcp
    networks:
      - public-gateway
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.25'
```

## Example Nginx Server Block

```nginx
server {
    listen 8585;
    server_name myapp.example.com;

    location / {
        proxy_pass http://my-app:80;
    }

    location /health {
        access_log off;
        proxy_pass http://my-app:80/health;
    }
}
```

## License

MIT
