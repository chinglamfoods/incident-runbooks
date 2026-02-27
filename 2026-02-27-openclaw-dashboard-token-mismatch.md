# Incident: OpenClaw Dashboard "Unauthorized: Gateway Token Mismatch"

**Date:** 2026-02-27
**Service:** OpenClaw Control UI
**Symptom:** Dashboard shows `unauthorized: gateway token mismatch (open the dashboard URL and paste the token in Control UI settings)`

---

## The Problem

After a gateway restart (e.g., reboot, crash, service restart), the OpenClaw gateway may generate or load a token that differs from the one cached in the browser's local storage. The Control UI dashboard refuses to connect because the tokens don't match.

This commonly happens after:
- Machine crash/reboot (like the OOM incident)
- Manual gateway restart
- Config changes that trigger a token regeneration

---

## How to Investigate

### 1. Confirm the error in gateway logs

```bash
openclaw logs --limit 10 | grep -i 'token_mismatch'
```

You'll see lines like:
```
unauthorized conn=... reason=token_mismatch
```

### 2. Get the current gateway token

```bash
grep 'token' ~/.openclaw/openclaw.json
```

Look for the `gateway` section's `token` field (not the channel bot tokens).

---

## How to Fix

### The dashboard settings page won't help — it can't load without auth.

Pass the token directly via URL query parameter:

```
http://localhost:18789/?token=<GATEWAY_TOKEN>
```

This authenticates the session and saves the token to the browser's local storage. Future visits to `http://localhost:18789` will work without the query parameter.

---

## Key Takeaway

When the Control UI says "paste the token in settings" but the settings page itself won't load — bypass it by appending `?token=<value>` to the dashboard URL. The token is in `~/.openclaw/openclaw.json` under the gateway section.
