# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview.
This is a Caddy reverse proxy infrastructure configuration for Greenacape.ge that routes traffic to applications. The setup uses Docker Compose with automated deployment via GitHub Actions to VM 35.188.90.180.

## Architecture
- **Main Config**: `caddy/Caddyfile` - Base Caddy configuration with health check, logging, and security headers
- **App Routing**: Main domain `greenacape.ge` and fallback `:80` for HTTP traffic:
  - Main app: Routes to your-app-name:80 for greenacape.ge
  - API: Routes `/api/*` to your-backend-app:3000
  - Frontend: Routes other traffic to your-frontend-app:3000
- **Docker Setup**: `caddy/compose.yml` - Caddy service configuration with volume mounts
- **Deployment**: `.github/workflows/deploy-caddy.yml` - Automated deployment to VM

## Common Commands

### Local Development
```bash
cd caddy
docker compose up -d              # Start Caddy locally
docker compose logs -f            # View logs
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile  # Reload config
curl http://localhost/health      # Test health endpoint
```

### Configuration Testing
```bash
# Validate Caddyfile syntax
docker run --rm -v ./caddy/Caddyfile:/etc/caddy/Caddyfile caddy:2 caddy validate --config /etc/caddy/Caddyfile
```

### Deployment
- Push changes to `caddy/**` on master branch triggers automatic deployment
- Manual deployment: Use "Run workflow" in GitHub Actions
- Target VM: 35.188.90.180 (greenscapegeo@instance-20250918-194008)
- Uses `VM_HOST`, `VM_SSH_USER` variables and `VM_SSH_KEY` secret

## Network Configuration
- External Docker network: `web` (must exist on deployment target)
- Exposed ports: 80 (HTTP) and 443 (HTTPS)
- Health check available at `/health`

## Security Features
- Security headers configured (X-Frame-Options, X-Content-Type-Options, etc.)
- Server header removal
- Gzip and Zstd compression enabled