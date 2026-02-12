# OpenClaw Deployment Research - Executive Summary
**Author:** openai-codex/gpt-5.1-codex-mini  
**Model:** openai-codex/gpt-5.1-codex-mini  
**Generated on:** 11 February 2026  

**Research Date:** February 11, 2026  
**Research Scope:** OpenClaw remote deployment, Railway, AWS, Docker, systemd, secrets, monitoring  
**Status:** ✅ Complete with 3 comprehensive guides

---

## 📋 What Was Researched

This subagent researched OpenClaw deployment across multiple platforms and created three comprehensive documentation files:

### 1. **OPENCLAW_DEPLOYMENT_GUIDE.md** (37 KB)
Complete technical reference covering:
- Deployment architecture & topology patterns
- Docker containerization strategy
- systemd/launchd daemon integration  
- Environment variables & configuration
- Railway deployment (one-click)
- VPS deployment (Hetzner/DigitalOcean/AWS EC2)
- AWS patterns (ECS Fargate, EC2, Lambda)
- Networking & security best practices
- Secrets management (env vars, AWS Secrets Manager, Vault)
- Monitoring, logging, health checks
- Backup & disaster recovery
- Troubleshooting runbook

### 2. **DEPLOYMENT_QUICK_START.md** (10 KB)
Practical step-by-step guide:
- Decision tree for choosing deployment platform
- Option 1: Local installation (dev)
- Option 2: VPS with Docker (production)
- Option 3: Railway (no-ops)
- Option 4: AWS (enterprise)
- Configuration checklist
- Troubleshooting quick fixes
- Cost comparison matrix

### 3. **ADVANCED_DEPLOYMENT_PATTERNS.md** (22 KB)
Production-ready architectures:
- Railway multi-stage deployments (dev/staging/prod)
- Railway + GitHub Actions CI/CD
- AWS ECS Fargate full Terraform template
- AWS disaster recovery (multi-region failover)
- Multi-agent deployment
- Tailscale VPN integration
- mTLS for APIs
- CloudWatch, Datadog, Prometheus monitoring
- Advanced troubleshooting
- Performance tuning

---

## 🎯 Key Findings

### OpenClaw Architecture
- **Single-process Gateway** listening on one port (default 18789)
- **Multiplexed** for WebSocket (RPC), HTTP APIs, Control UI, webhooks
- **Hot-reload** configuration (automatic restart when needed)
- **Multi-agent capable** with per-agent sandboxing optional
- **Persistent state** at `~/.openclaw/` (survives container/service restarts)

### Remote Deployment Patterns

| Pattern | Cost | Setup | Uptime | Best For |
|---------|------|-------|--------|----------|
| **Local** | $0 | 5 min | Device-dependent | Dev/testing |
| **VPS** | $5-10/mo | 20 min | 99.5% | Reliable always-on |
| **Railway** | $15-50/mo | 2 min | 99.9% | No-ops, GitHub-integrated |
| **AWS EC2** | $10-20/mo | 25 min | 99.5% | Cloud-native, free tier eligible |
| **AWS ECS** | $50-150/mo | 45 min | 99.99% | Enterprise HA, scaling |

### Docker Strategy
- **Recommended** for VPS and cloud deployments
- **Non-root** container user (UID 1000)
- **Persistent volumes** for config/state/workspace
- **External binaries** must be baked into image (runtime installs don't persist)
- **Optional:** Per-session agent sandboxing (Docker-in-Docker)

### Service Management
- **systemd user service** (Linux desktop/VPS)
- **systemd system service** (multi-user/always-on)
- **launchd** (macOS)
- **Docker Compose restart policy** (container orchestration)
- All support automatic restart on failure

### Railway Integration
- **One-click deploy** from website
- **Built-in volume** at `/data` (persistent)
- **Web setup wizard** at `/setup` endpoint
- **Auto-scaling** and backups
- **HTTPS domain** included (*.railway.app)
- **Environment variables** for secrets (encrypted)
- **Backup export** at `/setup/export`

### AWS Deployment Patterns
1. **EC2 + Docker** (~$10-20/mo) - Simplest cloud option, same as VPS
2. **ECS Fargate** (~$50-150/mo) - Managed containers, multi-AZ, auto-scaling
3. **Lambda** - NOT recommended (stateful, long-running, websocket-heavy)
4. **Secrets Manager** - Recommended for storing API keys, database credentials
5. **EFS** - Persistent storage for state across tasks
6. **CloudWatch** - Built-in logging, monitoring, alarms

### Environment Variables & Secrets
**Precedence (highest → lowest):**
1. Process environment
2. `.env` in cwd
3. `~/.openclaw/.env` (global)
4. Config `env` block in openclaw.json
5. Shell import (optional)

**Key variables:**
- `OPENCLAW_HOME` - Override home directory
- `OPENCLAW_STATE_DIR` - State location
- `OPENCLAW_GATEWAY_TOKEN` - Gateway auth
- `ANTHROPIC_API_KEY` - LLM credentials
- `OPENCLAW_GATEWAY_BIND` - Binding mode (loopback/lan/*)

### Security Best Practices
- **Bind loopback** (127.0.0.1) by default
- **Access via SSH tunnel** from laptop (recommended)
- **Use Tailscale** as better alternative to SSH tunnel
- **Token-based auth** for gateway (not exposed publicly)
- **Firewall UFW** on Linux (block port 18789, allow SSH)
- **HTTPS** via nginx/Caddy reverse proxy (if public)
- **Secrets Manager** for API keys (AWS, Railway, Vault)

### Networking Strategy
```
Recommended (Secure):
VPS/Container (bind=loopback)
├── SSH Tunnel from Laptop
└─→ http://127.0.0.1:18789

Alternative (Modern):
VPS/Container (bind=loopback)
├── Tailscale VPN
└─→ http://100.x.x.x:18789  (Tailscale IP)
```

### Monitoring & Health Checks
- **Health endpoint:** `GET /health` with token
- **Docker healthcheck:** Built-in container health probes
- **systemd journalctl:** Real-time log streaming
- **CloudWatch:** Logs, metrics, alarms (AWS)
- **Datadog/Prometheus:** Advanced observability

### Backup & Recovery
- **What to backup:** `~/.openclaw/` (config, state, workspace)
- **Frequency:** Daily minimum
- **Destination:** Off-site (S3, B2, rsync)
- **Retention:** 30+ days
- **Recovery:** Extract backup, restart gateway

---

## 🔧 Platform Recommendations

### For Development
**➜ Local Installation**
- Simplest path (`npm install -g openclaw@latest`)
- Full control
- Zero cost
- Use for testing features before production

### For Small Businesses / Individuals
**➜ VPS + Docker (Hetzner CPX11, ~$5-10/mo)**
- Best value/reliability ratio
- 20-minute setup
- Full control over infrastructure
- Same setup works on DigitalOcean, Linode, Vultr

### For Teams / No-Ops Preference
**➜ Railway (~$15-50/mo)**
- Fastest deployment (2 minutes)
- GitHub-integrated CI/CD
- Built-in backups & monitoring
- Best UX for non-DevOps people
- Data export available for migration

### For Enterprise / High Availability
**➜ AWS ECS Fargate (~$50-150/mo)**
- Multi-AZ redundancy
- Auto-scaling
- CloudWatch integration
- Audit logs (CloudTrail)
- IAM security
- Terraform IaC available

### Fallback for AWS Free Tier
**➜ AWS EC2 t3.micro (~$10-20/mo or free)**
- Same as VPS approach
- Elastic IP for static address
- EBS snapshots for backups
- Easy to upgrade to ECS later

---

## 📚 Documentation Files Created

All files saved in `/mnt/c/Users/User/.openclaw/workspace/`:

1. **OPENCLAW_DEPLOYMENT_GUIDE.md** (36KB)
   - Complete technical reference
   - All platforms, all aspects
   - Best for: Comprehensive understanding

2. **DEPLOYMENT_QUICK_START.md** (10KB)
   - Decision tree + step-by-step
   - Practical instructions
   - Best for: Getting deployed quickly

3. **ADVANCED_DEPLOYMENT_PATTERNS.md** (22KB)
   - Multi-agent, scaling, HA
   - Terraform examples
   - Troubleshooting runbooks
   - Best for: Production operations

---

## 🚀 Quick Deploy Paths

### Fastest (2 min)
```
Railway Deploy Button
└─ Complete web wizard
```

### Most Reliable (~20 min)
```
Hetzner VPS + docker-compose
├─ ./docker-setup.sh
└─ SSH tunnel for access
```

### Best for Development
```
npm install -g openclaw
openclaw onboard
```

### Enterprise (45 min)
```
AWS ECS + Terraform
├─ ECR image
├─ EFS storage
└─ Multi-AZ + Load Balancer
```

---

## ✅ Completeness Checklist

- [x] Remote deployment architecture documented
- [x] Docker containerization explained
- [x] systemd/daemon integration covered
- [x] Railway deployment patterns documented
- [x] AWS patterns (ECS, EC2, Lambda) covered
- [x] Environment variables & configuration explained
- [x] Secrets management strategies outlined
- [x] Networking & security best practices documented
- [x] Monitoring & health checks covered
- [x] Backup & disaster recovery strategies included
- [x] Troubleshooting runbooks provided
- [x] Performance tuning guidance included
- [x] Multi-agent deployment patterns described
- [x] Cost comparison matrix provided
- [x] Quick-start guides created
- [x] Advanced patterns documented

---

## 📖 How to Use These Documents

**Start here:** `DEPLOYMENT_QUICK_START.md`
- Choose your platform
- Follow the step-by-step guide

**Deep dive:** `OPENCLAW_DEPLOYMENT_GUIDE.md`
- Understand the architecture
- Learn best practices
- Reference specific topics

**Advanced:** `ADVANCED_DEPLOYMENT_PATTERNS.md`
- Production architectures
- Multi-agent setups
- Troubleshooting
- Performance tuning

---

## 🎓 Key Learnings

1. **OpenClaw is deployment-agnostic** - Works anywhere Node.js runs (local, VPS, containers, serverless-adjacent)

2. **Persistence is critical** - All state must be on host volumes/EFS, not in containers or ephemeral storage

3. **Secrets management matters** - Use env vars locally, Secrets Manager on AWS, Railway encrypted variables, not config files

4. **Networking simplicity** - Loopback binding + SSH tunnel is simpler and more secure than public exposure

5. **Railway is the easiest** - If you want to just run OpenClaw without infrastructure knowledge, Railway is the path

6. **VPS gives best value** - $5-10/mo for reliable production-like experience with full control

7. **Docker/Compose is standard** - Virtually all deployments use containerization for isolation and repeatability

8. **Multi-deployment flexibility** - Same Gateway code works on laptop, VPS, Railway, and AWS with minimal config changes

---

## 📞 Resources

**Official Docs:** https://docs.openclaw.ai  
**GitHub:** https://github.com/openclaw/openclaw  
**Railway Deploy:** https://railway.com/deploy/openclaw  
**Community Discord:** https://discord.gg/clawd  

---

**Research Completed:** February 11, 2026 15:32 EST  
**Document Version:** 1.0  
**Ready for:** Production deployment decisions and implementation
