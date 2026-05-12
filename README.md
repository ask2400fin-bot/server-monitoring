# server-monitoring

External-vantage port scan watchdog for Andrei's infrastructure. Public repo so GitHub Actions get **unlimited free minutes** (private repos cap at 2000/mo).

## What it does

Every 30 minutes, a GHA workflow runs `nmap -sT -Pn --open -p-` from a GitHub-hosted runner against:

- **zam-server** (Hetzner Helsinki, `89.167.36.90`) — expected `22/tcp, 80/tcp`
- **skandok-vps** (Timeweb Moscow, `72.56.37.86`) — expected `22/tcp, 80/tcp, 443/tcp`

If the observed port set differs from the baseline, the workflow sends a Telegram alert to the security-alerts bot. This catches:

- New container with unauthorised public port (defense-in-depth against `0.0.0.0:DB_PORT` leaks)
- Backdoor listener on arbitrary high port (e.g. `:41337`)
- Loss of expected service (port 80 disappears = nginx down)

Independent of host-side detection (`skandok-monitor` on zam-server), so a compromised host can't suppress the alert.

## Why public

Public GitHub repos get **unlimited free GHA minutes**. Two scans × full port range × every 30 min = ~960 min/month — fits the public quota and costs $0.

What's public:
- Workflow definition (this file)
- Workflow run logs — show only "scan diff: yes/no" + diff details

What's NOT public:
- `SECURITY_BOT_TOKEN` — GitHub repo secret, never logged
- `TG_CHAT_ID` — GitHub repo secret, never logged

Scanning your own IPs isn't sensitive — any opportunistic scanner sees the same surface.

## Setup (one-time, after repo creation)

1. Create dedicated Telegram bot via `@BotFather` (recommended: `@vw_security_alerts_bot`) to isolate blast radius from the `claude-bridge` bot.
2. Add repo secrets:
   ```
   gh secret set SECURITY_BOT_TOKEN -R ask2400fin-bot/server-monitoring --body '<bot token from BotFather>'
   gh secret set TG_CHAT_ID         -R ask2400fin-bot/server-monitoring --body '<andrei chat id>'
   ```
3. Trigger a manual test:
   ```
   gh workflow run external-portscan.yml -R ask2400fin-bot/server-monitoring
   gh run watch -R ask2400fin-bot/server-monitoring
   ```

Until both secrets are set, scans still run but alerts emit a workflow error (visible in GHA UI) instead of a Telegram message.

## Baseline maintenance

Expected port sets are hardcoded in `.github/workflows/external-portscan.yml` under `ZAM_EXPECTED` / `SKD_EXPECTED`. Update both whenever a server's authorised surface changes (e.g. adding `443/tcp` to zam-server after origin TLS migration).
