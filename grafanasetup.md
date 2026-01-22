Setting up **Grafana** is the standard way to visualize metrics (like CPU usage, logs, or website traffic).

Since you are already working with **AWS EC2**, I will show you the most common way to set it up: **Installing Grafana on an Amazon Linux EC2 instance.**

---

### Phase 1: Prepare the Server (Security Group)
Grafana runs on **Port 3000** by default. You must open this port in your AWS firewall.

1.  Go to **EC2 Console** -> **Security Groups**.
2.  Select your instance's Security Group.
3.  **Edit Inbound Rules** -> **Add Rule**.
    *   **Type:** Custom TCP.
    *   **Port:** `3000`.
    *   **Source:** Anywhere (`0.0.0.0/0`).
4.  Save Rules.

---

### Phase 2: Install Grafana (The Commands)
Connect to your EC2 instance via Git Bash (`ssh ec2-user@...`) and run these commands.

**1. Create the Repository File**
Amazon Linux doesn't have Grafana by default. We need to tell `yum` where to find it.
```bash
sudo nano /etc/yum.repos.d/grafana.repo
```

**2. Paste this content** (Right-click to paste):
```ini
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```
*(Save and Exit: `Ctrl+O`, `Enter`, `Ctrl+X`)*

**3. Install Grafana**
```bash
sudo yum install grafana -y
```

**4. Start the Server**
```bash
# Reload system configuration
sudo systemctl daemon-reload

# Start Grafana
sudo systemctl start grafana-server

# Enable it (so it restarts if the server reboots)
sudo systemctl enable grafana-server

# Check status (Should say "Active/Running")
sudo systemctl status grafana-server
```

---

### Phase 3: Log In
1.  Open your web browser.
2.  Type your EC2 Public IP followed by port 3000:
    `http://<YOUR-EC2-PUBLIC-IP>:3000`
3.  **Login Screen:**
    *   **Username:** `admin`
    *   **Password:** `admin`
4.  It will ask you to set a new password. You can skip or set one.

---

### Phase 4: Create Your First Dashboard (AWS CloudWatch)
Grafana is empty right now. Let's make it show **AWS EC2 CPU Usage**.

**1. Add Data Source**
*   On the left menu, go to **Connections** (or Configuration gear icon) -> **Data Sources**.
*   Click **Add data source**.
*   Search for **CloudWatch** and select it.
*   **Authentication Provider:** Choose **Access & secret key**.
    *   *Paste the AWS Access Key / Secret Key you used in previous steps.*
*   **Default Region:** Choose your region (e.g., `us-east-1`).
*   Click **Save & test**. (You should see green text: "Data source is working").

**2. Create Dashboard**
*   Click the **Plus (+)** icon on the top right -> **New dashboard**.
*   Click **Add visualization**.
*   Select **CloudWatch** (the source you just added).

**3. Configure the Graph**
*   **Region:** `default`
*   **Namespace:** `AWS/EC2`
*   **Metric Name:** `CPUUtilization`
*   **Statistic:** `Average`
*   **Dimensions:** Select `InstanceId` = (Choose your EC2 instance ID).

🎉 **Result:** You will immediately see a line graph showing the CPU usage of your AWS server over the last few hours.

---

### Alternative: The "Docker" Way (Fastest)
If you don't want to install it on Linux and just want to run it quickly on your laptop (if you have Docker Desktop installed):

```bash
docker run -d -p 3000:3000 --name=grafana grafana/grafana
```
Then just go to `localhost:3000` in your browser.
