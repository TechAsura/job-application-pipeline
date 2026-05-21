# Automated Job Application Pipeline

An end-to-end job hunting automation system that monitors Gmail for job alerts, parses listings, checks for duplicates, generates tailored resume bullets and cover letters using the Claude API, and logs everything to a Notion database — automatically, every 15 minutes.

Built and deployed on a self-hosted VPS using n8n.

---

## What It Does

1. **Monitors Gmail** for job alert emails from Indeed and LinkedIn every 15 minutes
2. **Parses** job title, company, and description from the email
3. **Filters** out non-job emails (profile views, newsletters, etc.)
4. **Checks for duplicates** before creating any Notion entries
5. **Logs the job** to a Notion database with source, date, and job description
6. **Calls the Claude API** with resume context to generate 3 tailored resume bullet points and a cover letter paragraph specific to each role
7. **Writes the generated content** back to the Notion row, ready for review and application

---

## Architecture

```
Gmail (Indeed + LinkedIn)
        │
        ▼
Code Node — Parse & Filter
        │
        ▼
Notion — Duplicate Check (Get pages by job_id)
        │
        ▼
IF Node — Skip if duplicate exists
        │ (true = new job)
        ▼
Notion — Create database page
        │
        ▼
HTTP Request — Claude API (resume tailoring)
        │
        ▼
Notion — Update page with tailored_bullets + cover_letter
```

---

## Tech Stack

| Component | Tool |
|---|---|
| Workflow Automation | n8n (self-hosted) |
| Hosting | Hostinger VPS |
| Email Source | Gmail API (OAuth2) |
| Job Sources | Indeed, LinkedIn |
| Database | Notion API |
| AI Tailoring | Anthropic Claude API (claude-sonnet-4-5) |
| Language | JavaScript (n8n Code node) |

---

## Notion Database Schema

| Column | Type | Description |
|---|---|---|
| `company` | Title | Company name |
| `job_id` | Text | Email ID used for deduplication |
| `source` | Select | Indeed or LinkedIn |
| `url` | URL | Job posting link (manual) |
| `jd_text` | Text | Job description snippet |
| `tailored_bullets` | Text | AI-generated resume bullets |
| `cover_letter` | Text | AI-generated cover letter paragraph |
| `status` | Select | New / Applied / Interview / Rejected |
| `applied_date` | Date | Date the alert was received |
| `notes` | Text | Manual notes |

---

## How It Works

### Gmail Trigger
Two Gmail Trigger nodes poll every 15 minutes — one filtered to `donotreply@match.indeed.com` and one to `messages-noreply@linkedin.com`. Both feed into the same Code node.

### Parse & Filter (Code Node)
The Code node:
- Skips emails that don't contain job-related keywords (`hiring`, `job`, `administrator`, `support`, etc.)
- Detects the source (Indeed vs LinkedIn) from the sender
- Parses job title and company — Indeed uses `"Job Title @ Company"` format; LinkedIn uses `"Job Title: Company and are hiring"`
- Cleans LinkedIn snippet noise characters

### Duplicate Check
Before creating any Notion entry, the pipeline queries the Job Applications database for an existing row with the same `job_id`. If one exists, the IF node routes to the false branch and the workflow stops — no duplicate created.

### Claude API Tailoring
The HTTP Request node calls `https://api.anthropic.com/v1/messages` with:
- A system prompt containing Elijah's resume context, skills, and projects
- The parsed job title, company, and description snippet
- Instructions to respond in a strict `BULLETS: / COVER_LETTER:` format

The response is split at the `COVER_LETTER:` marker and written to the appropriate Notion columns.

---

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Google Cloud project with Gmail API enabled
- Notion integration token with access to your job tracking database
- Anthropic API key

### Steps
1. Clone or import the n8n workflow JSON
2. Configure Gmail OAuth2 credential in n8n
3. Add Notion API key as a credential
4. Add Anthropic API key as a header in the HTTP Request node
5. Update the Claude prompt with your own resume context
6. Set your Notion database ID in the Notion nodes
7. Activate the workflow

---

## Example Output

Every new job alert automatically generates a Notion row like this:

**Job:** IT Helpdesk Support @ MyHealth Partners Inc.  
**Source:** Indeed  
**Status:** New  

**Tailored Bullets:**
- Diagnosed and resolved complex networking issues including SMB/NFS mount failures, LXC bind mount permission conflicts, and DNS resolution problems across a two-node Proxmox homelab serving 10+ services
- Configured and maintained critical network infrastructure including AdGuard Home for DNS-over-HTTPS, DHCP services, and VPN solutions (NordVPN, WireGuard, Tailscale)
- Documented all homelab configurations and troubleshooting procedures in version-controlled repositories at github.com/TechAsura

**Cover Letter:**  
My hands-on experience managing a production homelab environment has given me practical troubleshooting skills that translate directly to helpdesk support. I've diagnosed everything from network connectivity issues and DNS misconfigurations to storage mount failures and permission conflicts...

---

## Related Projects

- [Self-Hosted Homelab](https://github.com/TechAsura) — Two-node Proxmox VE cluster running 10+ self-hosted services
- [MapleLog](https://maplelog.co) — B2B SaaS compliance platform for Canadian SMBs

---

## Author

**Elijah Robinson** — [github.com/TechAsura](https://github.com/TechAsura)
