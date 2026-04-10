---
name: deploy-remote
description: Use when deploying any application to a remote host via SSH. Covers build-transfer-restart-verify pipeline, script-based deployment, service lifecycle, health checks, and rollback. Trigger on "deploy", "push to server", "restart service", "health check remote", "release to".
---

# Remote Deployment

## Overview

General-purpose deployment pipeline for any application to a remote host via SSH. Builds locally on Windows, transfers artifacts via SCP through WSL, and manages services on the remote.

**Prerequisite:** The `ssh-remote` skill must be working (ControlMaster socket active for the target host).

## Deployment Pipeline

```
1. Build (local)       — compile/bundle the application
2. Transfer (SCP)      — send artifacts to remote host
3. Stop (remote)       — stop old services gracefully
4. Start (remote)      — start new services with logging
5. Verify (remote)     — health checks and log inspection
```

## Before You Deploy

Ask the user (or check project docs) for:

| Question | Example |
|----------|---------|
| **SSH host alias** | `nirvana`, `prod-server` |
| **Build command** | `npx tsc --build`, `npm run build`, `cargo build --release` |
| **Post-build steps** | Copy non-compiled assets, generate configs |
| **Remote app directory** | `/opt/app`, `/home/user/myapp` |
| **What to transfer** | `dist/`, `build/`, entire project |
| **How to start** | `node server.js`, `docker compose up -d`, `systemctl start` |
| **Health check endpoint** | `http://localhost:PORT/health` |
| **Services needing sudo** | Reverse proxy, systemd units |

## Script-Based Deployment (Recommended)

Write a deploy script to `claude-tools/`, SCP it, execute it. This avoids quoting issues and makes deployments repeatable.

```bash
#!/bin/bash
set -e

# === Configuration (adapt per project) ===
APP_DIR="$1"            # passed as argument
SERVICE_NAME="$2"
PORT="$3"
START_CMD="$4"

echo "=== Stopping old service ==="
pkill -f "$SERVICE_NAME" || true
sleep 1

echo "=== Starting service ==="
cd "$APP_DIR"
nohup $START_CMD > /tmp/${SERVICE_NAME}.log 2>&1 &
echo $! > /tmp/${SERVICE_NAME}.pid
echo "PID: $(cat /tmp/${SERVICE_NAME}.pid)"

echo "=== Waiting for startup ==="
sleep 3

echo "=== Health Check ==="
if curl -sf http://localhost:${PORT}/health > /dev/null 2>&1; then
    echo "HEALTHY"
else
    echo "UNHEALTHY — check logs:"
    tail -20 /tmp/${SERVICE_NAME}.log
    exit 1
fi
```

Execute:
```bash
wsl bash -c "scp /mnt/c/<project>/claude-tools/deploy.sh <host>:/tmp/"
wsl bash -c "ssh <host> 'bash /tmp/deploy.sh /path/to/app myservice 4000 \"node dist/server.js\"'"
```

## Phase-by-Phase Reference

### Build (local)
```bash
# Run the project's build command
<build-command>

# Copy any non-compiled assets if needed (e.g., proto files, static assets)
cp -r src/assets dist/assets
```

### Transfer
```bash
# Full project sync
wsl bash -c "scp -r /mnt/c/<windows-path>/ <host>:/tmp/<app>/"

# Incremental (only dist)
wsl bash -c "scp -r /mnt/c/<windows-path>/dist/ <host>:<app-dir>/dist/"
```

### Stop
```bash
# By process pattern
pkill -f "<process-pattern>" || true

# By PID file
kill $(cat /tmp/<service>.pid) 2>/dev/null || true

# Docker
docker compose -f <compose-file> down

# Systemd
sudo -n systemctl stop <service>
```

`|| true` prevents errors when the process isn't running. `sudo -n` = non-interactive (requires NOPASSWD).

### Start
```bash
# Direct process with logging
nohup <start-command> > /tmp/<service>.log 2>&1 &
echo $! > /tmp/<service>.pid

# Docker
docker compose -f <compose-file> up -d

# Systemd
sudo -n systemctl start <service>
```

### Verify
```bash
# Health endpoint
curl -sf http://localhost:<port>/health && echo "OK" || echo "FAILED"

# Process alive check
kill -0 $(cat /tmp/<service>.pid) 2>/dev/null && echo "PID alive" || echo "PID dead"

# Port listening
ss -tlnp | grep :<port>

# Recent logs
tail -20 /tmp/<service>.log
```

## Rollback

If deployment fails:
1. **Inspect:** `tail -50 /tmp/<service>.log`
2. **Stop:** `pkill -f "<process-pattern>"`
3. **Restore:** use previous backup or `git checkout` on remote
4. **Restart:** re-run the start command with the old version

Keep a backup before deploying: `cp -r <app-dir> <app-dir>.bak`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Deploying without building first | Always run the build command locally before transfer |
| Non-compiled assets missing | Check if build tool copies everything (proto files, templates, etc.) |
| `sudo` blocks agent | Set up NOPASSWD for required commands (see `ssh-remote` skill) |
| Service dies silently after start | Always check log files and PID after starting |
| Old process still bound to port | `pkill` before starting, add `sleep 1` between stop and start |
| Transferring too much (node_modules) | Only transfer build output, install dependencies on remote |
| No health check defined | Add one — without it you can't verify deployment succeeded |
