<div align="center">

# 🐳 Docker + VPS + CI/CD
### Production Deployment Guide — Real World Experience

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Docker Hub](https://img.shields.io/badge/Docker_Hub-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://hub.docker.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)](https://nestjs.com/)
[![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)](https://nextjs.org/)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)

<p>A battle-tested, A–Z deployment guide built from real production experience.<br/>
Covers every issue, every fix, and every decision made along the way.</p>

</div>

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [The 3-Phase Procedure](#-the-3-phase-procedure)
- [Phase 1 — Dockerize Locally](#-phase-1--dockerize-locally)
- [Phase 2 — VPS Deploy](#-phase-2--vps-deploy)
- [Phase 3 — GitHub Actions CI/CD](#-phase-3--github-actions-cicd)
- [Production docker-compose.prod.yml](#-production-docker-composeprodyml)
- [Real Issues & Fixes](#-real-issues--fixes)
- [Quick Reference](#-quick-reference)
- [DevOps Roadmap](#-devops-roadmap)

---

## 🗺 Project Overview

3 separate repositories, each independently dockerized and deployed:

| Repo | Tech | Port | Docker Hub Image |
|---|---|---|---|
| `project-backend` | NestJS | 5000 | `bdcalling10/your-backend` |
| `project-dashboard` | Next.js | 3001 | `bdcalling10/your-dashboard` |
| `project-website` | Next.js | 3000 | `bdcalling10/your-client` |

### How Everything Connects

```
Push to main branch
        │
        ▼
GitHub Actions
  ├── Builds Docker image 
  ├── Pushes to Docker Hub
  └── SSHs into VPS
              │
              ▼
            VPS
  ├── Pulls new image from Docker Hub
  ├── Restarts only that container
  └── Nginx reverse proxies to container port
```

### VPS File Structure

```
/root/
├── deploy-api.sh         ← deploy script for api
├── deploy-admin.sh       ← deploy script for admin
└── deploy-client.sh      ← deploy script for client

/var/www/
├── docker-compose.prod.yml   ← ONE file, manages all 3 containers
├── api/
│   ├── .env                  ← api environment variables
│   └── uploads/              ← multer file uploads (persistent)
├── admin/
│   └── .env                  ← admin environment variables
└── client/
    └── .env                  ← client environment variables
```

> ⚠️ **Key Rule:** No application code lives on the VPS. Only `.env` files, `uploads/`, and `docker-compose.prod.yml`. Everything else is in Docker images on Docker Hub.

---

## 🔄 The 3-Phase Procedure

```
PHASE 1                    PHASE 2                    PHASE 3
───────────────            ───────────────            ───────────────
Dockerize locally    →     VPS Deploy           →     GitHub Actions CI/CD

• Dockerfile               • Install Docker           • SSH key setup
• .dockerignore            • Create .env files        • GitHub secrets
• docker-compose.yml       • docker-compose.prod      • deploy scripts
• Test locally             • Pull & run images        • workflow files
• Build & push             • Nginx + SSL + Cron       • Auto deploy
  to Docker Hub
```

---

## 🐳 Phase 1 — Dockerize Locally

> All commands run on your **local machine**.

---

### NestJS API

#### `Dockerfile`

```dockerfile
# ---- Stage 1: Build ----
FROM node:22-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ---- Stage 2: Production ----
FROM node:22-alpine AS production
WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist

RUN mkdir -p uploads

EXPOSE 5000
CMD ["node", "dist/src/main.js"]
```

> 💡 Entry point is `dist/src/main.js` confirmed from `"start:prod": "node dist/src/main.js"` in package.json. Verify with: `docker exec -it api sh` then `ls dist/src/`

#### `.dockerignore`

```
node_modules
dist
uploads
.env
.env.*
.git
.gitignore
README.md
*.md
.eslintrc*
.prettierrc
eslint.config.mjs
nest-cli.json
test
coverage
```

#### `docker-compose.yml` (local dev only)

```yaml
services:
  api:
    build: .
    container_name: backend-api
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "5000:5000"
    volumes:
      - ./uploads:/app/uploads
```

#### Test Locally

```bash
docker compose up --build
docker ps
docker logs backend-api
# should see: Application is running on: http://localhost:5000
```

#### Build & Push to Docker Hub

```bash
docker login
docker build -t username/your_docker_image:latest .
docker push username/your_docker_image:latest
```

---

### Next.js Admin Dashboard

#### ⚠️ Update `next.config.ts` First

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',   // REQUIRED for Docker

..........
  },
};

export default nextConfig;
```

#### `Dockerfile`

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG NEXT_PUBLIC_BACKEND_API_URL
ENV NEXT_PUBLIC_BACKEND_API_URL=$NEXT_PUBLIC_BACKEND_API_URL

ARG NEXTAUTH_URL
ENV NEXTAUTH_URL=$NEXTAUTH_URL

ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:22-alpine AS production
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3001

RUN addgroup --system --gid 1001 nodejs
RUN adduser  --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3001
CMD ["node", "server.js"]
```

#### `.dockerignore`

```
node_modules
.next
.env
.env.*
.env.local
.git
.gitignore
README.md
*.md
.eslintrc*
```

#### `docker-compose.yml` (local dev only)

```yaml
services:
  admin:
    build:
      context: .
      args:
        NEXT_PUBLIC_BACKEND_API_URL: http://localhost:5000/api/v1
        NEXTAUTH_URL: http://localhost:3001
    container_name: admin-dashboard
    restart: unless-stopped
    env_file:
      - .env.local
    ports:
      - "3001:3001"
```

#### Build & Push to Docker Hub

```bash
docker build \
  --build-arg NEXT_PUBLIC_BACKEND_API_URL=https://api.yourdomain.com/api/v1 \
  --build-arg NEXTAUTH_URL=https://admin.yourdomain.com \
  -t username/your_docker_image:latest .

docker push username/your_docker_image:latest
```

---

### Next.js Client

#### ⚠️ Update `next.config.ts` First

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',   // REQUIRED for Docker

  ...........
};

export default nextConfig;
```

#### `Dockerfile`

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG NEXT_PUBLIC_API_BASE_URL
ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL

ARG NEXTAUTH_URL
ENV NEXTAUTH_URL=$NEXTAUTH_URL

ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:22-alpine AS production
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000

RUN addgroup --system --gid 1001 nodejs
RUN adduser  --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

#### `.dockerignore`

```
node_modules
.next
.env
.env.*
.env.local
.git
.gitignore
README.md
*.md
.eslintrc*
```

#### `docker-compose.yml` (local dev only)

```yaml
services:
  client:
    build:
      context: .
      args:
        NEXT_PUBLIC_API_BASE_URL: http://localhost:5000/api/v1
        NEXTAUTH_URL: http://localhost:3000
    container_name: client
    restart: unless-stopped
    env_file:
      - .env.local
    ports:
      - "3000:3000"
```

#### Build & Push to Docker Hub

```bash
docker build \
  --build-arg NEXT_PUBLIC_API_BASE_URL=https://api.yourdomain.com/api/v1 \
  --build-arg NEXTAUTH_URL=https://yourdomain.com \
  -t username/your_docker_image:latest .

docker push username/your_docker_image:latest
```

---

## 🖥️ Phase 2 — VPS Deploy

> All commands run on your **VPS** unless marked `[LOCAL]`.

### Step 1 — SSH + Update
```bash
ssh root@your_vps_ip
sudo apt update && sudo apt upgrade -y
sudo apt install curl git -y
```

### Step 2 — Install Docker
```bash
curl -fsSL https://get.docker.com | sh
sudo apt install docker-compose-plugin -y
docker -v && docker compose version
```

> Replaces Node.js + PM2. Docker brings its own Node. `restart: always` replaces `pm2 startup`.

### Step 3 — Create Directory Structure
```bash
mkdir -p /var/www/api/uploads
mkdir -p /var/www/admin
mkdir -p /var/www/client
```

### Step 4 — Create .env Files

```bash
nano /var/www/api/.env
```
```env
PORT=5000
NODE_ENV=production
MONGO_URI=your_mongo_connection_string
JWT_SECRET=your_jwt_secret
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
FRONTEND_URL=https://yourdomain.com
ADMIN_URL=https://admin.yourdomain.com
```

```bash
nano /var/www/admin/.env
```
```env
NEXT_PUBLIC_BACKEND_API_URL=https://api.yourdomain.com/api/v1
NEXTAUTH_SECRET=your_secret
NEXTAUTH_URL=https://admin.yourdomain.com
```

```bash
nano /var/www/client/.env
```
```env
NEXTAUTH_SECRET=your_secret
NEXTAUTH_URL=https://yourdomain.com
```

> 🔒 `.env` files never in git, never inside Docker images. Created manually once, stay permanently.
> `NEXT_PUBLIC_` vars are baked into the image — no need in VPS `.env`.

### Step 5 — Login to Docker Hub on VPS
```bash
docker login
```

### Step 6 — Pull & Start All Containers
```bash
cd /var/www
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
docker ps
docker logs api/admin/client
```

### Step 7 — Configure Nginx
```bash
sudo apt install nginx -y
```

**API:**
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;
    client_max_body_size 100M;
    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Admin:**
```nginx
server {
    listen 80;
    server_name admin.yourdomain.com;
    client_max_body_size 100M;
    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Client:**
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    client_max_body_size 100M;
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/api    /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/admin  /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/client /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

### Step 8 — SSL + Auto-Renew
```bash
sudo apt install certbot python3-certbot-nginx -y
certbot --nginx -d api.yourdomain.com
certbot --nginx -d admin.yourdomain.com
certbot --nginx -d yourdomain.com -d www.yourdomain.com

sudo apt install cron -y
sudo systemctl enable cron && sudo systemctl start cron
crontab -e
# Add: 0 3 * * * certbot renew --quiet
```

---

## ⚙️ Phase 3 — GitHub Actions CI/CD

### Step 1 — Generate SSH Key `[LOCAL]`
```bash
ssh-keygen -t ed25519 -C "youremail@example.com"
cat ~/.ssh/id_ed25519.pub   # public  → add to VPS
cat ~/.ssh/id_ed25519       # private → add to GitHub Secrets
```

### Step 2 — Add Public Key to VPS(root)
```bash
nano ~/.ssh/authorized_keys
# paste public key → save
```

### Step 3 — GitHub Secrets (all 3 repos)

| Secret | Value |
|---|---|
| `SSH_HOST` | Run `curl -4 ifconfig.me` on VPS — use IPv4! |
| `SSH_USER` | `root` |
| `SSH_KEY` | Full content of `id_ed25519` including BEGIN/END lines |
| `DOCKER_USERNAME` | Your Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub PAT (recommended over password) |

### Step 4 — Deploy Scripts on VPS

```bash
nano ~/deploy-api.sh
```
```bash
set -e
echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
docker pull bdcalling10/bernie-backend:latest
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps api
docker image prune -f
```

```bash
nano ~/deploy-admin.sh
```
```bash
set -e
echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
docker pull ahmmed009/admin-dashboard:latest
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps admin
docker image prune -f
```

```bash
nano ~/deploy-client.sh
```
```bash
set -e
echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
docker pull amitdev17/barnie-client:latest
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps client
docker image prune -f
```

```bash
chmod +x ~/deploy-api.sh ~/deploy-admin.sh ~/deploy-client.sh
```

### Step 5 — Workflow Files

**api-repo `.github/workflows/deploy.yml`**
```yaml
name: Deploy API

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: bdcalling10/bernie-backend:latest

      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          envs: DOCKER_USERNAME,DOCKER_PASSWORD
          script: bash ~/deploy-api.sh
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

**admin-repo `.github/workflows/deploy.yml`**
```yaml
name: Deploy Admin

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          build-args: |
            NEXT_PUBLIC_BACKEND_API_URL=https://api.yourdomain.com/api/v1
            NEXTAUTH_URL=https://admin.yourdomain.com
          tags: ahmmed009/admin-dashboard:latest

      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          envs: DOCKER_USERNAME,DOCKER_PASSWORD
          script: bash ~/deploy-admin.sh
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

**client-repo `.github/workflows/deploy.yml`**
```yaml
name: Deploy Client

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          build-args: |
            NEXT_PUBLIC_API_BASE_URL=https://api.yourdomain.com/api/v1
            NEXTAUTH_URL=https://yourdomain.com
          tags: amitdev17/barnie-client:latest

      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          envs: DOCKER_USERNAME,DOCKER_PASSWORD
          script: bash ~/deploy-client.sh
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

---

## 📦 Production docker-compose.prod.yml

Lives at `/var/www/docker-compose.prod.yml` — never in any repo:

```yaml
services:
  api:
    image: bdcalling10/bernie-backend:latest
    container_name: api
    restart: always
    env_file: /var/www/api/.env
    ports:
      - "5000:5000"
    volumes:
      - /var/www/api/uploads:/app/uploads
    networks:
      - app-network

  admin:
    image: ahmmed009/admin-dashboard:latest
    container_name: admin
    restart: always
    env_file: /var/www/admin/.env
    ports:
      - "3001:3001"
    networks:
      - app-network

  client:
    image: amitdev17/barnie-client:latest
    container_name: client
    restart: always
    env_file: /var/www/client/.env
    ports:
      - "3000:3000"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

---

## 🔥 Real Issues & Fixes

These are real issues encountered during this exact deployment:

### 1. `NEXT_PUBLIC_` variables showing as `undefined`
**Symptom:** API calls hitting `https://yourdomain.com/undefined/endpoint`

**Cause:** `NEXT_PUBLIC_` vars are baked at **build time**, not runtime. Passing via `env_file` on VPS does nothing.

**Fix:** Use `ARG` + `ENV` in Dockerfile builder stage:
```dockerfile
ARG NEXT_PUBLIC_API_BASE_URL
ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL
```
```bash
docker build --build-arg NEXT_PUBLIC_API_BASE_URL=https://api.yourdomain.com/api/v1 .
```

---

### 2. CORS error for admin but not client
**Cause:** Backend only allowed `FRONTEND_URL` (client domain). Admin domain was blocked.

**Fix — `app.config.ts`:**
```ts
adminUrl: process.env.ADMIN_URL || '*',
```

**Fix — `main.ts`:**
```ts
app.enableCors({
  origin: [
    configService.get<string>('app.frontendUrl'),
    configService.get<string>('app.adminUrl'),
  ],
  credentials: true,
});
```

**Fix — `/var/www/api/.env`:**
```env
FRONTEND_URL=https://yourdomain.com
ADMIN_URL=https://admin.yourdomain.com
```

---

### 3. Certbot failing — no valid A records
**Symptom:** `no valid A records found for yourdomain.com`

**Cause:** DNS A records for root domain not added — only subdomains configured.

**Fix:** Add in DNS panel:
| Type | Name | Value |
|---|---|---|
| A | `@` | VPS IP |
| A | `www` | VPS IP |
| A | `api` | VPS IP |
| A | `admin` | VPS IP |

Wait 5–30 min, then re-run certbot.

---

### 4. GitHub Actions SSH timeout
**Symptom:** `dial tcp ***:22: i/o timeout`

**Cause:** `SSH_HOST` had IPv6 address. GitHub Actions connects via IPv4 only.

**Fix:**
```bash
curl -4 ifconfig.me   # note the -4 flag
```
Update `SSH_HOST` secret with IPv4 address.

---

### 5. `NEXTAUTH_URL` wrong in production
**Symptom:** Login redirects going to `localhost`.

**Cause:** `NEXTAUTH_URL=http://localhost:3000` copied from local dev into VPS `.env`.

**Fix:** Bake `NEXTAUTH_URL` at build time like `NEXT_PUBLIC_` vars:
```dockerfile
ARG NEXTAUTH_URL
ENV NEXTAUTH_URL=$NEXTAUTH_URL
```
Pass via `--build-arg NEXTAUTH_URL=https://yourdomain.com` when building.

---

### 6. Docker daemon not running (Windows)
**Symptom:** `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`

**Fix:** Open Docker Desktop, wait for whale icon 🐳 to stop animating.

---

### 7. `version` attribute warning in compose
**Symptom:** `the attribute version is obsolete`

**Fix:** Remove `version: '3.9'` line. Modern Docker Compose doesn't need it.

---

## 📖 Quick Reference

### Changing ENV Variables

| Var type | Action needed |
|---|---|
| Regular vars (`MONGO_URI`, `JWT_SECRET`) | Edit `/var/www/api/.env` on VPS → restart container |
| `NEXT_PUBLIC_` vars | Rebuild image → push → pull on VPS |
| `NEXTAUTH_URL` | Rebuild image → push → pull on VPS |

### Restart a Single Container
```bash
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps api
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps admin
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps client
```

### Docker vs PM2 Commands

| PM2 | Docker |
|---|---|
| `pm2 list` | `docker ps` |
| `pm2 logs api` | `docker logs api` |
| `pm2 logs api -f` | `docker logs api -f` |
| `pm2 restart api` | `docker compose -f /var/www/docker-compose.prod.yml restart api` |
| `pm2 startup` | `restart: always` in compose (already set) |

### Check Logs
```bash
docker logs api --tail 50
docker logs api -f        # live logs
```

### Daily Development (no Docker needed)
```bash
npm run start:dev    # NestJS hot reload
npm run dev          # Next.js hot reload
```

> Use Docker only for testing production build or deploying — not for daily development.

---

## 🗺️ DevOps Roadmap

```
✅ Done:    Docker + Docker Compose locally
✅ Done:    VPS Deploy + Nginx + SSL + Cron
✅ Done:    GitHub Actions CI/CD + Docker Hub

→ Next:     Health checks in docker-compose.prod.yml
→ Next:     Build caching in GitHub Actions (faster CI)
→ Then:     Kubernetes — orchestrate containers at scale
→ Then:     Helm Charts — K8s package manager
→ Then:     ArgoCD / Jenkins — advanced pipelines with rollback
→ Later:    Prometheus + Grafana — monitoring and alerting
```

---

<div align="center">

### Built from real production experience — every issue documented, every fix included.

*From local Docker setup to live production with automated CI/CD.*

</div>
