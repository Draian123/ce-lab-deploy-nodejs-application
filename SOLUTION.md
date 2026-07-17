# Deploy Node.js Application Lab - Solution

**Student Name:** [Your Name]
**Date Completed:** [Date]

---

# Environment Details

| Item | Value |
|------|-------|
| Instance ID | [i-xxxxxxxxxxxxx] |
| Region | [us-east-2] |
| Public IP | [xx.xx.xx.xx] |
| Node.js Version | [v18.x.x] |
| App Name (PM2) | [myapp] |
| App Port | [8080] |

---

# Step 1: Install Node.js

- [ ] Installed Node.js with `sudo dnf install -y nodejs`
- [ ] `node -v` prints a version number

**My Node.js version:** `_______________________`

---

# Step 2: Create the Application

- [ ] Created `~/app` and ran `npm init -y`
- [ ] Installed `express`
- [ ] Created `app.js` with `/` and `/health` routes

---

# Step 3: Install and Start PM2

## Screenshot 1 – PM2 List (Online)

```
screenshots/01-pm2-list-online.png
```

![PM2 List Online](screenshots/01-pm2-list-online.png)

---

- [ ] Installed PM2 globally with `sudo npm install -g pm2`
- [ ] Created `ecosystem.config.js` setting `NODE_ENV=production` and `PORT=8080`
- [ ] Started the app with `pm2 start ecosystem.config.js`
- [ ] `pm2 list` shows `status: online`, restart count 0
- [ ] `curl localhost:8080/health` returns `{"status":"healthy",...}`
- [ ] `curl localhost:8080/` shows `"environment":"production"`

---

# Step 4: Configure the Application to Survive a Reboot

## Screenshot 2 – PM2 Startup Enabled

```
screenshots/02-pm2-startup-enabled.png
```

![PM2 Startup Enabled](screenshots/02-pm2-startup-enabled.png)

---

- [ ] Ran `pm2 startup` and copied the printed `sudo env PATH=...` command
- [ ] Ran that printed command
- [ ] Ran `pm2 save` to persist the process list
- [ ] `systemctl is-enabled pm2-ec2-user` prints `enabled`

---

# Step 5: Configure Nginx

- [ ] Installed Nginx with `sudo dnf install -y nginx`
- [ ] Created `/etc/nginx/conf.d/app.conf` proxying port 80 → 8080, marked `default_server`
- [ ] Removed the stock `listen 80 default_server;` line from `nginx.conf`
- [ ] `sudo nginx -t` passes
- [ ] Enabled and started Nginx with `sudo systemctl enable --now nginx`

---

# Step 6: Test the Full Chain

## Screenshot 3 – Health Endpoint

```
screenshots/03-health-endpoint.png
```

![Health Endpoint](screenshots/03-health-endpoint.png)

---

- [ ] `curl localhost/health` on the instance returns `{"status":"healthy",...}`
- [ ] `curl http://YOUR_PUBLIC_IP/health` from my laptop returns the same
- [ ] `curl http://YOUR_PUBLIC_IP/` shows `"environment":"production"`

---

# Step 7: Verify Automatic Restart After a Crash

## Screenshot 5 – Crash Recovery

```
screenshots/05-crash-recovery.png
```

![Crash Recovery](screenshots/05-crash-recovery.png)

---

- [ ] Noted the restart count in `pm2 list` before the crash
- [ ] Ran `kill -9 $(pm2 pid myapp)`
- [ ] `pm2 list` shows the restart count (`↺`) incremented and status back to `online`

---

# Step 8: Verify the Application Survives a Reboot

## Screenshot 4 – After Reboot

```
screenshots/04-after-reboot.png
```

![After Reboot](screenshots/04-after-reboot.png)

---

- [ ] Ran `sudo reboot`
- [ ] Waited ~60 seconds and reconnected
- [ ] Without starting anything manually, `pm2 list` shows `myapp` online
- [ ] `curl localhost/health` responds successfully

---

# Submission Checklist

Repository name: `ce-lab-deploy-nodejs` (**public**)

- [ ] `application/` files committed (`app.js`, `package.json`, `ecosystem.config.js`)
- [ ] `configs/app.conf` committed
- [ ] `deploy.sh` committed, capturing the commands run
- [ ] All 5 screenshots present
- [ ] `README.md` complete with deployment process, why PM2 over `node app.js &`, and reboot proof
- [ ] App online under PM2 with 0 unexpected restarts
- [ ] PM2 startup service enabled (`systemctl is-enabled pm2-ec2-user`)
- [ ] Nginx reverse proxy working (port 80 → 8080)
- [ ] `/health` endpoint reachable from outside the instance
- [ ] Crash recovery demonstrated (`kill -9` → auto-restart)
- [ ] Reboot survival demonstrated
- [ ] Repository URL submitted
