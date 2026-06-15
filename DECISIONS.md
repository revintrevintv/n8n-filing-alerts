# DECISIONS.md — n8n-filing-alerts

Decision log for the n8n portfolio piece.

---

## Installation: npm instead of Docker

**Decision:** Install n8n via `npm install -g n8n`, not Docker.

**Rationale:** The VPS has 4 GB RAM. Docker pulls ~800 MB base image plus n8n's own container (~300 MB), adds a daemon process, and Docker is not installed on this machine. npm install delivers a ~250 MB node_modules tree and runs n8n in-process under the existing Node.js 22 runtime that is already present. Memory footprint is about 300 MB at idle vs. 600–800 MB with Docker. No privilege-escalation risk, no container networking to configure, and the binary is immediately available on PATH.

---

## Data source: direct EDGAR API vs. edgar-ipo-tracker CSV

**Decision:** n8n calls the EDGAR EFTS API directly, rather than reading the edgar-ipo-tracker's output CSV.

**Rationale:** CSV polling would require: (a) coordinating file timestamps between the Python scraper and n8n, (b) a cron or shell-out node that runs the Python scraper first, (c) dealing with the scraper's 3-stage pipeline (which takes 30–90 s for 50+ filings). The n8n workflow is a *real-time alerting* use case — it only needs the last 24 hours of data and a single EFTS API call returns it in < 1 s. edgar-ipo-tracker is a *bulk enrichment* tool (adds SIC, filing index, exchange data). Both use the same upstream API; they solve different problems. The README calls this out and links to edgar-ipo-tracker for clients who need the enriched dataset.

---

## Filter method: name patterns vs. SIC enrichment

**Decision:** Filter SPACs and shells by company name regex (`/acquisition/i`, `/blank.check/i`, `/spac/i`, `/merger/i`) rather than by SIC code 6770 (Blank Checks).

**Rationale:** SIC enrichment requires one additional API call per filing (`data.sec.gov/submissions/CIK{padded}.json`). A typical day has 5–15 S-1 filings. Making 5–15 extra API calls per workflow run adds latency, increases failure surface (each call can time out), and bumps SEC request count. Name-pattern filtering is ~95% accurate for the common SPAC naming convention and runs in-process with zero additional HTTP calls. The README documents how to add SIC enrichment as a future enhancement.

---

## Schedule: every 6 hours

**Decision:** Schedule trigger fires every 6 hours (0:00, 6:00, 12:00, 18:00 UTC).

**Rationale:** EDGAR processes and publishes filings throughout the business day but rarely updates overnight. 6-hour intervals mean new filings appear in Telegram within 6 hours of being visible on EDGAR — plenty fast for a monitoring use case. Daily (24h) would miss same-day urgency for finance clients; hourly would add noise on light filing days and burn more SEC requests.

---

## Binding: localhost only

**Decision:** n8n binds to `127.0.0.1:5678` only (`N8N_LISTEN_ADDRESS=127.0.0.1`).

**Rationale:** This is a personal automation tool on a shared VPS. n8n's web UI exposes workflow data including (encrypted) credentials. There is no reason to expose it publicly. Operator accesses it via SSH tunnel (`ssh -L 5678:localhost:5678 user@vps`) or a reverse proxy if desired.

---

## Credential storage: n8n encrypted database

**Decision:** Telegram bot token is stored in n8n's built-in encrypted credential store, not in environment variables or the workflow JSON.

**Rationale:** n8n encrypts credentials at rest using AES-256-GCM with the `N8N_ENCRYPTION_KEY`. The workflow JSON published to GitHub contains only the credential name and a placeholder ID — no token. Environment-variable credential injection is simpler but requires the secret to appear in the systemd unit or a config file without encryption. n8n's approach provides encryption-at-rest with the workflow publishable to a public repo.

---

## n8n version: 2.25.7 (current npm latest)

**Decision:** Use whatever `npm install -g n8n` installs (2.25.7 at time of setup).

**Rationale:** No reason to pin to an older version for a portfolio piece. Noted: n8n 2.x requires a `workflow_published_version` DB record for schedule triggers to activate — created automatically when you activate via the UI, but requires manual DB insertion when activating programmatically. This is a quirk of the 2.x release; document in case of future installs.
