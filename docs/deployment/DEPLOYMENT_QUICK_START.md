# OpenClaw Deployment Quick Start Guide
## Choose Your Path & Deploy in Minutes

---

## 🚀 Quick Path Decision Tree

```
Do you want to run OpenClaw?
│
├─ ON YOUR LAPTOP (Dev, always-on personal)
│  └─ → Go to: LOCAL INSTALLATION
│
├─ ON A CHEAP VPS ($5-10/mo, reliable)
│  └─ → Go to: VPS WITH DOCKER
│
├─ NO-OPS CLOUD ($15-50/mo, one-click)
│  └─ → Go to: RAILWAY DEPLOY
│
└─ ENTERPRISE AWS (Scale, HA, compliance)
   └─ → Go to: AWS ECS/EC2
```

---

## Option 1: Local Installation (Dev)

### Time: ~10 minutes

```bash
# 1. Install (macOS / Linux / WSL2)
npm install -g openclaw@latest

# 2. Start onboarding wizard
openclaw onboard --install-daemon

# 3. Complete in wizard:
#    - Choose LLM provider (Anthropic recommended)
#    - Paste API key
#    - Optional: Add Telegram/Discord/Slack tokens
#    - Approve device pairing

# 4. Access Control UI
openclaw dashboard
# Opens: http://127.0.0.1:18789/

# 5. Verify running
systemctl --user status openclaw-gateway
# or (macOS):
launchctl list | grep openclaw
```

**Pros:** Zero cost, local control, fast  
**Cons:** Requires laptop always-on

---

## Option 2: VPS with Docker (Production)

### Time: ~20 minutes

### Prerequisites
- Hetzner/DigitalOcean/AWS account
- SSH access to a Linux VPS (Ubuntu 22.04 LTS or Debian 12)
- Anthropic API key (or another LLM provider)

### Step-by-Step

#### 1. Provision VPS
```bash
# Create VPS with ~2 vCPU, 4GB RAM
# Hetzner: CPX11 (~$5-10/mo)
# DigitalOcean: Basic Droplet (~$6/mo)
# AWS: t3.micro or t4g.micro (free tier)
```

#### 2. Connect & Install Docker
```bash
ssh root@YOUR_VPS_IP

# Install Docker
apt-get update
apt-get install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sh

# Verify
docker --version
docker compose version
```

#### 3. Clone OpenClaw
```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Create persistent directories
mkdir -p ~/.openclaw/workspace
```

#### 4. Configure Environment
```bash
# Create .env file with secrets
cat > .env <<'EOF'
OPENCLAW_IMAGE=openclaw:latest
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
OPENCLAW_GATEWAY_BIND=lan
ANTHROPIC_API_KEY=sk-ant-YOUR_KEY_HERE
SETUP_PASSWORD=$(openssl rand -hex 16)
EOF

# Generate secure tokens
openssl rand -hex 32  # Copy this for OPENCLAW_GATEWAY_TOKEN
openssl rand -hex 16  # Copy this for SETUP_PASSWORD

# Edit .env and paste the values
nano .env
```

#### 5. Customize Dockerfile (Important!)
```bash
# Edit Dockerfile to add required binaries
# (They won't survive container restarts if installed at runtime)

cat >> Dockerfile <<'EOF'

# Example: Add gog (Gmail CLI) if your skills use it
RUN curl -L https://github.com/steipete/gog/releases/latest/download/gog_Linux_x86_64.tar.gz \
    | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/gog
EOF
```

#### 6. Build & Launch
```bash
# Build image with all dependencies baked in
docker compose build

# Start the gateway
docker compose up -d openclaw-gateway

# Verify running
docker compose logs -f openclaw-gateway
# Wait for: "[gateway] listening on ws://0.0.0.0:18789"
```

#### 7. Access from Your Laptop
```bash
# On your laptop, create SSH tunnel
ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP

# In another terminal:
open http://127.0.0.1:18789/setup
# or:
xdg-open http://127.0.0.1:18789/setup

# Enter SETUP_PASSWORD (from .env)
# Complete the setup wizard in browser
```

#### 8. Verify & Persist
```bash
# Check gateway health
curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
  http://127.0.0.1:18789/health

# Verify volumes persisted
docker compose exec openclaw-gateway ls -la /home/node/.openclaw
```

### Daily Operations

```bash
# Start
docker compose up -d openclaw-gateway

# Restart
docker compose restart openclaw-gateway

# Stop
docker compose down

# Check logs
docker compose logs -f --tail=100 openclaw-gateway

# Backup (daily)
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
# Upload to S3 or external storage
```

---

## Option 3: Railway (No-Ops)

### Time: ~2 minutes

### Prerequisites
- GitHub account (recommended)
- Anthropic API key

### Step-by-Step

#### 1. One-Click Deploy
Click: [Deploy on Railway](https://railway.com/deploy/openclaw)

#### 2. Configure in Railway UI
```
- New Project
- Add service from template
- Service: OpenClaw
- Environment: production
```

#### 3. Set Environment Variables
```
SETUP_PASSWORD=<generate strong password>
PORT=8080
ANTHROPIC_API_KEY=sk-ant-YOUR_KEY
OPENCLAW_GATEWAY_TOKEN=<generate with: openssl rand -hex 32>
OPENCLAW_STATE_DIR=/data/.openclaw
OPENCLAW_WORKSPACE_DIR=/data/workspace
```

#### 4. Add Volume
```
Railway → Service → Storage
├── Add Volume
├── Mount Path: /data
└── Size: 10GB (default)
```

#### 5. Enable Public Networking
```
Railway → Service → Settings
├── Public Networking: ON
├── Port: 8080
└── Domain: auto-generated (e.g., openclaw-prod.railway.app)
```

#### 6. Access Setup Wizard
```
Open: https://YOUR_RAILWAY_DOMAIN/setup
├── Username: (ignored)
└── Password: SETUP_PASSWORD

Complete wizard:
├── Choose LLM provider
├── Paste API key
├── Optional: Add Telegram/Discord/Slack
└── Approve device pairing
```

#### 7. Access Control UI
```
https://YOUR_RAILWAY_DOMAIN/openclaw
```

### Ongoing

**Backup & Export:**
```
https://YOUR_RAILWAY_DOMAIN/setup/export
# Downloads entire config + workspace
# Use for migration to another platform
```

**Scale:**
```
Railway → Service → Settings → Instance Type
├── Railway.micro ($5/mo)
├── Railway.small ($10/mo)
└── Railway.large ($25+/mo)
```

---

## Option 4: AWS (Enterprise)

### Architecture Decisions

Choose based on needs:

#### A. EC2 + Docker (Cost-Effective, ~$10-20/mo)
Same as VPS option above, but on AWS.
- Launch t3.micro or t4g.micro instance (free tier eligible)
- Follow "VPS with Docker" steps above
- Use Elastic IP for static address
- Use EBS snapshots for backups

#### B. ECS Fargate (Highly Available, ~$50-150/mo)
More complex but provides:
- Multi-AZ redundancy
- Auto-scaling
- Managed load balancer
- CloudWatch integration

**Steps (simplified):**
```bash
# 1. Push image to Amazon ECR
aws ecr create-repository --repository-name openclaw
docker tag openclaw:latest 123456.dkr.ecr.us-east-1.amazonaws.com/openclaw:latest
docker push 123456.dkr.ecr.us-east-1.amazonaws.com/openclaw:latest

# 2. Create ECS Cluster
aws ecs create-cluster --cluster-name openclaw

# 3. Create Task Definition (see full guide for details)
# References ECR image, mounts EFS, sets env vars

# 4. Create ECS Service
# Launches 2 tasks across availability zones

# 5. Create Application Load Balancer
# HTTPS listener + target group pointing to tasks

# 6. Store secrets in AWS Secrets Manager
aws secretsmanager create-secret --name openclaw/anthropic-api-key \
  --secret-string 'sk-ant-...'
```

**Pros:**
- High availability (multi-AZ)
- Auto-scaling
- IAM integration
- Audit logs (CloudTrail)

**Cons:**
- Steep learning curve
- Expensive (~$50-150/mo)
- Requires AWS expertise

**Recommendation:** Start with EC2 + Docker, migrate to ECS if you outgrow it.

---

## Configuration Checklist

After deploying, configure OpenClaw:

### Basic Config
```bash
# File: ~/.openclaw/openclaw.json
{
  "identity": {
    "name": "My Assistant"
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-20250514",
      "workspace": "~/.openclaw/workspace"
    }
  }
}
```

### Add Channels (Optional)
```bash
# Telegram
openclaw channels add --channel telegram --token "123:ABCD..."

# Discord
openclaw channels add --channel discord --token "MTA..."

# WhatsApp (interactive QR login)
openclaw channels login whatsapp
```

### Verify Health
```bash
openclaw doctor       # Check for config issues
openclaw status       # Gateway status
openclaw models list  # Available models
```

---

## Troubleshooting

### Gateway won't start
```bash
openclaw doctor
# Shows exact issues + suggestions

# Restart
systemctl restart openclaw-gateway  # Linux
# or:
docker compose restart openclaw-gateway
```

### Can't connect to Control UI
```bash
# Check gateway token
echo $OPENCLAW_GATEWAY_TOKEN

# Verify binding
netstat -tlnp | grep 18789

# Check firewall
sudo ufw status  # Linux
# or AWS Security Group rules
```

### Messages not sending
```bash
# Check channel connectivity
openclaw channels list

# Verify API keys
openclaw models status

# View channel logs
grep "discord\|telegram\|whatsapp" ~/.openclaw/*.log
```

### High memory usage
```bash
# Restart gateway
systemctl restart openclaw-gateway

# Monitor
watch -n 5 'systemctl status openclaw-gateway'

# Or in Docker
docker stats openclaw-gateway
```

---

## Backup & Migration

### Backup
```bash
# Archive entire state
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/

# Upload to S3, B2, or external storage
# Keep at least 30 days of backups
```

### Restore
```bash
# Stop gateway
systemctl stop openclaw-gateway

# Extract backup (WARNING: overwrites config!)
tar -xzf openclaw-backup-YYYYMMDD.tar.gz -C /

# Start
systemctl start openclaw-gateway

# Verify
openclaw status
```

### Migrate Between Platforms
```bash
# Export from old platform
# (Railway: https://domain/setup/export)

# Deploy on new platform
# (VPS, AWS, local, etc.)

# Copy backup files to ~/.openclaw/

# Restart gateway
```

---

## Cost Comparison

| Platform | Monthly Cost | Setup | Uptime | Notes |
|----------|-------------|-------|--------|-------|
| **Local** | $0 | 5 min | Device-dependent | Dev/testing |
| **Hetzner VPS** | $5-10 | 20 min | 99.5% | Best value |
| **DigitalOcean** | $6 | 20 min | 99.5% | Simple |
| **Railway** | $15-50 | 2 min | 99.9% | One-click, no ops |
| **AWS EC2** | $10-20 | 25 min | 99.5% | Free tier eligible |
| **AWS ECS** | $50-150 | 45 min | 99.99% | HA, enterprise |

---

## Next Steps

1. **Choose your deployment path** (local, VPS, Railway, or AWS)
2. **Follow the step-by-step guide** above
3. **Complete the setup wizard** in the browser
4. **Add your channels** (Telegram, Discord, WhatsApp)
5. **Send a test message** to verify everything works
6. **Set up backups** (daily recommended)
7. **Monitor health** regularly

---

## Support & Resources

**Official Docs:** https://docs.openclaw.ai  
**GitHub:** https://github.com/openclaw/openclaw  
**Discord Community:** https://discord.gg/clawd  
**Railway Deploy:** https://railway.com/deploy/openclaw  

---

**Ready? Pick your path and deploy!**
