# 📊 Grafana + CloudWatch Monitoring on EC2

> **Production-grade observability stack** — Grafana deployed on Amazon EC2 with full CloudWatch metrics integration, automated via Terraform, with Docker-based runtime management.

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Account                          │
│                                                             │
│   Internet ──► Security Group ──► EC2 (Grafana:3000)        │
│                                       │                     │
│                              Docker + CW Agent              │
│                                       │                     │
│                              CloudWatch API ◄── All AWS      │
│                              (EC2, RDS, ELB, Lambda, Logs)  │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
| Component | Role |
|-----------|------|
| EC2 (t3.medium) | Hosts Grafana via Docker |
| IAM Role + Policy | Grants least-privilege CloudWatch read access |
| Security Group | Allows port 3000 (Grafana), 22 (SSH), 443 (HTTPS) |
| Elastic IP | Stable public endpoint |
| CloudWatch Agent | Pushes OS-level metrics (CPU, Mem, Disk) to CWAgent namespace |
| CloudWatch Alarm | CPU > 80% triggers SNS notification |

---

## 📁 Project Structure

```
grafana-cloudwatch-setup/
├── terraform/
│   ├── main.tf                    # Core infrastructure (EC2, IAM, SG, EIP, CW Alarm)
│   ├── variables.tf               # All input variables with defaults
│   ├── outputs.tf                 # Grafana URL, SSH command, IP outputs
│   ├── user-data.sh               # EC2 bootstrap: Docker + Grafana + CW Agent
│   └── terraform.tfvars.example   # Copy → terraform.tfvars, fill in secrets
├── scripts/
│   ├── install-grafana.sh         # Manual RPM-based Grafana install script
│   └── configure-cloudwatch.sh    # CW Agent config helper
├── grafana-config/
│   ├── datasource.yaml            # Auto-provisions CloudWatch datasource
│   └── dashboard.json             # Pre-built EC2 overview dashboard
└── README.md
```

---

## ✅ Prerequisites

```bash
# 1. Terraform >= 1.0
terraform -version

# 2. AWS CLI configured with sufficient permissions
aws configure
aws sts get-caller-identity   # Verify credentials

# 3. SSH key pair (generate if needed)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/grafana-key
cat ~/.ssh/grafana-key.pub    # Copy this into terraform.tfvars → public_key
```

**Required IAM permissions for the deploying user:**
`ec2:*`, `iam:CreateRole`, `iam:AttachRolePolicy`, `cloudwatch:*`, `sns:*`

---

## 🚀 Quick Deploy (Terraform)

```bash
# Step 1: Clone / create project directory
mkdir grafana-cloudwatch && cd grafana-cloudwatch

# Step 2: Configure variables
cd terraform
cp terraform.tfvars.example terraform.tfvars
vi terraform.tfvars          # Fill in required values (see below)

# Step 3: Initialize and deploy
terraform init
terraform plan               # Review resources before applying
terraform apply              # Type 'yes' to confirm

# Step 4: Get access details
terraform output grafana_url
terraform output ssh_command
```

### `terraform.tfvars` — Required Values

```hcl
aws_region   = "us-east-1"
project_name = "grafana-monitoring"

# EC2
instance_type    = "t3.medium"
root_volume_size = 20

# Networking — RESTRICT IN PRODUCTION
allowed_cidr_blocks     = ["YOUR.OFFICE.IP/32"]
ssh_allowed_cidr_blocks = ["YOUR.OFFICE.IP/32"]

# SSH key
create_key_pair = true
public_key      = "ssh-rsa AAAAB3... <output of cat ~/.ssh/grafana-key.pub>"

# Grafana admin password — use a strong password
grafana_admin_password = "Change@Me2024!"

# Optional — SNS topic for CPU alarms
sns_topic_arn = ""
```

---

## 🛠️ Manual Installation (No Terraform)

Use this if you already have an existing EC2 instance.

### Step 1 — SSH into EC2

```bash
ssh -i ~/.ssh/your-key.pem ec2-user@<instance-public-ip>
```

### Step 2 — Install Grafana (RPM method)

```bash
# Add Grafana repo
sudo tee /etc/yum.repos.d/grafana.repo << 'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

# Install and start
sudo yum install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

### Step 3 — Install via Docker (Alternative)

```bash
# Install Docker
sudo amazon-linux-extras install docker -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

# Prepare directories
mkdir -p /opt/grafana/{data,logs,plugins,provisioning/datasources}
chmod 777 /opt/grafana/{data,logs,plugins}

# Run Grafana container
docker run -d \
  --name grafana \
  --restart unless-stopped \
  -p 3000:3000 \
  -e "GF_SECURITY_ADMIN_PASSWORD=YourSecurePassword" \
  -v /opt/grafana/data:/var/lib/grafana \
  -v /opt/grafana/provisioning:/etc/grafana/provisioning \
  grafana/grafana:latest
```

### Step 4 — Configure CloudWatch Datasource (Grafana UI)

1. Open browser → `http://<EC2-PUBLIC-IP>:3000`
2. Login: `admin` / `admin` → change password on first login
3. Navigate: **Connections → Data Sources → Add data source**
4. Select: **CloudWatch**
5. Configure:
   - **Auth Provider:** EC2 IAM Role *(no keys needed — IAM role on instance handles this)*
   - **Default Region:** `us-east-1`
6. Click **Save & Test** → should show ✅ green

### Step 5 — Install & Configure CloudWatch Agent

```bash
# Install agent
sudo yum install -y amazon-cloudwatch-agent

# Create config (captures CPU, Memory, Disk, Network)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# OR use pre-built config
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "cpu":    { "measurement": ["cpu_usage_idle","cpu_usage_user","cpu_usage_system"], "metrics_collection_interval": 60 },
      "mem":    { "measurement": ["mem_used_percent"], "metrics_collection_interval": 60 },
      "disk":   { "measurement": ["used_percent"], "metrics_collection_interval": 60, "resources": ["*"] },
      "swap":   { "measurement": ["swap_used_percent"], "metrics_collection_interval": 60 }
    }
  }
}
EOF

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

---

## 🔐 IAM Policy — CloudWatch Read Access

The EC2 instance needs this IAM role policy attached via instance profile:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "cloudwatch:DescribeAlarms",
      "cloudwatch:ListMetrics",
      "cloudwatch:GetMetricData",
      "cloudwatch:GetMetricStatistics",
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams",
      "logs:GetLogEvents",
      "logs:FilterLogEvents",
      "logs:StartQuery",
      "logs:StopQuery",
      "logs:GetQueryResults",
      "ec2:DescribeInstances",
      "rds:DescribeDBInstances",
      "tag:GetResources"
    ],
    "Resource": "*"
  }]
}
```

---

## 📈 CloudWatch Metrics Reference

| Namespace | Key Metrics | Notes |
|-----------|-------------|-------|
| `AWS/EC2` | CPUUtilization, NetworkIn/Out, StatusCheckFailed | Built-in, no agent needed |
| `AWS/ELB` | RequestCount, Latency, HTTPCode_ELB_5XX | ALB/NLB metrics |
| `AWS/Lambda` | Invocations, Errors, Duration, Throttles | Function-level |
| `AWS/RDS` | CPUUtilization, FreeStorageSpace, Connections | DB instance metrics |
| `CWAgent` | mem_used_percent, disk/used_percent, cpu_usage_* | Requires CW Agent on EC2 |

---

## 🔔 Alerting Setup

### Create SNS Topic + Email Subscription

```bash
# Create topic
aws sns create-topic --name grafana-alerts --region us-east-1

# Subscribe your email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:grafana-alerts \
  --protocol email \
  --notification-endpoint ops-team@yourcompany.com
```

### CloudWatch Alarm (CPU > 80%)
Already configured via Terraform `main.tf`. For manual creation:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-GrafanaServer" \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --period 300 \
  --statistic Average \
  --threshold 80 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:grafana-alerts \
  --dimensions Name=InstanceId,Value=i-xxxxxxxxx
```

---

## 🛡️ Production Security Checklist

- [ ] **Restrict `allowed_cidr_blocks`** to office/VPN IP only — never `0.0.0.0/0`
- [ ] **Enable HTTPS** — use ACM certificate with ALB in front of Grafana
- [ ] **Enable Grafana MFA** — Settings → Security → Two-Factor Auth
- [ ] **Rotate admin password** — Use AWS Secrets Manager, inject via parameter
- [ ] **Enable VPC Flow Logs** on the Grafana VPC
- [ ] **Enable CloudTrail** for API-level audit logging
- [ ] **EBS volume encryption** — already set to `encrypted = true` in Terraform
- [ ] **Disable anonymous access** — set `auth.anonymous.enabled = false` in `grafana.ini`

---

## 💰 Cost Estimate (us-east-1, On-Demand)

| Resource | Cost/Month |
|----------|-----------|
| EC2 t3.medium (On-Demand) | ~$30.37 |
| EBS gp3 20GB | ~$1.60 |
| Elastic IP (attached) | ~$0 *(free when attached)* |
| CloudWatch API calls | ~$1–5 |
| CloudWatch Agent metrics | ~$0.30 per metric/month |
| Data transfer | Variable |
| **Estimated Total** | **~$35–45/month** |

> 💡 **Save ~35%** by using a Reserved Instance (1-year, no upfront).

---

## 🔧 Maintenance & Updates

```bash
# Update Grafana (Docker)
cd /opt/grafana
docker-compose pull
docker-compose up -d

# Update Grafana (RPM)
sudo yum update grafana -y
sudo systemctl restart grafana-server

# Backup Grafana DB
sudo cp /var/lib/grafana/grafana.db ~/grafana-backup-$(date +%Y%m%d).db

# Check Grafana logs
sudo journalctl -u grafana-server -f
docker logs grafana --tail 100 -f
```

---

## 🐛 Troubleshooting

| Problem | Check | Fix |
|---------|-------|-----|
| Can't reach port 3000 | Security group inbound rules | Add port 3000 from your IP |
| CloudWatch datasource fails | IAM role attached to EC2 | Verify instance profile in EC2 console |
| Grafana won't start | Service logs | `sudo journalctl -u grafana-server -f` |
| No CWAgent metrics | Agent status | `sudo systemctl status amazon-cloudwatch-agent` |
| Docker container exits | Container logs | `docker logs grafana` |

```bash
# Validate IAM permissions from inside EC2
aws cloudwatch list-metrics --namespace AWS/EC2

# Check CW agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

# Test port 3000 locally
curl -s http://localhost:3000/api/health
```

---

## 🧹 Cleanup

```bash
# Destroy all Terraform-managed resources
cd terraform
terraform destroy

# Verify no orphaned resources
aws ec2 describe-instances --filters "Name=tag:Project,Values=Grafana Monitoring" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}'
```

---

## 📚 Resources

- [Grafana Documentation](https://grafana.com/docs/)
- [Grafana CloudWatch Plugin](https://grafana.com/docs/grafana/latest/datasources/cloudwatch/)
- [CloudWatch Metrics Reference](https://docs.aws.amazon.com/cloudwatch/latest/monitoring/aws-services-cloudwatch-metrics.html)
- [CloudWatch Agent Configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

*Maintained by Applied Cloud Computing — DevOps & Cloud Engineering Team*
