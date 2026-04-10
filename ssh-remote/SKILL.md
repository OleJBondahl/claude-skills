---
name: ssh-remote
description: Use when running commands on remote machines or WSL from Windows. Covers SSH via WSL ControlMaster, file transfer, and remote execution. Trigger on "remote", "SSH", "WSL command", "run on server", or any task requiring execution on another machine.
---

# SSH Remote Execution

## Overview

On Windows, SSH from Git Bash cannot handle interactive password prompts. All remote commands must go through WSL, which has a ControlMaster socket for password-free SSH reuse.

**NEVER** run `ssh` directly from Git Bash — it triggers an interactive password prompt that blocks the agent.

## Reliable Command Patterns

### What works

| Target | Pattern |
|--------|---------|
| WSL local | `wsl bash -c "<command>"` |
| Remote host | `wsl bash -c "ssh <host> '<command>'"` |
| File to remote | `wsl bash -c "scp /mnt/c/... <host>:/path/"` |
| Script on remote | Write to file, scp, then `bash /tmp/script.sh` |

### What does NOT work

| Pattern | Problem |
|---------|---------|
| `ssh` from Git Bash directly | Blocks on interactive password prompt |
| `wsl -- ssh <host> "command"` | Inconsistent — Git Bash mangles paths and quotes |
| `wsl -- scp /mnt/c/...` | SCP misinterprets the colon in `/mnt/c` as a hostname separator |
| `sudo` via SSH without cached credentials | Blocks on TTY requirement — see "Sudo on Remote Hosts" |

**Key rule:** Always use `wsl bash -c "..."` with single quotes inside for SSH commands. This avoids Git Bash path mangling.

## Script Pattern (recommended for complex commands)

For anything beyond a simple one-liner, write a script and execute it remotely:

```bash
# 1. Write script locally (to claude-tools/ or /tmp)
# 2. Copy to remote
wsl bash -c "scp /mnt/c/<windows-path>/script.sh <host>:/tmp/"
# 3. Execute
wsl bash -c "ssh <host> 'bash /tmp/script.sh'"
```

This avoids all quoting, line-wrapping, and escaping issues.

## File Transfer

Windows paths map to `/mnt/c/...` inside WSL.

```bash
# Windows → Remote
wsl bash -c "scp /mnt/c/Users/<user>/path/to/file <host>:/destination/"

# Remote → Windows
wsl bash -c "scp <host>:/remote/path /mnt/c/Users/<user>/destination/"

# Directory sync
wsl bash -c "scp -r /mnt/c/.../dir/ <host>:/destination/"
```

## Background Services on Remote

```bash
# Start with nohup, capture PID and logs
nohup <command> > /tmp/<service>.log 2>&1 &
echo $! > /tmp/<service>.pid

# Stop
kill $(cat /tmp/<service>.pid)
# or by process name
pkill -f "<process-pattern>"

# Check logs
tail -50 /tmp/<service>.log
```

## Sudo on Remote Hosts

Agents cannot enter passwords, so `sudo` blocks without preparation. Use **global timestamp caching** — the user authenticates once, and agents reuse the cached credential for a time window (like SSH ControlMaster for sudo).

### Setup (one-time, run on the remote host)

```bash
echo "Defaults:<user> timestamp_type=global, timestamp_timeout=240" | sudo tee /etc/sudoers.d/<user>-cached
sudo chmod 440 /etc/sudoers.d/<user>-cached
```

- `timestamp_type=global` — shares sudo cache across all TTYs/sessions for that user (not per-terminal)
- `timestamp_timeout=240` — cache lasts 4 hours (matches SSH ControlPersist)

### Activating the sudo cache

The user must SSH in and run `sudo -v` (enter password once). This primes the cache for 4 hours. When it expires, the user runs `sudo -v` again.

### Agent usage

Once the cache is primed, agents can use sudo normally:

```bash
wsl bash -c "ssh <host> 'sudo systemctl restart myservice'"
```

### If sudo fails with "a terminal is required"

The cache has expired. Tell the user:

> The sudo cache on `<host>` has expired. Please SSH in and run `sudo -v` to refresh it (lasts 4 hours). I'll retry after.

Wait for user confirmation before retrying.

## Connection Failure

If a command fails with connection errors:

1. **Do NOT retry repeatedly** — the socket has likely expired
2. **Tell the user** to re-authenticate:

> The SSH socket has expired. Please run `ssh <host>` in a WSL terminal, enter your password, then exit. I'll retry after.

3. Wait for user confirmation before retrying

## Setting Up a New Remote Host

1. User connects from WSL once to accept the host key
2. Add entry to WSL `~/.ssh/config`:

```bash
wsl bash -c "cat >> ~/.ssh/config << 'EOF'

Host <alias>
    HostName <ip>
    User <user>
    ControlMaster auto
    ControlPath /tmp/ssh-%h
    ControlPersist 4h
EOF"
```

3. User authenticates: `ssh <alias>` from WSL, enters password, then exits
4. Verify from agent: `wsl bash -c "ssh <alias> 'hostname'"`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `wsl --` instead of `wsl bash -c` | `wsl bash -c "ssh <host> '...'"` is reliable |
| SCP with `/mnt/c` via `wsl --` | Wrap in `wsl bash -c "scp ..."` |
| Long complex commands via SSH | Write a script, scp it, execute it |
| sudo without cached credentials | User must SSH in and run `sudo -v` first — see "Sudo on Remote Hosts" |
| Retrying after socket expiry | Ask user to re-authenticate |
