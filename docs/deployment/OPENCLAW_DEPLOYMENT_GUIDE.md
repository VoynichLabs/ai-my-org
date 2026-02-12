# OpenClaw Deployment on Railway & AWS
**Author:** openai-codex/gpt-5.1-codex-mini  
**Model:** openai-codex/gpt-5.1-codex-mini  
**Generated on:** 11 February 2026  

## Comprehensive Research & Architecture Guide

**Date:** February 11, 2026  
**Status:** Production-Ready Reference  
**Focus Areas:** Remote Deployment, Docker, systemd, Railway, AWS, Networking, Secrets, Monitoring

---

## Executive Summary

OpenClaw is a self-hosted AI Gateway for multi-channel messaging (WhatsApp, Telegram, Discord, etc.). This guide documents:

- **Deployment Architecture:** How OpenClaw handles remote deployments
- **Container Strategy:** Docker containerization for isolated, scalable runtimes
- **Service Management:** systemd/launchd daemon configuration for 24/7 operation
- **Platform Patterns:** Railway vs. VPS vs. AWS deployment tradeoffs
- **Security:** Networking, authentication, secrets management
- **Operations:** Monitoring, health checks, logging, and troubleshooting

---

## Part 1: OpenClaw Deployment Architecture

### Core Concepts

**OpenClaw is a single-process Gateway:**
- Always-on routing, control plane, channel connections
- Multiplexed single port for WebSocket (RPC), HTTP APIs, Control UI, webhooks
- Configuration hot-reload (hybrid mode: auto-restarts when necessary)
- Multi-agent capable with per-agent sandboxing

**What Persists on the Host:**
```
~/.openclaw/
├── openclaw.json              # Main config (hot-reloadable)
├── auth-profiles.json         # Model provider credentials (OAuth/API keys)
├── .env                        # Environment variables (fallback)
├── agents/
│   ├── <agent-id>/
│   │   ├── sessions/          # Chat history & state
│   │   └── auth-profiles.json # Per-agent credential overrides
├── workspace/                  # Agent workspace (code, artifacts)
├── channels/                   # Channel state (WhatsApp QR, etc.)
└── [other state]
```

**Gateway Runtime Model:**
```
Node.js Process (OpenClaw)
├── Port 18789 (default)
│   ├── WebSocket: RPC & Control messages
│   ├── HTTP: OpenAI-compatible API, responses, tool invoke
│   ├── Control UI: Browser dashboard
│   └── Webhooks: Cron, automations, channel callbacks
├── Configuration hot-reload (watches ~/.openclaw/openclaw.json)
├── Agent spawning (isolated per agent or shared)
└── Channel multiplexing (WhatsApp, Telegram, Discord, etc.)
```

### Deployment Topology Patterns

#### Pattern 1: Local/Single-Machine (Dev)
```
User's Laptop
├── OpenClaw Gateway (systemd user service or manual)
├── ~/.openclaw/ (local state)
└── Browser → http://127.0.0.1:18789
```
**Use case:** Development, testing, always-on personal assistant  
**Pros:** No cloud cost, full control, fast  
**Cons:** Requires always-on laptop

#### Pattern 2: VPS/Cloud VM (Production)
```
Hetzner/AWS/DigitalOcean VM
├── Docker (isolated OpenClaw)
├── Persistent volumes: /root/.openclaw, /root/.openclaw/workspace
├── systemd (system unit) or docker-compose
└── SSH Tunnel from Laptop → VM:18789
```
**Use case:** Reliable always-on, shared team access, moderate scale  
**Pros:** Cheap (~$5-20/mo), simple, full control  
**Cons:** Requires Linux ops knowledge, manual backup

#### Pattern 3: Platform-as-a-Service (Railway)
```
Railway Infrastructure
├── Docker container (managed by Railway)
├── Persistent volume (/data)
├── Web setup wizard (/setup endpoint)
├── HTTPS domain (auto + custom)
├── Environment variables (encrypted secrets)
└── Auto-restart, auto-scaling (optional)
```
**Use case:** No-ops, GitHub/GitLab connected, turnkey setup  
**Pros:** One-click deploy, automatic backups, built-in domain  
**Cons:** Monthly cost (~$5-50), less control

#### Pattern 4: AWS (ECS/Lambda + RDS)
```
AWS Account
├── ECS Fargate (container orchestration)
├── EFS (persistent storage for ~/.openclaw)
├── RDS (optional: session/state database)
├── ALB/API Gateway (HTTPS ingress)
├── Secrets Manager (for credentials)
└── CloudWatch (logging & monitoring)
```
**Use case:** Enterprise, high availability, compliance  
**Pros:** Scalable, redundant, audit logs, IAM integration  
**Cons:** Complex, expensive ($50-500+/mo), steep learning curve

---

## Part 2: Container Deployment (Docker)

### Docker Strategy

OpenClaw is **container-agnostic** but Docker is the recommended approach for:
- Isolated environments (VPS, cloud)
- Reproducible builds
- Easy upgrades without reinstalling dependencies
- Per-session agent sandboxing (optional)

### Quick Start: Docker Compose

**File Structure:**
```
openclaw/
├── Dockerfile              # Build config (from repo)
├── docker-compose.yml      # Service orchestration
├── .env                    # Secrets & env vars
├── Dockerfile.sandbox      # Optional: agent sandboxing
└── scripts/docker-setup.sh # One-click bootstrap
```

**Minimal docker-compose.yml:**
```yaml
services:
  openclaw-gateway:
    image: openclaw:latest
    build: .
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_BIND=lan
      - OPENCLAW_GATEWAY_PORT=18789
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      # Config & state persist on host (survives container recreation)
      - ~/.openclaw:/home/node/.openclaw
      - ~/.openclaw/workspace:/home/node/.openclaw/workspace
    ports:
      # Recommended: loopback-only; access via SSH tunnel
      - "127.0.0.1:18789:18789"
    healthcheck:
      test: ["CMD", "node", "dist/index.js", "health", "--token", "${OPENCLAW_GATEWAY_TOKEN}"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Container User & Permissions:**
- Runs as non-root `node` user (UID 1000)
- Host files must be owned by UID 1000:
  ```bash
  sudo chown -R 1000:1000 ~/.openclaw
  ```

### Custom Dockerfile for Production

**Goal:** Bake external binaries into the image (they don't survive container restarts if installed at runtime)

```dockerfile
FROM node:22-bookworm

# System dependencies
RUN apt-get update && apt-get install -y \
    git curl wget ca-certificates \
    ffmpeg build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install external binaries that are required by skills
# Example 1: Gmail CLI (gog)
RUN curl -L https://github.com/steipete/gog/releases/latest/download/gog_Linux_x86_64.tar.gz \
    | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/gog

# Example 2: WhatsApp CLI (wacli)
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli_Linux_x86_64.tar.gz \
    | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/wacli

# Example 3: Google Places CLI (goplaces)
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_Linux_x86_64.tar.gz \
    | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/goplaces

# Install OpenClaw dependencies
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

# Build OpenClaw
COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

# Non-root user
RUN useradd -m -u 1000 node
USER node

ENV NODE_ENV=production
EXPOSE 18789
CMD ["node", "dist/index.js"]
```

### Agent Sandboxing (Docker in Docker)

Optional: Run agent tools in isolated Docker containers (separate from gateway container).

**Use Case:** Untrusted agents, multi-tenant, strict isolation

**Config:**
```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "scope": "agent",
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "network": "none",
          "user": "1000:1000",
          "capDrop": ["ALL"],
          "memory": "1g",
          "cpus": 1
        },
        "prune": {
          "idleHours": 24,
          "maxAgeDays": 7
        }
      }
    }
  }
}
```

**Build sandbox image:**
```bash
scripts/sandbox-setup.sh
# Builds: openclaw-sandbox:bookworm-slim
```

---

## Part 3: Service Management (systemd & launchd)

### systemd User Service (Linux)

**Recommended for:** Always-on VPS, CI/CD servers, Linux desktops

**Auto-install:**
```bash
openclaw gateway install
# Creates ~/.config/systemd/user/openclaw-gateway.service
```

**Manual setup:**
```bash
mkdir -p ~/.config/systemd/user/
cat > ~/.config/systemd/user/openclaw-gateway.service <<'EOF'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
Environment="OPENCLAW_GATEWAY_TOKEN=YOUR_TOKEN"
Environment="ANTHROPIC_API_KEY=YOUR_KEY"

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway
```

**Enable lingering (survives logout):**
```bash
sudo loginctl enable-linger $USER
```

**Monitor:**
```bash
systemctl --user status openclaw-gateway
journalctl --user-unit=openclaw-gateway -f
```

### systemd System Service (Root)

**For multi-user or always-on machines:**

```bash
sudo cat > /etc/systemd/system/openclaw-gateway.service <<'EOF'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=openclaw
Group=openclaw
WorkingDirectory=/home/openclaw
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
EnvironmentFile=/home/openclaw/.openclaw/.env

[Install]
WantedBy=multi-user.target
EOF

sudo useradd -m -s /bin/false openclaw
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway
```

### launchd (macOS)

**Recommended for:** macOS development, always-on Mac mini servers

**Auto-install:**
```bash
openclaw gateway install
# Creates ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

**Manual setup:**
```bash
cat > ~/Library/LaunchAgents/ai.openclaw.gateway.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ai.openclaw.gateway</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/openclaw</string>
    <string>gateway</string>
    <string>--port</string>
    <string>18789</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>OPENCLAW_GATEWAY_TOKEN</key>
    <string>YOUR_TOKEN</string>
    <key>ANTHROPIC_API_KEY</key>
    <string>YOUR_KEY</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/tmp/openclaw.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/openclaw.err</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

### Docker as Service Manager

**When running in Docker Compose:**

```yaml
services:
  openclaw-gateway:
    restart: unless-stopped  # Restart on failure, unless manually stopped
```

**Systemd manages Docker:**
```bash
sudo systemctl enable --now docker
docker compose -f /path/to/docker-compose.yml up -d
```

---

## Part 4: Environment Variables & Configuration

### Environment Variable Precedence

OpenClaw reads env vars in this order (first wins):

1. **Process environment** (exported in shell, systemd, Docker, etc.)
2. **`.env` in cwd** (if running from a specific directory)
3. **`~/.openclaw/.env`** (global fallback; does not override)
4. **Config `env` block** in `openclaw.json` (if missing from above)
5. **Shell import** (optional: `env.shellEnv.enabled: true`)

**Recommended pattern for production:**

```bash
# File: ~/.openclaw/.env
ANTHROPIC_API_KEY=sk-ant-...
OPENCLAW_GATEWAY_TOKEN=<strong-random-token>
TELEGRAM_BOT_TOKEN=123456:ABC...
DISCORD_BOT_TOKEN=MTA...
GMAIL_OAUTH_CREDENTIALS=...
```

### Key Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `OPENCLAW_HOME` | Override home directory | `/var/lib/openclaw` |
| `OPENCLAW_STATE_DIR` | Config/state directory | `/var/lib/openclaw/state` |
| `OPENCLAW_CONFIG_PATH` | Config file path | `/etc/openclaw/openclaw.json` |
| `OPENCLAW_WORKSPACE_DIR` | Agent workspace root | `/var/lib/openclaw/workspace` |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token | `generated-by-openclaw` |
| `OPENCLAW_GATEWAY_BIND` | Bind mode | `loopback`, `lan`, `*` |
| `OPENCLAW_GATEWAY_PORT` | Listen port | `18789` |
| `OPENCLAW_LOAD_SHELL_ENV` | Import missing vars from shell | `1` |
| `ANTHROPIC_API_KEY` | Anthropic credentials | `sk-ant-...` |
| `OPENAI_API_KEY` | OpenAI credentials | `sk-...` |
| `NODE_ENV` | Node environment | `production` |

### Configuration File (openclaw.json)

**Structure:**
```json
{
  "identity": {
    "name": "MyAssistant",
    "avatar": "https://..."
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-20250514",
      "workspace": "~/.openclaw/workspace",
      "sandbox": {
        "mode": "off"
      }
    }
  },
  "channels": {
    "whatsapp": {
      "allowFrom": ["+1234567890"]
    },
    "telegram": {
      "token": "${TELEGRAM_BOT_TOKEN}"
    },
    "discord": {
      "token": "${DISCORD_BOT_TOKEN}"
    }
  },
  "gateway": {
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "token": "${OPENCLAW_GATEWAY_TOKEN}"
    },
    "reload": {
      "mode": "hybrid",
      "debounceMs": 300
    }
  },
  "env": {
    "shellEnv": {
      "enabled": false
    }
  }
}
```

### Env Var Substitution in Config

Use `${VAR_NAME}` syntax to reference env vars in config:

```json
{
  "channels": {
    "discord": {
      "token": "${DISCORD_BOT_TOKEN}"
    }
  },
  "models": {
    "providers": {
      "custom": {
        "apiKey": "${CUSTOM_API_KEY}"
      }
    }
  }
}
```

**Rules:**
- Only uppercase names match: `[A-Z_][A-Z0-9_]*`
- Missing vars throw an error at load time
- Escape with `$${VAR}` for literal `${VAR}` output

---

## Part 5: Railway Deployment

### Why Railway?

**Pros:**
- One-click deploy (GitHub/GitLab connected)
- Built-in persistent volumes (/data)
- Auto-restart & backups
- HTTPS domain included
- Web setup wizard (no terminal needed)
- Multi-environment support (dev/staging/prod)

**Cons:**
- Monthly cost (~$5-50 depending on compute)
- Less control than VPS
- Data export required for migration

### Quick Deploy

**Option A: One-Click (Recommended)**

1. Click: [Deploy on Railway](https://railway.com/deploy/openclaw)
2. Create volume mounted at `/data`
3. Set env vars: `SETUP_PASSWORD`, `PORT=8080`
4. Enable HTTP Proxy on port 8080
5. Open `https://<domain>/setup` → Complete wizard

**Option B: From GitHub**

1. Fork https://github.com/openclaw/openclaw
2. Connect fork to Railway
3. Deploy
4. Complete setup wizard

### Required Railway Configuration

**Variables:**
```
SETUP_PASSWORD=<strong-password>
PORT=8080
OPENCLAW_STATE_DIR=/data/.openclaw
OPENCLAW_WORKSPACE_DIR=/data/workspace
OPENCLAW_GATEWAY_TOKEN=<strong-token>
ANTHROPIC_API_KEY=sk-ant-...
```

**Volume:**
```
Mount: /data
Persistence: 50GB (Railway default)
```

**Public Networking:**
```
Port: 8080
HTTPS: Auto (Railway provides *.railway.app domain)
Custom Domain: Optional
```

### Setup Flow on Railway

1. Visit `https://<domain>/setup` (password-protected)
2. Choose LLM provider (Anthropic recommended)
3. Paste API key
4. Optional: Add Telegram/Discord/Slack tokens
5. Approve device pairing (browser)
6. Access Control UI: `https://<domain>/openclaw`

### Backup & Migration

**Export backup:**
```
GET https://<domain>/setup/export
```

This downloads:
- Config (openclaw.json)
- Credentials (auth-profiles.json)
- Workspace files
- Channel state (WhatsApp QR, etc.)

**Migrate to another host:**
1. Export from Railway
2. Deploy to new platform (VPS, AWS, etc.)
3. Upload backup files to `~/.openclaw/`
4. Restart gateway

---

## Part 6: VPS Deployment (Hetzner/AWS/DigitalOcean)

### Architecture: Docker on Linux VM

```
VPS (Ubuntu/Debian)
├── Docker Engine
├── docker-compose
├── Persistent host directories:
│   ├── /root/.openclaw         (config, state)
│   └── /root/.openclaw/workspace (agent workspace)
└── External binaries (installed in Dockerfile)
```

### Step 1: Provision VPS

**Recommended specs:**
- **Hetzner:** CPX11 (~$5-10/mo), 2 vCPU, 4GB RAM, 40GB SSD
- **DigitalOcean:** Basic Droplet, 2GB RAM, 50GB SSD (~$6/mo)
- **AWS:** t3.micro or t4g.micro (free tier eligible)

**OS:** Ubuntu 22.04 LTS or Debian 12

### Step 2: Install Docker

```bash
sudo apt-get update
sudo apt-get install -y git curl ca-certificates

# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Verify
docker --version
docker compose version
```

### Step 3: Clone & Prepare

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Create persistent directories
mkdir -p ~/.openclaw/workspace
sudo chown -R 1000:1000 ~/.openclaw
```

### Step 4: Configure Environment

**File: `.env`**
```bash
OPENCLAW_IMAGE=openclaw:latest
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_PORT=18789
ANTHROPIC_API_KEY=sk-ant-...
SETUP_PASSWORD=$(openssl rand -hex 16)
```

### Step 5: Custom Dockerfile

**Why?** Runtime package installs don't persist across container restarts. Bake everything into the image.

```dockerfile
FROM node:22-bookworm

# System packages
RUN apt-get update && apt-get install -y \
    git curl wget ca-certificates \
    ffmpeg build-essential \
    && rm -rf /var/lib/apt/lists/*

# External binaries required by your skills
RUN curl -L https://github.com/steipete/gog/releases/latest/download/gog_Linux_x86_64.tar.gz \
    | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/gog

# ... (repeat for other binaries)

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

USER node
ENV NODE_ENV=production
EXPOSE 18789
CMD ["node", "dist/index.js"]
```

### Step 6: Docker Compose Configuration

**File: `docker-compose.yml`**
```yaml
version: '3.8'

services:
  openclaw-gateway:
    image: ${OPENCLAW_IMAGE}
    build: .
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - HOME=/home/node
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
      - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      # Persist host directories
      - /root/.openclaw:/home/node/.openclaw
      - /root/.openclaw/workspace:/home/node/.openclaw/workspace
    ports:
      # Loopback-only recommended (SSH tunnel for access)
      - "127.0.0.1:18789:18789"
      # Optional: expose if firewalled and auth-protected
      # - "0.0.0.0:18789:18789"
    healthcheck:
      test: ["CMD", "node", "dist/index.js", "health", "--token", "${OPENCLAW_GATEWAY_TOKEN}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

### Step 7: Build & Launch

```bash
docker compose build
docker compose up -d openclaw-gateway

# Verify
docker compose logs -f openclaw-gateway
docker compose exec openclaw-gateway node dist/index.js health --token $OPENCLAW_GATEWAY_TOKEN
```

### Step 8: Access from Laptop

**SSH Tunnel (Recommended):**
```bash
ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP

# In another terminal:
open http://127.0.0.1:18789/setup
# Paste SETUP_PASSWORD
```

### Persistence Checklist

| Component | Location | Mechanism | Survives? |
|-----------|----------|-----------|-----------|
| Config (openclaw.json) | /root/.openclaw/ | Host volume | ✅ |
| Credentials | /root/.openclaw/ | Host volume | ✅ |
| Agent workspace | /root/.openclaw/workspace/ | Host volume | ✅ |
| WhatsApp QR | /root/.openclaw/ | Host volume | ✅ |
| External binaries | /usr/local/bin/ | Dockerfile | ✅ |
| Node runtime | Container filesystem | Docker image | ✅ |

### Backup & Recovery

**Daily backup to S3 (example):**
```bash
#!/bin/bash
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
aws s3 cp openclaw-backup-*.tar.gz s3://my-bucket/openclaw/
```

**Restore:**
```bash
aws s3 cp s3://my-bucket/openclaw/openclaw-backup-YYYYMMDD.tar.gz .
tar -xzf openclaw-backup-YYYYMMDD.tar.gz -C /root/
docker compose restart openclaw-gateway
```

---

## Part 7: AWS Deployment Patterns

### Pattern 1: ECS Fargate (Recommended for Scale)

**Architecture:**
```
AWS Account
├── VPC
├── ECS Cluster
├── ECS Service (Fargate)
│   ├── Task Definition (Docker image)
│   ├── 2 tasks (multi-AZ)
│   ├── EFS mount at /data
│   └── CloudWatch logs
├── Application Load Balancer
│   ├── HTTPS listener (ACM cert)
│   └── Target group: port 18789
├── EFS (persistent storage)
├── Secrets Manager (credentials)
└── CloudWatch (monitoring)
```

**Cost:** ~$50-150/mo (depends on task size, data transfer)

**Setup Overview:**
1. Create ECS task definition (references Docker image from ECR)
2. Push OpenClaw Docker image to Amazon ECR
3. Create EFS volume, mount at `/data`
4. Create ECS service with 2 tasks across AZs
5. Create ALB with HTTPS
6. Store credentials in Secrets Manager
7. Configure CloudWatch alarms

**Pros:**
- Highly available (multi-AZ)
- Auto-scaling (if needed)
- Audit logs (CloudTrail)
- IAM integration
- VPC isolation

**Cons:**
- Complex networking
- Expensive compared to VPS
- Steep learning curve

### Pattern 2: EC2 + Docker Compose (Cost-Effective)

**Architecture:** Same as VPS approach, but using AWS EC2

**Steps:**
1. Launch t3.micro or t4g.micro instance (eligible for free tier)
2. Install Docker
3. Clone repo, configure, and deploy
4. Use Elastic IP for static address
5. Use Route53 for DNS
6. Attach EBS volume for persistent storage

**Cost:** ~$10-20/mo (or free tier eligible)

**Pros:**
- Familiar (same as Hetzner/DigitalOcean)
- Cheap
- Flexible
- Easy backups (EBS snapshots)

**Cons:**
- Single point of failure (no multi-AZ)
- Manual patching

### Pattern 3: Lambda (Not Recommended)

OpenClaw is not suitable for Lambda because:
- Stateful (needs persistent storage)
- Long-running (Lambda timeout is 15 min max)
- Requires always-on gateway
- WebSocket support is complex

**Better alternatives:** ECS Fargate, EC2, VPS

### AWS Infrastructure-as-Code (Terraform)

**Community-maintained templates:**
- [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

**Example Terraform structure:**
```hcl
# main.tf
resource "aws_ecs_cluster" "openclaw" {
  name = "openclaw-cluster"
}

resource "aws_ecs_task_definition" "openclaw" {
  family           = "openclaw"
  network_mode     = "awsvpc"
  cpu              = "256"
  memory           = "512"
  
  container_definitions = jsonencode([{
    name      = "openclaw-gateway"
    image     = "${aws_ecr_repository.openclaw.repository_url}:latest"
    portMappings = [{
      containerPort = 18789
      hostPort      = 18789
      protocol      = "tcp"
    }]
    mountPoints = [{
      sourceVolume  = "openclaw-data"
      containerPath = "/data"
      readOnly      = false
    }]
    environment = [
      { name = "NODE_ENV", value = "production" }
    ]
    secrets = [
      { name = "OPENCLAW_GATEWAY_TOKEN", valueFrom = aws_secretsmanager_secret.gateway_token.arn }
      { name = "ANTHROPIC_API_KEY", valueFrom = aws_secretsmanager_secret.anthropic_key.arn }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.openclaw.name
        awslogs-region        = data.aws_region.current.name
        awslogs-stream-prefix = "ecs"
      }
    }
  }])

  volume {
    name = "openclaw-data"
    efs_volume_configuration {
      file_system_id = aws_efs_file_system.openclaw.id
    }
  }
}
```

---

## Part 8: Networking & Security

### Port & Binding Strategy

**OpenClaw listens on a single port (default 18789):**
```
Port 18789
├── WebSocket (RPC) → /gateway
├── HTTP APIs → /api, /tool, /response
├── Control UI → /
├── Webhooks → /hook
└── Setup wizard → /setup
```

**Binding Modes:**

| Mode | Binding | Access | Use Case |
|------|---------|--------|----------|
| `loopback` | 127.0.0.1 | Local only | Dev, SSH tunnel |
| `lan` | 0.0.0.0 (Docker) | Local network | Container-only networks |
| `*` or `0.0.0.0` | All interfaces | Public (if firewall allows) | Public internet (risky!) |

**Recommended production pattern:**
```
VPS/Container: bind = loopback (127.0.0.1)
↓ (SSH Tunnel)
User's Laptop: http://127.0.0.1:18789
```

**SSH Tunnel Setup:**
```bash
# Terminal 1: SSH tunnel
ssh -N -L 18789:127.0.0.1:18789 user@vps-ip

# Terminal 2: Access control UI
open http://127.0.0.1:18789
```

### Authentication & Authorization

**Gateway Authentication:**

OpenClaw uses **token-based** authentication for gateway access:

```json
{
  "gateway": {
    "auth": {
      "token": "${OPENCLAW_GATEWAY_TOKEN}"
    }
  }
}
```

**Where the token is sent:**
1. Control UI: Pasted into settings
2. WebSocket clients: `?token=<token>` in URL or header
3. HTTP APIs: `Authorization: Bearer <token>` header

**Device Pairing (Browser Access):**
1. First browser: Requests pairing code
2. OpenClaw: Generates code + displays in terminal
3. Browser: Submits code
4. OpenClaw: Approves, issues long-lived session token

**Model Provider Authentication:**

Separate from gateway auth. Uses OAuth or API keys:
- **Anthropic:** API key (recommended) or setup-token
- **OpenAI:** API key
- **Google:** OAuth (for Google Gemini, Gmail, Google Places)
- **Custom:** OPENROUTER_API_KEY, etc.

### Firewall Recommendations

**For VPS exposed to internet (not recommended):**

```bash
# UFW on Linux
sudo ufw enable
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 443/tcp       # HTTPS only
sudo ufw allow 80/tcp        # HTTP (redirect to HTTPS)
sudo ufw deny 18789          # Block gateway port

# Use reverse proxy (nginx/caddy) for HTTPS + auth
```

**Better approach:** Use SSH tunnel or VPN

```bash
# AWS Security Group
- Ingress 22/tcp from YOUR_IP/32 (SSH only)
- Ingress 443/tcp from 0.0.0.0/0 (HTTPS reverse proxy)
- Egress to 0.0.0.0/0 (outbound for API calls)
- NO inbound on 18789
```

### HTTPS & Reverse Proxy

**If exposing to internet, use nginx/Caddy:**

**nginx.conf:**
```nginx
upstream openclaw {
  server 127.0.0.1:18789;
}

server {
  listen 443 ssl http2;
  server_name myclaw.example.com;

  ssl_certificate /etc/letsencrypt/live/myclaw.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/myclaw.example.com/privkey.pem;

  # Auth redirect
  auth_request /auth;
  location = /auth {
    proxy_pass http://127.0.0.1:18789/api/auth/check;
  }

  # Gateway
  location / {
    proxy_pass http://openclaw;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}

# Redirect HTTP to HTTPS
server {
  listen 80;
  server_name myclaw.example.com;
  return 301 https://$host$request_uri;
}
```

### VPC & Network Isolation (AWS)

**Recommended AWS networking:**
```
VPC (10.0.0.0/16)
├── Public Subnet (ALB)
│   ├── ALB (HTTPS)
│   └── NAT Gateway (for outbound)
├── Private Subnet (ECS Tasks)
│   ├── ECS Task (OpenClaw)
│   └── EFS
└── Secrets Manager (encrypted credentials)
```

---

## Part 9: Secrets Management

### Environment Variables (Secure)

**Local machine (dev):**
```bash
# File: ~/.openclaw/.env
ANTHROPIC_API_KEY=sk-ant-...
OPENCLAW_GATEWAY_TOKEN=...
```

**systemd user service:**
```bash
# File: ~/.openclaw/.env
# Same as above; systemd reads it
```

**systemd system service:**
```bash
# File: /etc/openclaw/.env
# Owned by root, readable by openclaw user
ANTHROPIC_API_KEY=sk-ant-...
```

**Docker:**
```bash
# File: .env (in compose directory)
ANTHROPIC_API_KEY=sk-ant-...
DISCORD_BOT_TOKEN=...

# Loaded by docker-compose from .env or env_file
```

**AWS Secrets Manager (Recommended for AWS):**

```bash
# Store secret
aws secretsmanager create-secret \
  --name openclaw/anthropic-key \
  --secret-string 'sk-ant-...'

# Reference in ECS task definition
"secrets": [{
  "name": "ANTHROPIC_API_KEY",
  "valueFrom": "arn:aws:secretsmanager:us-east-1:123456:secret:openclaw/anthropic-key"
}]
```

**Railway Secrets (Recommended for Railway):**

1. Go to Railway → Service → Variables
2. Add secrets as encrypted variables
3. Railway does NOT expose in logs or UI

**HashiCorp Vault (Enterprise):**

```bash
vault write secret/openclaw/anthropic \
  api_key="sk-ant-..."

# Reference in config
{
  "env": {
    "vars": {
      "ANTHROPIC_API_KEY": "sk-ant-..."
    }
  }
}
```

### Credential Rotation (Important!)

**Anthropic API Key Rotation:**

1. Generate new key in Anthropic Console
2. Update environment variable
3. Restart gateway
4. Verify working in Control UI
5. Delete old key from console

**Discord/Telegram Bot Token Rotation:**

1. Regenerate token in Discord/Telegram settings
2. Update config or env var
3. Restart gateway
4. Delete old token

**Gateway Token (OPENCLAW_GATEWAY_TOKEN):**

```bash
# Generate new token
openssl rand -hex 32

# Update in config or env
export OPENCLAW_GATEWAY_TOKEN=<new-token>

# Restart gateway
systemctl restart openclaw-gateway  # or docker compose restart
```

---

## Part 10: Monitoring & Operations

### Health Checks

**Built-in health endpoint:**
```bash
curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
  http://127.0.0.1:18789/health

# Response: { "status": "ok", ... }
```

**Docker health check:**
```yaml
healthcheck:
  test: ["CMD", "node", "dist/index.js", "health", "--token", "${OPENCLAW_GATEWAY_TOKEN}"]
  interval: 30s
  timeout: 10s
  retries: 3
```

**systemd ExecStartPost health check:**
```bash
[Unit]
ExecStartPost=/bin/bash -c 'sleep 5 && curl -f http://127.0.0.1:18789/health || systemctl stop openclaw-gateway'
```

### Logging & Monitoring

**Local logs:**
```bash
# Real-time log follow
openclaw logs --follow

# Filter by component
journalctl --user-unit=openclaw-gateway -f

# Export logs
docker compose logs --timestamps openclaw-gateway > openclaw-2026-02-11.log
```

**systemd Journal:**
```bash
# User service
journalctl --user-unit=openclaw-gateway --since "2 hours ago"

# System service
sudo journalctl -u openclaw-gateway --since "2 hours ago"
```

**Docker logs:**
```bash
docker compose logs -f --tail=100 openclaw-gateway
```

**Structured logging (JSON):**

If logs are sent to aggregators (CloudWatch, Datadog, ELK), they can be parsed as JSON:

```json
{"level":"info","msg":"Gateway started","port":18789,"timestamp":"2026-02-11T20:32:00Z"}
```

### Key Metrics to Monitor

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Gateway uptime | systemd/Docker | < 99.5% (>4h downtime/mo) |
| Memory usage | Docker stats / systemd | > 80% of allocated |
| CPU usage | Docker stats / systemd | > 70% sustained |
| Disk space (state dir) | du -sh ~/.openclaw | > 80% of volume |
| Auth failures | Logs | > 5 in 1 minute |
| Model API errors | Logs | > 10% request failure rate |
| Message latency | Response headers | > 5 seconds |
| WebSocket disconnects | Logs | Unexpected spikes |

### Alerting Strategies

**systemd + Email:**
```bash
[Unit]
OnFailure=send-alert@%n.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo "OpenClaw gateway failed" | mail -s "Alert" admin@example.com'
```

**CloudWatch Alarms (AWS):**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name openclaw-memory-high \
  --metric-name MemoryUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456:alerts
```

**Datadog / New Relic / Grafana:**

Point agent to container logs → parse JSON → visualize in dashboards

### Backup & Disaster Recovery

**What to backup:**
1. `~/.openclaw/` (config, credentials, state)
2. `~/.openclaw/workspace/` (agent code & artifacts)
3. Database (if using RDS for sessions)

**Backup frequency:** Daily minimum, hourly for critical setups

**Backup destination:** Off-site (S3, B2, rsync to external server)

**Example: Daily S3 Backup:**
```bash
#!/bin/bash
set -e

# Backup
tar -czf /tmp/openclaw-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  ~/.openclaw/

# Upload to S3
aws s3 cp /tmp/openclaw-backup-*.tar.gz \
  s3://my-backup-bucket/openclaw/ \
  --sse AES256

# Cleanup local
rm /tmp/openclaw-backup-*.tar.gz

# Retention: keep last 30 days
aws s3 rm s3://my-backup-bucket/openclaw/ \
  --recursive \
  --exclude "*" \
  --include "openclaw-backup-*" \
  --older-than 30
```

**Restore from backup:**
```bash
# Download
aws s3 cp s3://my-backup-bucket/openclaw/openclaw-backup-YYYYMMDD-HHMMSS.tar.gz .

# Stop gateway
systemctl stop openclaw-gateway  # or docker compose stop

# Extract (WARNING: overwrites config!)
tar -xzf openclaw-backup-YYYYMMDD-HHMMSS.tar.gz -C /

# Start
systemctl start openclaw-gateway  # or docker compose up -d
```

### Troubleshooting Runbook

**Gateway won't start:**
```bash
# Check config syntax
openclaw doctor

# View logs
openclaw logs --follow

# Check permissions
ls -la ~/.openclaw/
```

**High memory usage:**
```bash
# Check for memory leaks
docker stats openclaw-gateway  # or watch systemctl status

# Restart
docker compose restart openclaw-gateway

# Monitor
watch -n 5 'docker stats openclaw-gateway --no-stream'
```

**Messages not sending:**
```bash
# Check channel connectivity
openclaw channels list

# View channel logs
grep "whatsapp\|telegram\|discord" ~/.openclaw/.log

# Restart channel
openclaw channels restart whatsapp
```

**Auth failures in Control UI:**
```bash
# Check gateway token
echo $OPENCLAW_GATEWAY_TOKEN

# Regenerate and update config
openssl rand -hex 32 > /tmp/new-token
# Update ~/.openclaw/openclaw.json or .env
# Restart gateway
```

---

## Part 11: Comparison Matrix

| Aspect | Local | VPS (Hetzner) | Railway | AWS ECS |
|--------|-------|-------|---------|---------|
| **Cost/mo** | $0 | $5-10 | $15-50 | $50-150 |
| **Setup time** | 5 min | 20 min | 2 min | 45 min |
| **Uptime SLA** | N/A | 99.5% | 99.9% | 99.99% |
| **HA/Redundancy** | ❌ | ❌ (single VM) | ✅ (multi-region) | ✅ (multi-AZ) |
| **Scaling** | Manual | Manual | Auto | Auto |
| **Backups** | Manual | Manual | Auto | Manual (EBS snapshots) |
| **HTTPS** | ❌ | ✅ (nginx) | ✅ (built-in) | ✅ (ACM) |
| **Monitoring** | ❌ | Manual | ✅ (built-in) | ✅ (CloudWatch) |
| **Ops Overhead** | Low | Medium | Minimal | High |
| **Data Control** | Full | Full | Limited (Railway) | Full (but AWS) |
| **Compliance** | N/A | N/A | GDPR/SOC2 | HIPAA/PCI/SOC2 |

---

## Part 12: Quick Reference

### Essential Commands

**Local/systemd:**
```bash
openclaw gateway status
openclaw gateway restart
openclaw logs --follow
openclaw doctor
openclaw onboard
```

**Docker:**
```bash
docker compose build
docker compose up -d openclaw-gateway
docker compose logs -f openclaw-gateway
docker compose exec openclaw-gateway bash
docker compose restart openclaw-gateway
```

**Configuration:**
```bash
openclaw config get gateway.port
openclaw config set agents.defaults.model "anthropic/claude-opus-4-20250514"
openclaw config show  # Print full config
```

**Channels:**
```bash
openclaw channels list
openclaw channels login whatsapp
openclaw channels add --channel telegram --token "123:ABC"
```

**Models:**
```bash
openclaw models status
openclaw models set anthropic/claude-opus-4-20250514
```

### Debugging Checklist

- [ ] Is the gateway running? `systemctl status openclaw-gateway`
- [ ] Are logs clean? `journalctl -u openclaw-gateway | grep ERROR`
- [ ] Is config valid? `openclaw doctor`
- [ ] Is auth token set? `echo $OPENCLAW_GATEWAY_TOKEN`
- [ ] Can gateway bind to port? `netstat -tlnp | grep 18789`
- [ ] Is firewall blocking? `sudo ufw status` or AWS Security Groups
- [ ] Are credentials valid? `openclaw models status`
- [ ] Is workspace writable? `touch ~/.openclaw/workspace/.test`

---

## References & Resources

**Official Documentation:**
- https://docs.openclaw.ai
- https://github.com/openclaw/openclaw

**Deployment Guides:**
- Railway: https://docs.openclaw.ai/install/railway
- Docker: https://docs.openclaw.ai/install/docker
- Hetzner VPS: https://docs.openclaw.ai/install/hetzner

**Community:**
- Discord: https://discord.gg/clawd
- DeepWiki: https://deepwiki.com/openclaw/openclaw
- GitHub Issues: https://github.com/openclaw/openclaw/issues

**Infrastructure-as-Code:**
- openclaw-terraform-hetzner: https://github.com/andreesg/openclaw-terraform-hetzner
- openclaw-docker-config: https://github.com/andreesg/openclaw-docker-config

---

**Document Status:** Complete  
**Last Updated:** February 11, 2026  
**Target Audience:** DevOps, SRE, Full-Stack Developers
