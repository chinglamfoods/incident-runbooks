# Incident: OOM Crash + Tailscale Inbound Flood

**Date:** 2026-02-27
**Machine:** AWS EC2 (aarch64, 3.7GB RAM, 2 CPU)
**Symptoms:** CPU 68%, 157M network in, 1.55M network out, 10K packets, machine crashed and rebooted

---

## The Problem

The machine kept crashing and rebooting. The user observed high CPU, massively asymmetric network traffic (100:1 inbound vs outbound ratio), and the machine becoming unresponsive.

**Root cause:** Two issues compounding each other:

1. **OOM (Out of Memory)** — The `claude` CLI process was consuming ~3GB RSS on a 3.7GB machine with zero swap. The kernel OOM killer repeatedly killed it, causing cascading failures.
2. **Tailscale Funnel misconfiguration** — A Funnel route was proxying to `localhost:8788` where nothing was listening, causing Tailscale ingress nodes to flood the machine with retried connections that all failed.

---

## How to Investigate (Step by Step)

### 1. Check boot history — was there actually a crash?

```bash
last -x reboot shutdown | head -10
journalctl --list-boots
```

**What to look for:** Short-lived boots (minutes), unexpected reboots, gaps between shutdown and reboot.

In this case: boot -1 lasted only 3 minutes — clear sign of instability.

### 2. Check for OOM kills

```bash
journalctl -b -2 | grep -iE 'oom|out of memory|killed process'
```

**What to look for:** `Out of memory: Killed process <name>` lines. Note the process name and `anon-rss` size — that tells you who ate the memory.

In this case: `claude` was killed 3 times across 2 days, each time using ~3GB.

### 3. Check historical resource usage (sar/sysstat)

```bash
# CPU over time
sudo sar -u -f /var/log/sysstat/sa<DD>

# Memory over time
sudo sar -r -f /var/log/sysstat/sa<DD>

# Network over time
sudo sar -n DEV -f /var/log/sysstat/sa<DD>
```

**What to look for:** Spikes in `%system` + `%iowait` (kernel doing OOM work), sudden memory drops, asymmetric network rx/tx.

In this case: At 02:10, `%system=23%, %iowait=12.3%` and network showed 380kB/s in vs 17kB/s out.

### 4. Check what's causing network asymmetry

```bash
# Current connections
ss -tunapo | head -50

# Tailscale status
sudo tailscale status
sudo tailscale funnel status
sudo tailscale serve status

# Check journal for repeated errors
journalctl -b -2 | grep -c 'proxy error'
```

**What to look for:** Tailscale "proxy error: connection refused" spam means a Funnel/Serve route is pointing to a dead backend. Tailscale ingress nodes will keep retrying, creating massive inbound traffic with almost no outbound response.

### 5. Check for cascading failures

```bash
journalctl -b -2 | grep -iE 'watchdog timeout|SIGABRT|failed with result'
```

**What to look for:** Services timing out because the system was too busy handling OOM. In this case, `snapd` hit its watchdog timeout and crashed, contributing to system instability.

---

## Pattern Recognition

When you see these symptoms together, think **OOM + network amplifier**:

| Symptom | Points to |
|---------|-----------|
| High CPU with high `%system` + `%iowait` | Kernel OOM killer / page reclaim |
| Asymmetric network (high in, low out) | Something accepting connections but failing to respond (dead backend) |
| Short-lived boots | System too unstable to stay up |
| `proxy error: connection refused` spam | Tailscale Funnel/Serve pointing to dead service |
| `Watchdog timeout` on unrelated services | Cascading failure from resource starvation |

---

## Fixes Applied

### 1. Add swap (immediate relief)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Why:** Zero swap on a memory-tight machine means the OOM killer fires immediately. Swap gives the system breathing room.

### 2. Remove dead Tailscale Funnel

```bash
sudo tailscale funnel off
```

**Why:** The Funnel was proxying `/gmail-pubsub` to `localhost:8788` where nothing was running. Tailscale ingress nodes kept flooding the machine with retries. Removing it stops the inbound traffic storm.

### 3. (Deferred) Upgrade instance RAM

The real fix is more RAM. `claude` CLI regularly uses 2.9-3.1GB RSS. A 3.7GB machine is too tight. Recommend 8GB+ (e.g., `t4g.large`).

### 4. (Deferred) Evaluate snapd

`snapd` watchdog timeout caused a secondary crash. Consider installing `amazon-ssm-agent` via apt and removing snapd, or disabling its watchdog.

---

## Key Takeaway

**Don't chase the network numbers first.** The asymmetric traffic looked like an attack but was actually a misconfigured Tailscale Funnel hitting a dead backend. The real root cause was memory — always check OOM kills early in your investigation. The network issue was an amplifier, not the cause.
