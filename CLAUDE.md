# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
This is a Caddy reverse proxy infrastructure configuration that routes traffic to multiple applications. The setup uses Docker Compose with automated deployment via GitHub Actions.

## Architecture
- **Main Config**: `caddy/Caddyfile` - Base Caddy configuration with health check, logging, and security headers
- **App Routing**: `caddy/conf.d/apps.caddy` - Defines reverse proxy rules for applications:
  - app1: Routes `/` and `/api/*` to ports 3000/3001
  - app2: Routes `/app2/*` and `/app2/api/*` to ports 3000/3001 (with prefix stripping)
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
- Push changes to `infra/caddy/**` on master branch triggers automatic deployment
- Manual deployment: Use "Run workflow" in GitHub Actions
- Target VM: Uses `VM_HOST`, `VM_SSH_USER` variables and `VM_SSH_KEY` secret

## Network Configuration
- External Docker network: `web` (must exist on deployment target)
- Exposed ports: 80 (HTTP) and 443 (HTTPS)
- Health check available at `/health`

## Security Features
- Security headers configured (X-Frame-Options, X-Content-Type-Options, etc.)
- Server header removal
- Gzip and Zstd compression enabled