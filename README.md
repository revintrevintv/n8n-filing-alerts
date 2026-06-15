# n8n-filing-alerts

An n8n workflow that polls the SEC EDGAR full-text search API every 6 hours, filters out SPACs and blank-check shells, and sends a clean Telegram digest of real S-1 filings.

## What it does

| Step | Node | What happens |
|------|------|--------------|
| 1 | **Every 6 Hours** | Schedule trigger fires |
| 2 | **Build Date Range** | Computes yesterday and today as ISO date strings |
| 3 | **EDGAR EFTS Search** | GET `efts.sec.gov/LATEST/search-index?forms=S-1&dateRange=custom&…` |
| 4 | **Filter & Format** | Drops SPACs/shells by name pattern; formats HTML Telegram message |
| 5 | **Any Interesting?** | IF gate — skips Telegram send when no results |
| 6 | **Send Telegram Alert** | Posts to your bot/chat via Telegram node |

Filtering excludes companies whose name matches `/acquisition/i`, `/blank.check/i`, `/spac/i`, or `/merger/i`. Everything else passes.

## Verified execution (2026-06-15)

The workflow ran end-to-end through n8n on 2026-06-15, fired by n8n's own schedule trigger (execution mode: `trigger`, id: 2). All 6 active nodes succeeded in 1.8 s and the Telegram node confirmed delivery. See [`n8n-execution-log.html`](n8n-execution-log.html) for the full node-by-node trace extracted from n8n's execution database.

**Node execution trace** (from n8n execution id=2, 2026-06-15T20:15:48Z):

| Node | Time | Result |
|------|------|--------|
| Every 6 Hours | 1 ms | fired |
| Build Date Range | 10 ms | `{startdt: "2026-06-14", enddt: "2026-06-15"}` |
| EDGAR EFTS Search | 1535 ms | HTTP 200, 6 S-1 filings returned |
| Filter & Format | 14 ms | hasFilings=true, 4 of 6 passed filter |
| Any Interesting? | 2 ms | routed to Telegram branch |
| Send Telegram Alert | 234 ms | Telegram ok=true, message_id=1130 |

**Message delivered** (copied verbatim from n8n's `Send Telegram Alert` node output):

```
📋 New S-1 Filings — last 24h
4 companies registered to go public:

1. Pattern Group Inc. (PTRN) — Lehi, UT
   S-1 · 2026-06-15
   View on EDGAR →

2. AMERICAN BATTERY MATERIALS, INC. (BLTH) — Greenwich, CT
   S-1/A · 2026-06-15
   View on EDGAR →

3. Decoy Therapeutics Inc. (DCOY) — Houston, TX
   S-1 · 2026-06-15
   View on EDGAR →

4. DPC Holdings Ltd (DPC) — Derby
   S-1/A · 2026-06-15
   View on EDGAR →

2 SPAC/shell filing(s) filtered out.
```

Filtered out: Bridge III Acquisition Ltd (BDDD) and Southern Cross Acquisition I Corp — both matched `/acquisition/i`.

## Requirements

- [n8n](https://n8n.io/) self-hosted (v2.x) — installed via `npm install -g n8n`
- A Telegram bot token and chat ID

No other dependencies. Uses SEC EDGAR public API — no API key required.

## Setup

### 1. Install n8n

```bash
npm install -g n8n
```

Configure and start n8n. To run it bound to localhost only (recommended for a VPS):

```bash
export N8N_LISTEN_ADDRESS=127.0.0.1
export N8N_PORT=5678
export N8N_SECURE_COOKIE=false
n8n start
```

Visit `http://localhost:5678` and complete the setup wizard.

### 2. Create a Telegram credential

In n8n: **Settings → Credentials → New → Telegram API**. Paste your bot token.

### 3. Import the workflow

```bash
n8n import:workflow --input=workflow.json --activeState=fromJson
```

Or: open n8n UI → **Workflows → Import from File** → select `workflow.json`.

### 4. Wire up the credential

Open the imported workflow, click the **Send Telegram Alert** node, and select your Telegram credential. Set the **Chat ID** to your target chat.

### 5. Activate

Toggle the workflow to **Active** in the n8n UI. First alert fires within 6 hours (or trigger manually from the UI).

## Systemd service (optional — keep running on reboot)

```ini
# ~/.config/systemd/user/n8n.service
[Unit]
Description=n8n Workflow Automation
After=network.target

[Service]
Type=simple
Environment=N8N_PORT=5678
Environment=N8N_LISTEN_ADDRESS=127.0.0.1
Environment=N8N_SECURE_COOKIE=false
ExecStart=/path/to/n8n start
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now n8n
```

## Customizing the filter

Edit the **Filter & Format** code node. The `BORING` array holds name-pattern regexes. Add or remove patterns to tune which companies pass through.

To also filter by SIC code (industry), add a second **HTTP Request** node after `EDGAR EFTS Search` that fetches `https://data.sec.gov/submissions/CIK{padded}.json` and extracts `sic` — then add a SIC condition to the IF node.

## SEC access policy

EDGAR requires a descriptive `User-Agent` header identifying you. The workflow sends:

```
User-Agent: n8n-filing-alerts your@email.com
```

Edit the **EDGAR EFTS Search** node header to use your own contact email. SEC rate limit is 10 req/s; this workflow runs once per trigger (1 request) so load is negligible.

## Related

- **[edgar-ipo-tracker](https://github.com/revintrevintv/edgar-ipo-tracker)** — the Python scraper that pulls S-1 filings in bulk with SIC enrichment and filing-index parsing. This n8n workflow uses the same EDGAR API for lightweight, real-time alerting.
