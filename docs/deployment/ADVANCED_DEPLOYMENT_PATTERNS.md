# Advanced OpenClaw Deployment Patterns
**Author:** openai-codex/gpt-5.1-codex-mini  
**Model:** openai-codex/gpt-5.1-codex-mini  
**Generated on:** 11 February 2026  

## Production Architectures, Multi-Agent, Scaling, and Troubleshooting

---

## Part 1: Railway Advanced Configuration

### Multi-Stage Deployments (Dev/Staging/Prod)

**Goal:** Use same codebase with different configurations per environment

```yaml
# railway.toml (in repo root)
[build]
builder = "dockerfile"

[deploy]
startCommand = "node dist/index.js"

# Environment-specific overrides
[environments.development]
  ENVIRONMENT=development
  SETUP_PASSWORD=dev-password
  OPENCLAW_GATEWAY_TOKEN=dev-token

[environments.staging]
  ENVIRONMENT=staging
  SETUP_PASSWORD=staging-password
  OPENCLAW_GATEWAY_TOKEN=staging-token

[environments.production]
  ENVIRONMENT=production
  SETUP_PASSWORD=${PROD_SETUP_PASSWORD}
  OPENCLAW_GATEWAY_TOKEN=${PROD_GATEWAY_TOKEN}
```

**Deploy to specific environment:**
```bash
# Railway CLI
railway up --environment production

# In GitHub Actions (optional)
- name: Deploy to Railway
  run: railway up --environment production
```

### Railway + GitHub Actions (CI/CD)

**Workflow: Auto-deploy on push to main**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Railway

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: railway-app/action@v1
        with:
          token: ${{ secrets.RAILWAY_TOKEN }}
          service: openclaw-gateway
          environment: production
```

### Railway Volume Strategy

**Problem:** Volumes persist between deployments, but can cause config drift

**Solution:** Separate data volumes by type

```yaml
# docker-compose.yml (in Railway deployment)
services:
  openclaw-gateway:
    volumes:
      - openclaw-config:/data/.openclaw        # Config (read-write)
      - openclaw-workspace:/data/workspace     # Agent code (read-write)
      - openclaw-cache:/data/.cache            # Temp/cache (safe to reset)

volumes:
  openclaw-config:
    driver: local
  openclaw-workspace:
    driver: local
  openclaw-cache:
    driver: local
```

### Railway Scaling & Resource Management

**Increase resources if hitting limits:**

```
Railway Dashboard
├── Service → Settings → Instance Type
│   ├── Railway.micro → Railway.small (+memory)
│   └── Railway.small → Railway.large (+CPU/memory)
└── Service → Settings → Concurrency
    └── Set max concurrent connections
```

**Monitor resource usage:**
```
Railway Dashboard
├── Deployments → Logs
└── Look for: OOM (Out of Memory), CPU throttling, timeout errors
```

### Railway Backup & Export

**Automated daily export to GitHub:**

```yaml
# .github/workflows/backup.yml
name: Backup OpenClaw State

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Download backup
        run: |
          curl -o backup.tar.gz https://${{ secrets.RAILWAY_DOMAIN }}/setup/export \
            -H "Authorization: Bearer ${{ secrets.SETUP_PASSWORD }}"
      
      - name: Push to GitHub (encrypted)
        run: |
          git add backup.tar.gz
          git commit -m "Daily backup $(date +%Y%m%d)"
          git push
```

---

## Part 2: AWS Advanced Patterns

### AWS ECS Fargate Architecture (Detailed)

**Full stack with HA:**

```yaml
# terraform/main.tf (Simplified)

# VPC
resource "aws_vpc" "openclaw" {
  cidr_block = "10.0.0.0/16"
}

# Public & Private Subnets across 2 AZs
resource "aws_subnet" "public_1a" {
  vpc_id            = aws_vpc.openclaw.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "private_1a" {
  vpc_id            = aws_vpc.openclaw.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "private_1b" {
  vpc_id            = aws_vpc.openclaw.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "us-east-1b"
}

# NAT Gateway (for outbound traffic from private subnets)
resource "aws_nat_gateway" "openclaw" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_1a.id
}

# EFS (persistent storage)
resource "aws_efs_file_system" "openclaw" {
  creation_token = "openclaw-efs"
  encrypted      = true
  performance_mode = "generalPurpose"
}

# Mount targets in private subnets
resource "aws_efs_mount_target" "openclaw_1a" {
  file_system_id      = aws_efs_file_system.openclaw.id
  subnet_id           = aws_subnet.private_1a.id
  security_groups     = [aws_security_group.efs.id]
}

resource "aws_efs_mount_target" "openclaw_1b" {
  file_system_id      = aws_efs_file_system.openclaw.id
  subnet_id           = aws_subnet.private_1b.id
  security_groups     = [aws_security_group.efs.id]
}

# ECR Repository
resource "aws_ecr_repository" "openclaw" {
  name                 = "openclaw"
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration {
    scan_on_push = true
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "openclaw" {
  name = "openclaw-cluster"
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "openclaw" {
  name              = "/ecs/openclaw"
  retention_in_days = 30
}

# IAM Role for ECS Task
resource "aws_iam_role" "ecs_task_role" {
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# IAM Policy for Secrets Manager access
resource "aws_iam_role_policy" "ecs_task_policy" {
  role = aws_iam_role.ecs_task_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "kms:Decrypt"
      ]
      Resource = "*"
    }]
  })
}

# ECS Task Definition
resource "aws_ecs_task_definition" "openclaw" {
  family                   = "openclaw"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([{
    name      = "openclaw-gateway"
    image     = "${aws_ecr_repository.openclaw.repository_url}:latest"
    essential = true

    portMappings = [{
      containerPort = 18789
      hostPort      = 18789
      protocol      = "tcp"
    }]

    mountPoints = [{
      sourceVolume  = "openclaw-storage"
      containerPath = "/data"
      readOnly      = false
    }]

    environment = [
      { name = "NODE_ENV", value = "production" },
      { name = "OPENCLAW_GATEWAY_BIND", value = "0.0.0.0" },
      { name = "OPENCLAW_GATEWAY_PORT", value = "18789" },
      { name = "OPENCLAW_STATE_DIR", value = "/data/.openclaw" },
      { name = "OPENCLAW_WORKSPACE_DIR", value = "/data/workspace" }
    ]

    secrets = [
      { name = "OPENCLAW_GATEWAY_TOKEN", valueFrom = "${aws_secretsmanager_secret.gateway_token.arn}" },
      { name = "ANTHROPIC_API_KEY", valueFrom = "${aws_secretsmanager_secret.anthropic_key.arn}" }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.openclaw.name
        "awslogs-region"        = "us-east-1"
        "awslogs-stream-prefix" = "ecs"
      }
    }

    healthCheck = {
      command     = ["CMD-SHELL", "node dist/index.js health --token $OPENCLAW_GATEWAY_TOKEN || exit 1"]
      interval    = 30
      timeout     = 10
      retries     = 3
      startPeriod = 60
    }
  }])

  volume {
    name = "openclaw-storage"
    efs_volume_configuration {
      file_system_id          = aws_efs_file_system.openclaw.id
      root_directory          = "/data"
      transit_encryption      = "ENABLED"
      authorization_config {
        access_point_id = aws_efs_access_point.openclaw.id
      }
    }
  }
}

# Application Load Balancer
resource "aws_lb" "openclaw" {
  name               = "openclaw-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.public_1a.id, aws_subnet.public_1b.id]
}

# Target Group
resource "aws_lb_target_group" "openclaw" {
  name        = "openclaw-tg"
  port        = 18789
  protocol    = "HTTP"
  vpc_id      = aws_vpc.openclaw.id
  target_type = "ip"

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }
}

# HTTPS Listener (requires ACM cert)
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.openclaw.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.openclaw.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.openclaw.arn
  }
}

# Redirect HTTP → HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.openclaw.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ECS Service (launches and manages tasks)
resource "aws_ecs_service" "openclaw" {
  name            = "openclaw-service"
  cluster         = aws_ecs_cluster.openclaw.id
  task_definition = aws_ecs_task_definition.openclaw.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = [aws_subnet.private_1a.id, aws_subnet.private_1b.id]
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.openclaw.arn
    container_name   = "openclaw-gateway"
    container_port   = 18789
  }

  depends_on = [aws_lb_listener.https]
}

# Auto Scaling Target
resource "aws_autoscaling_target" "openclaw" {
  max_capacity       = 4
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.openclaw.name}/${aws_ecs_service.openclaw.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Auto Scaling Policy (CPU-based)
resource "aws_autoscaling_policy" "openclaw_cpu" {
  policy_type            = "TargetTrackingScaling"
  name                   = "openclaw-cpu-scaling"
  resource_id            = aws_autoscaling_target.openclaw.resource_id
  scalable_dimension     = aws_autoscaling_target.openclaw.scalable_dimension
  service_namespace      = aws_autoscaling_target.openclaw.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# Secrets Manager
resource "aws_secretsmanager_secret" "anthropic_key" {
  name = "openclaw/anthropic-api-key"
}

resource "aws_secretsmanager_secret_version" "anthropic_key" {
  secret_id     = aws_secretsmanager_secret.anthropic_key.id
  secret_string = var.anthropic_api_key
}

resource "aws_secretsmanager_secret" "gateway_token" {
  name = "openclaw/gateway-token"
}

resource "aws_secretsmanager_secret_version" "gateway_token" {
  secret_id     = aws_secretsmanager_secret.gateway_token.id
  secret_string = var.gateway_token
}
```

### AWS Disaster Recovery

**RTO (Recovery Time Objective): ~5 minutes**  
**RPO (Recovery Point Objective): ~1 hour**

**Strategy:**

```
Primary Region (us-east-1)
├── ECS Service (2 tasks)
├── EFS (backed by AWS Backup)
└── RDS (multi-AZ, daily snapshots)
        ↓ (automated replication)
Secondary Region (us-west-2) [Standby]
├── Automated failover on primary failure
└── Route53 health checks trigger DNS switch
```

**Backup configuration:**

```bash
# AWS Backup plan for EFS
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "openclaw-daily",
    "Rules": [{
      "RuleName": "DailyBackups",
      "TargetBackupVaultName": "openclaw-vault",
      "ScheduleExpression": "cron(0 2 * * *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 120,
      "Lifecycle": {
        "DeleteAfterDays": 30
      }
    }]
  }'
```

---

## Part 3: Multi-Agent Deployment

### Multiple Agents on One Gateway

**Use case:** Personal assistant + family agent + work agent

**Configuration:**

```json
{
  "identity": {
    "name": "Master Gateway"
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-20250514",
      "workspace": "~/.openclaw/workspace"
    },
    "list": [
      {
        "id": "personal",
        "name": "Personal Assistant",
        "workspace": "~/.openclaw/workspace/personal",
        "model": "anthropic/claude-opus-4-20250514"
      },
      {
        "id": "family",
        "name": "Family Assistant",
        "workspace": "~/.openclaw/workspace/family",
        "model": "anthropic/claude-opus-4-20250514",
        "tools": {
          "sandbox": {
            "tools": {
              "deny": ["exec", "shell", "read", "write"]
            }
          }
        }
      },
      {
        "id": "work",
        "name": "Work Assistant",
        "workspace": "~/.openclaw/workspace/work",
        "model": "anthropic/claude-opus-4-20250514",
        "sandbox": {
          "mode": "off"
        }
      }
    ]
  },
  "channels": {
    "telegram": {
      "routing": {
        "default": "personal",
        "overrides": {
          "FAMILY_CHAT_ID": "family",
          "WORK_CHAT_ID": "work"
        }
      }
    }
  }
}
```

### Per-Agent Secrets & Credentials

**Example: Each agent has own credentials**

```json
{
  "agents": {
    "list": [
      {
        "id": "work",
        "models": {
          "providers": {
            "work-api": {
              "apiKey": "${WORK_API_KEY}"
            }
          }
        }
      }
    ]
  }
}
```

**Environment:**
```bash
# ~/.openclaw/.env
WORK_API_KEY=company-key-here
PERSONAL_API_KEY=personal-key-here
```

### Agent Sandboxing (Advanced)

**Isolate agent tool execution:**

```json
{
  "agents": {
    "list": [
      {
        "id": "untrusted-agent",
        "sandbox": {
          "mode": "all",
          "scope": "session",
          "docker": {
            "image": "openclaw-sandbox:bookworm-slim",
            "network": "none",
            "memory": "512m",
            "cpus": 0.5,
            "user": "1000:1000",
            "capDrop": ["ALL"],
            "readOnlyRoot": true,
            "ulimits": {
              "nofile": { "soft": 1024, "hard": 1024 },
              "nproc": 256
            }
          }
        }
      }
    ]
  }
}
```

---

## Part 4: Networking & Security Advanced

### Tailscale Integration (Recommended for Remote Access)

**Better than SSH tunnel + more secure**

```json
{
  "gateway": {
    "tailscale": {
      "enabled": true,
      "authKeyPath": "/var/lib/openclaw/.tailscale-auth-key",
      "port": 18789
    }
  }
}
```

**Setup:**

```bash
# Install Tailscale on VPS
curl -fsSL https://tailscale.com/install.sh | sh

# Get auth key from https://login.tailscale.com/admin/settings/keys

# Start Tailscale
sudo tailscale up --auth-key=<auth-key> --operator=openclaw

# Tailscale IP appears in:
tailscale ip -4

# Access from any connected device
open http://100.100.100.1:18789  # Tailscale IP
```

**Pros over SSH tunnel:**
- No port forwarding
- Works on mobile
- Encrypted end-to-end
- Works across networks

### Mutual TLS (mTLS) for APIs

**If exposing OpenAI-compatible API endpoint:**

```json
{
  "gateway": {
    "tls": {
      "enabled": true,
      "certPath": "/etc/openclaw/cert.pem",
      "keyPath": "/etc/openclaw/key.pem",
      "clientAuthRequired": true,
      "clientCertPath": "/etc/openclaw/client-certs.pem"
    }
  }
}
```

**Generate certificates:**

```bash
# Self-signed for testing
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Client certificate
openssl req -newkey rsa:4096 -keyout client-key.pem -out client.csr
openssl x509 -req -in client.csr -CA cert.pem -CAkey key.pem -out client-cert.pem -days 365
```

---

## Part 5: Monitoring & Observability

### CloudWatch Detailed Monitoring (AWS)

**Configure detailed metrics:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name openclaw-memory-high \
  --alarm-description "Alert when OpenClaw memory exceeds 80%" \
  --metric-name MemoryUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456:alerts
```

### Datadog Integration

**Ship logs to Datadog:**

```bash
# On VPS/ECS, configure Datadog agent
DD_AGENT_MAJOR_VERSION=7 bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"

# Add OpenClaw log collection
cat > /etc/datadog-agent/conf.d/openclaw.d/conf.yaml <<'EOF'
logs:
  - type: file
    path: /var/log/openclaw/*.log
    service: openclaw
    source: openclaw
    tags:
      - environment:production
EOF
```

### Prometheus + Grafana

**Export metrics from OpenClaw:**

```json
{
  "monitoring": {
    "prometheus": {
      "enabled": true,
      "port": 9090,
      "path": "/metrics"
    }
  }
}
```

**Scrape config for Prometheus:**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: openclaw
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'
```

**Grafana dashboard:**
- Memory usage over time
- Request latency (p50, p95, p99)
- Error rate (4xx, 5xx)
- Message throughput (msg/sec)
- Gateway uptime

---

## Part 6: Advanced Troubleshooting

### Debug Logs

**Enable verbose logging:**

```bash
# CLI
openclaw gateway --verbose --log-level debug

# Or in config
{
  "logging": {
    "level": "debug",
    "format": "json"
  }
}
```

### Memory Profiling

**If memory usage is high:**

```bash
# Generate heap dump
node --inspect dist/index.js &
# Visit chrome://inspect in Chrome
# Capture heap snapshot

# Or use clinic.js
npx clinic doctor -- node dist/index.js
```

### Database Queries (if using RDS)

**Check slow queries:**

```bash
# AWS RDS Performance Insights
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier-type DB_INSTANCE \
  --identifier-values db-instance-id
```

### Network Issues

**Diagnose connectivity:**

```bash
# Test gateway endpoint
curl -v http://127.0.0.1:18789/health

# Check DNS resolution
nslookup your-domain.com

# Trace network path
tracert openclaw.example.com

# Inspect firewall rules
sudo iptables -L -n -v  # Linux
Get-NetFirewallRule | Get-NetFirewallPortFilter -PolicyStore ActiveStore  # Windows
```

### WebSocket Connection Issues

**If Control UI shows "disconnected":**

```bash
# Browser console (F12)
// Check WebSocket connection
ws://127.0.0.1:18789/gateway

# View gateway logs
openclaw logs | grep -i websocket

# Restart and reconnect
systemctl restart openclaw-gateway
# Refresh browser
```

---

## Part 7: Disaster Recovery Runbook

### Data Loss Scenario

**Scenario:** Host disk fails, container lost

**Recovery (assuming backups exist):**

```bash
# 1. Stop current gateway
docker compose down

# 2. Restore from backup
tar -xzf openclaw-backup-latest.tar.gz -C /

# 3. Verify backup integrity
ls -la ~/.openclaw/
file ~/.openclaw/openclaw.json

# 4. Restart gateway
docker compose up -d openclaw-gateway

# 5. Verify channels and data
openclaw channels list
openclaw status

# 6. Check if messages are flowing
# Send test message from Telegram/Discord
```

### Credential Leak Scenario

**Scenario:** API key accidentally committed to GitHub

**Response (immediate):**

```bash
# 1. Revoke old key in provider console
# (Anthropic/OpenAI/etc. - takes ~5 minutes)

# 2. Generate new key

# 3. Update gateway
openclaw config set env.vars.ANTHROPIC_API_KEY "sk-ant-new-key"

# 4. Restart
systemctl restart openclaw-gateway

# 5. Verify
openclaw models status

# 6. Check git history
git log --all --full-history -- <file-with-key>
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch <file>' \
  --prune-empty --tag-name-filter cat -- --all
```

### Corrupted Config Scenario

**Scenario:** openclaw.json is invalid JSON

**Recovery:**

```bash
# 1. Diagnosis
openclaw doctor
# Shows: "Malformed JSON in openclaw.json"

# 2. Restore from backup
cp ~/.openclaw/openclaw.json.backup ~/.openclaw/openclaw.json

# 3. Or manually fix
# - Validate JSON syntax
# - Check for missing commas/brackets

# 4. Verify
openclaw doctor

# 5. Restart
systemctl restart openclaw-gateway
```

---

## Part 8: Performance Tuning

### Node.js Optimization

```bash
# Increase max file descriptors
ulimit -n 65535

# Enable clustering for multi-CPU systems
export NODE_OPTIONS="--max-old-space-size=2048"

# JIT compilation
export NODE_ENV=production  # Enables V8 optimizations
```

### Gateway Configuration Tuning

```json
{
  "gateway": {
    "cache": {
      "enabled": true,
      "ttlMs": 300000
    },
    "session": {
      "idleTimeoutMs": 3600000,
      "maxConcurrent": 100
    },
    "websocket": {
      "readTimeoutMs": 30000,
      "writeTimeoutMs": 30000
    }
  }
}
```

### Container Resource Limits

```yaml
# docker-compose.yml
services:
  openclaw-gateway:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
```

---

**Document Status:** Complete  
**Last Updated:** February 11, 2026  
**Target Audience:** DevOps/SRE engineers, infrastructure architects
