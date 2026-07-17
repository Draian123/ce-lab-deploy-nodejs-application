# Lab M2.07 - Deploy Node.js Application

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-deploy-nodejs-application](https://github.com/cloud-engineering-bootcamp/ce-lab-deploy-nodejs-application)

**Activity Type:** Individual  
**Estimated Time:** 45-60 minutes

## Learning Objectives

- [ ] Deploy complete Node.js application to EC2
- [ ] Configure environment variables securely
- [ ] Set up application as systemd service
- [ ] Implement health checks
- [ ] Monitor application with PM2

## Prerequisites

- [ ] EC2 instance running Amazon Linux 2023
- [ ] SSH access to the instance
- [ ] Security group allows HTTP (80)

---

## Introduction

Running an app with `node app.js &` is not a deployment - it dies when the SSH session ends, does not restart after a crash, and does not return after a reboot. A real deployment restarts automatically, survives a reboot, runs in a defined environment, and exposes a health check. This lab uses PM2 and Nginx to provide all of this.

## Scenario

An application has been running as a backgrounded shell process and has gone down twice - once when an SSH session closed, once after a reboot. You have been asked to deploy it properly and verify it survives a reboot before closing the task.

---

## Your Task

Deploy a production-ready Node.js Express application:
1. Install Node.js and dependencies
2. Create the application code
3. Configure environment variables
4. Set up PM2 process manager
5. Configure Nginx reverse proxy
6. Add health check endpoint

**Time limit:** 45-60 minutes

---

## Step-by-Step Instructions

### Step 1: Install Node.js

```bash
sudo dnf install -y nodejs   # Amazon Linux 2023 has Node.js in its repos
node -v
```

**Expected outcome:** A version number prints.

---

### Step 2: Create the Application

```bash
mkdir -p ~/app && cd ~/app
npm init -y
npm install express
```

```bash
cat > app.js <<'EOF'
const express = require('express');
const os = require('os');
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  res.json({
    message: 'Week 2 Deployment Lab',
    hostname: os.hostname(),
    uptime: process.uptime(),
    environment: process.env.NODE_ENV || 'development'
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date() });
});

app.listen(port, () => console.log(`Server running on port ${port}`));
EOF
```

`uptime` resets to near zero after a reboot (used in Step 8), and `environment` shows `development` until `NODE_ENV` is set in Step 3.

---

### Step 3: Install and Start PM2

Install PM2, then define the app in a PM2 ecosystem file. This file is your **PM2 configuration** deliverable, and its `env` block sets `NODE_ENV` cleanly:

```bash
sudo npm install -g pm2   # install PM2 globally

cat > ~/app/ecosystem.config.js <<'EOF'
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'app.js',
    env: { NODE_ENV: 'production', PORT: 8080 }
  }]
};
EOF

cd ~/app
pm2 start ecosystem.config.js   # starts myapp with NODE_ENV=production
pm2 list
```

**Expected outcome:** `status` is `online`.

> If the restart count (`↺`) keeps rising, the app is crash-looping. Check `pm2 logs myapp --lines 50`.

Verify locally:
```bash
curl localhost:8080/health   # {"status":"healthy",...}
curl localhost:8080/         # "environment":"production"
```

---

### 📸 Screenshot Required

**Filename**

```
screenshots/01-pm2-list-online.png
```

**Capture**

`pm2 list` showing `myapp` as `online` with a restart count of 0.

**Purpose**

Confirms the app runs under PM2 rather than as a bare background process.

---

### Step 4: Configure the Application to Survive a Reboot

```bash
pm2 startup
```

`pm2 startup` does not configure anything itself - it **prints** a command you must copy and run, like:

```
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user
```

Run that command, then save the process list and verify:

```bash
pm2 save
systemctl is-enabled pm2-ec2-user   # should print: enabled
```

**Expected outcome:** `enabled`. If it says `disabled` or errors, you skipped the printed `sudo env PATH=...` command.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/02-pm2-startup-enabled.png
```

**Capture**

`systemctl is-enabled pm2-ec2-user` returning `enabled`.

**Purpose**

Proves the PM2 startup service is registered with systemd - what makes the app survive a reboot.

---

### Step 5: Configure Nginx

Put Nginx in front of the app so public traffic on port 80 is proxied to the app on 8080. Your block must be the `default_server`, and the stock one in `nginx.conf` must be removed, or `nginx -t` fails with "duplicate default server".

```bash
sudo dnf install -y nginx

sudo tee /etc/nginx/conf.d/app.conf <<'EOF'
server {
    listen 80 default_server;
    server_name _;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }
}
EOF

sudo sed -i '/listen       80 default_server;/d' /etc/nginx/nginx.conf   # remove stock default
sudo nginx -t                  # validate config
sudo systemctl enable --now nginx   # start now and on boot
```

**Expected outcome:** `nginx -t` passes.

---

### Step 6: Test the Full Chain

```bash
curl localhost/health              # on the instance, through Nginx
curl http://YOUR_PUBLIC_IP/health  # from your laptop
```

**Expected outcome:** Both return `{"status":"healthy",...}`, and `curl http://YOUR_PUBLIC_IP/` shows `"environment":"production"`.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/03-health-endpoint.png
```

**Capture**

The `/health` response at `http://YOUR_PUBLIC_IP/health` (browser or `curl`), showing `{"status":"healthy",...}`.

**Purpose**

Shows the health endpoint is reachable through Nginx from outside the instance.

---

### Step 7: Verify Automatic Restart After a Crash

```bash
pm2 list                    # note the restart count
kill -9 $(pm2 pid myapp)
sleep 2
pm2 list                    # restart count increased; status back to online
```

**Expected outcome:** PM2 restarts the process automatically.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/05-crash-recovery.png
```

**Capture**

`pm2 list` after the `kill -9`, showing the incremented restart count (`↺`) and `online` status.

**Purpose**

Proves PM2 restarts the app after a crash.

---

### Step 8: Verify the Application Survives a Reboot

```bash
sudo reboot
```

Wait ~60 seconds, reconnect, then - without starting anything:

```bash
pm2 list              # myapp should be online
curl localhost/health # should respond
```

**Expected outcome:** The app is running with no manual start. If not, the printed `pm2 startup` command from Step 4 was never run.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/04-after-reboot.png
```

**Capture**

`pm2 list` after reconnecting from the reboot, showing `myapp` online.

**Purpose**

Proves the app returned automatically after a reboot - the lab's core success criterion.

---

## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-deploy-nodejs` containing:

1. **Application code** - `app.js`, `package.json`
2. **PM2 configuration** - `ecosystem.config.js`
3. **Nginx configuration** - `app.conf`
4. **Deployment script** - `deploy.sh` capturing the commands you ran
5. **Screenshots** (in `screenshots/`):
   - `01-pm2-list-online.png` - app online under PM2
   - `02-pm2-startup-enabled.png` - startup service enabled
   - `03-health-endpoint.png` - `/health` response
   - `04-after-reboot.png` - app online after reboot
   - `05-crash-recovery.png` - restart count after `kill -9`
6. **README.md**:
   - Deployment process
   - Why PM2 instead of `node app.js &`
   - How you proved the app survives a reboot

**Structure:**
```
ce-lab-deploy-nodejs/
├── README.md
├── deploy.sh
├── application/
│   ├── app.js
│   ├── package.json
│   └── ecosystem.config.js
├── configs/
│   └── app.conf
└── screenshots/
    ├── 01-pm2-list-online.png
    ├── 02-pm2-startup-enabled.png
    ├── 03-health-endpoint.png
    ├── 04-after-reboot.png
    └── 05-crash-recovery.png
```

---

## Screenshot Checklist

Before submitting, verify these are present in the `screenshots/` folder:

- [ ] `screenshots/01-pm2-list-online.png`
- [ ] `screenshots/02-pm2-startup-enabled.png`
- [ ] `screenshots/03-health-endpoint.png`
- [ ] `screenshots/04-after-reboot.png`
- [ ] `screenshots/05-crash-recovery.png`

---

## Grading Rubric

| Criteria | Points |
|----------|--------|
| **Application deployed and publicly accessible** | 30 |
| **PM2 configured** (incl. startup persistence verified) | 20 |
| **Nginx proxy working** | 20 |
| **Health check endpoint** | 15 |
| **Documentation** (incl. reboot proof) | 15 |
| **Total** | **100** |

---

**Great work deploying a production-ready application!** 🚀
