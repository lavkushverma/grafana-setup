# grafana-setup
![Image](https://grafana.com/media/docs/tempo/tempo_arch.png)

![Image](https://grafana.com/media/docs/grafana/dashboards-overview/complex-dashboard-example.png)

![Image](https://prometheus.io/assets/docs/grafana_configuring_datasource.png)

![Image](https://www.robustperception.io/wp-content/uploads/2019/07/Screenshot_2019-07-01_13-37-15.png)

## 📊 **Grafana Setup – Complete Hands-On Guide (Beginner → Production)**

This guide explains **how to set up Grafana step by step**, with **real-world options**, **commands**, and **interview-ready explanations**.

We’ll cover:

1. What Grafana is
2. Local / EC2 setup
3. Data source setup (Prometheus & CloudWatch)
4. Dashboard creation
5. Production best practices

Used with:

* **Grafana Labs**
* **Amazon EC2**
* **Amazon CloudWatch**
* **Prometheus**

---

# 1️⃣ What is Grafana?

**Grafana** is an **open-source visualization and monitoring tool** used to:

* Visualize metrics
* Build dashboards
* Set alerts
* Monitor infrastructure & applications

📌 Grafana **does not store data** – it only **queries data sources**.

---

# 2️⃣ Grafana Architecture (Simple)

```
Application / Infra
        ↓
 Metrics (Prometheus / CloudWatch / OpenSearch)
        ↓
       Grafana
        ↓
   Dashboards & Alerts
```

---

# 3️⃣ Option A: Install Grafana on EC2 (Most Common)

### Step 1: Launch EC2

* Amazon Linux 2
* Port **3000** open in Security Group

---

### Step 2: Install Grafana

```bash
sudo yum update -y
sudo tee /etc/yum.repos.d/grafana.repo <<EOF
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
EOF

sudo yum install grafana -y
```

---

### Step 3: Start Grafana

```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

---

### Step 4: Access Grafana

```
http://<EC2-Public-IP>:3000
```

**Default login**

```
username: admin
password: admin
```

(Change password immediately)

---

# 4️⃣ Option B: Grafana using Docker (Quick Setup)

```bash
docker run -d \
-p 3000:3000 \
--name grafana \
grafana/grafana
```

---

# 5️⃣ Add Data Source (MOST IMPORTANT)

## 5.1 Prometheus Data Source

### Steps:

```
Grafana → Settings → Data Sources → Add data source → Prometheus
```

### URL:

```
http://<prometheus-ip>:9090
```

Click **Save & Test**

📌 Used for:

* Kubernetes metrics
* Node metrics
* Application metrics

---

## 5.2 CloudWatch Data Source (AWS)

### IAM Role (Attach to EC2)

```json
{
  "Effect": "Allow",
  "Action": [
    "cloudwatch:ListMetrics",
    "cloudwatch:GetMetricData",
    "cloudwatch:GetMetricStatistics"
  ],
  "Resource": "*"
}
```

### Grafana Setup

```
Data Source → CloudWatch
Authentication → AWS SDK default
Region → ap-south-1
```

Click **Save & Test**

📌 Used for:

* EC2 CPU
* ALB latency
* ECS metrics
* RDS metrics

---

# 6️⃣ Create Your First Dashboard

### Steps:

1. ➕ Create → Dashboard
2. Add new panel
3. Choose Data Source
4. Select Metric
5. Choose Visualization (Graph / Stat / Gauge)
6. Save Dashboard

### Example:

* EC2 CPU Utilization
* ECS Service CPU
* ALB Request Count

---

# 7️⃣ Alerts Setup (Production)

### Example Alert:

```
CPU > 80% for 5 minutes → Send email / Slack
```

Grafana supports:

* Email
* Slack
* PagerDuty
* Webhook

---

# 8️⃣ (Optional) Grafana + Kubernetes (EKS)

### Common setup:

* Prometheus + node-exporter
* kube-state-metrics
* Grafana dashboards (imported)

Popular dashboards:

* Kubernetes cluster overview
* Node metrics
* Pod metrics

---

# 9️⃣ Security Best Practices

✔️ Change default admin password
✔️ Use **IAM roles** (no access keys)
✔️ Enable HTTPS (ALB + ACM)
✔️ Restrict port 3000
✔️ Enable RBAC (teams & folders)

---

# 🔟 Common Issues & Fixes

| Issue               | Cause                 | Fix                  |
| ------------------- | --------------------- | -------------------- |
| Grafana not loading | Port blocked          | Open 3000            |
| No metrics          | Wrong data source URL | Check endpoint       |
| Access denied       | IAM missing           | Add CloudWatch perms |
| High load           | Too many panels       | Optimize queries     |

