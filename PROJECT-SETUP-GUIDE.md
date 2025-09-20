# Project Setup Guide for Greenacape.ge Caddy Infrastructure

This guide explains how to build and deploy applications that work with the Caddy reverse proxy infrastructure for `greenacape.ge`.

## Infrastructure Overview

Your Caddy setup routes traffic as follows:
- **Domain**: `greenacape.ge` (with automatic HTTPS)
- **VM**: `35.188.90.180` (greenscapegeo@instance-20250918-194008)
- **Health Check**: Available at `/health`
- **Docker Network**: `web` (external network)

## Current Routing Configuration

```
greenacape.ge → your-app-name:80          # Main application
:80/api/*     → your-backend-app:3000     # API requests
:80/*         → your-frontend-app:3000    # Frontend requests
```

## Project Requirements

### 1. Docker Setup
Your application must be containerized with Docker and use Docker Compose.

### 2. Network Configuration
All containers must connect to the external `web` network to communicate with Caddy.

### 3. Container Naming
Update the container names in `caddy/Caddyfile` to match your actual services:
- Replace `your-app-name:80`
- Replace `your-backend-app:3000`
- Replace `your-frontend-app:3000`

## Example Project Structure

```
your-project/
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   └── src/...
├── backend/
│   ├── Dockerfile
│   └── src/...
└── README.md
```

## Sample Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: ./frontend
    container_name: greenacape-frontend
    restart: unless-stopped
    networks:
      - web
    environment:
      - API_URL=https://greenacape.ge/api
    ports:
      - "3000:3000"

  backend:
    build: ./backend
    container_name: greenacape-backend
    restart: unless-stopped
    networks:
      - web
    environment:
      - NODE_ENV=production
      - DATABASE_URL=your-db-connection
    ports:
      - "3001:3000"

  database:
    image: postgres:15
    container_name: greenacape-db
    restart: unless-stopped
    networks:
      - web
    environment:
      - POSTGRES_DB=greenacape
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secure_password
    volumes:
      - db_data:/var/lib/postgresql/data

networks:
  web:
    external: true

volumes:
  db_data:
```

## Updating Caddy Configuration

After creating your services, update the Caddy configuration:

### Edit `caddy/Caddyfile`

```caddyfile
greenacape.ge {
  log {
    output stdout
    format console
    level DEBUG
  }

  encode gzip zstd

  # Main application (update with your main app)
  reverse_proxy greenacape-frontend:3000

  header {
    -Server
    X-Content-Type-Options nosniff
    X-Frame-Options DENY
    Referrer-Policy "strict-origin-when-cross-origin"
  }
}

:80 {
  log {
    output stdout
    format console
    level DEBUG
  }

  encode gzip zstd

  @health path /health
  respond @health "OK" 200

  # API → Backend
  handle /api/* {
    reverse_proxy greenacape-backend:3000
  }

  # UI → Frontend
  handle {
    reverse_proxy greenacape-frontend:3000
  }

  header {
    -Server
    X-Content-Type-Options nosniff
    X-Frame-Options DENY
    Referrer-Policy "strict-origin-when-cross-origin"
  }
}
```

## Deployment Steps

### 1. Prepare Your VM

```bash
# SSH into VM
ssh greenscapegeo@35.188.90.180

# Create project directory
mkdir -p ~/projects/greenacape-app
cd ~/projects/greenacape-app

# Ensure web network exists
docker network create web 2>/dev/null || true
```

### 2. Deploy Your Application

```bash
# Copy your project files to VM (from local machine)
scp -r ./your-project/* greenscapegeo@35.188.90.180:~/projects/greenacape-app/

# SSH into VM and start services
ssh greenscapegeo@35.188.90.180
cd ~/projects/greenacape-app

# Build and start your application
docker compose up -d

# Verify containers are running
docker ps
```

### 3. Update and Deploy Caddy

```bash
# Update the Caddy configuration with your container names
# Then push changes to trigger automatic deployment

git add caddy/Caddyfile
git commit -m "Update Caddy config for greenacape application"
git push origin master
```

### 4. Verify Deployment

```bash
# Test health endpoint
curl http://35.188.90.180/health

# Test your application
curl http://35.188.90.180
curl https://greenacape.ge

# Check Caddy logs
docker compose -f ~/caddy/compose.yml logs caddy -f
```

## Frontend Configuration

### API Endpoint Configuration
Your frontend should use these API endpoints:

```javascript
// Production
const API_BASE_URL = 'https://greenacape.ge/api';

// Development (when testing locally)
const API_BASE_URL = 'http://35.188.90.180/api';
```

### Environment Variables
```bash
# .env.production
REACT_APP_API_URL=https://greenacape.ge/api
VITE_API_URL=https://greenacape.ge/api
NEXT_PUBLIC_API_URL=https://greenacape.ge/api
```

## Backend Configuration

### CORS Settings
Configure CORS to allow requests from your domain:

```javascript
// Express.js example
app.use(cors({
  origin: ['https://greenacape.ge', 'http://35.188.90.180'],
  credentials: true
}));
```

### API Routes
Structure your API routes to work with the `/api/*` prefix:

```javascript
// All routes should be under /api
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/data', dataRoutes);
```

## SSL Certificate

Caddy automatically obtains SSL certificates from Let's Encrypt for `greenacape.ge`. Ensure:

1. **DNS is configured**: `greenacape.ge` points to `35.188.90.180`
2. **Ports are open**: 80 and 443 are accessible
3. **Domain is valid**: Let's Encrypt can verify domain ownership

## Monitoring and Troubleshooting

### Check Application Logs
```bash
# Application logs
docker logs greenacape-frontend
docker logs greenacape-backend

# Caddy logs
docker compose -f ~/caddy/compose.yml logs caddy
```

### Common Issues

1. **502 Bad Gateway**: Container not running or wrong port
2. **504 Gateway Timeout**: Application taking too long to respond
3. **SSL Certificate Issues**: DNS not pointing to correct IP

### Health Checks
Add health check endpoints to your applications:

```javascript
// Backend health check
app.get('/api/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});
```

## Security Considerations

1. **Environment Variables**: Never commit secrets to git
2. **Database Security**: Use strong passwords and limit access
3. **Container Security**: Keep base images updated
4. **Network Security**: Only expose necessary ports

## Automatic Deployment

The Caddy configuration automatically deploys when you push changes to the master branch. For your application:

1. Set up GitHub Actions in your project repository
2. Use similar deployment workflow as the Caddy setup
3. Deploy to the same VM using the SSH keys already configured

## Example GitHub Actions Workflow

```yaml
name: Deploy Application

on:
  push:
    branches: [ master ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to VM
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_SSH_USER }}
          key: ${{ secrets.VM_SSH_KEY }}
          script: |
            cd ~/projects/greenacape-app
            git pull origin master
            docker compose up -d --build
```

## Support

For issues with the Caddy infrastructure, check:
- GitHub Actions: https://github.com/greenscapegeo/CADDY-to-VM/actions
- VM logs: `sudo journalctl -u docker`
- Caddy logs: `docker compose logs caddy`

---

**Next Steps:**
1. Build your application following this guide
2. Update the Caddy configuration with your container names
3. Deploy and test your application
4. Set up monitoring and automated deployments