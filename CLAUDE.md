# Lead Profile Enrichment - Development Guide

## Project Overview

n8n automation workflow that enriches new signup leads, scores them against ICP (Ideal Customer Profile), and generates personalized outreach emails.

**Input:** Email + Website URL (via webhook)
**Output:** Enriched lead data, ICP score, personalized emails, Slack notifications, Google Sheets tracking

## Architecture

```text
Webhook → Extract Domain → Parallel Enrichment → Merge Data
                                ↓
                         AI ICP Scoring (Gemini)
                                ↓
                      AI Email Generation (Gemini)
                                ↓
              ┌─────────────────┼─────────────────┐
        Slack Notify     Gmail Draft      Google Sheets
        (High Value)    (High Value)       (All Leads)
```

## Tech Stack

| Component | Service |
|-----------|---------|
| Workflow Engine | n8n (Docker) |
| Database | PostgreSQL 15 |
| AI | Google Gemini 1.5 Flash |
| Person Enrichment | Hunter.io, Apollo.io |
| Company Enrichment | Apollo.io, PDL |
| Tech Detection | HTML scraping (free) |
| Output | Google Sheets, Gmail, Slack |

## File Structure

```text
lead-profile-enrichment-n8n/
├── docker-compose.yml          # n8n + PostgreSQL setup
├── .env.example                # Environment variables template
├── .env                        # Local config (git-ignored)
├── README.md                   # User documentation
├── workflows/
│   ├── lead-enrichment.json    # Main workflow
│   └── lead-enrichment-minimal.json
├── templates/
│   └── google-sheet-template.csv
├── scripts/
│   ├── icp-scoring-prompt.md   # AI prompt customization
│   └── email-generation-prompt.md
└── docs/
    ├── api-setup-guide.md
    └── troubleshooting.md
```

## Environment Variables

API keys are passed to n8n via `docker-compose.yml` environment section:

```bash
# Required
GEMINI_API_KEY      # AI processing
HUNTER_API_KEY      # Email verification + domain search
APOLLO_API_KEY      # Person + company enrichment
GOOGLE_SHEET_ID     # Output tracking

# Optional
PDL_API_KEY         # Fallback company enrichment

# n8n Config
N8N_DEFAULT_USER_EMAIL
N8N_DEFAULT_USER_PASSWORD
N8N_ENCRYPTION_KEY
DB_PASSWORD
```

Access in workflows via `$env.VARIABLE_NAME`.

## Webhook Endpoint

```text
POST /webhook/lead-enrichment

{
  "email": "user@company.com",
  "website_url": "https://company.com",
  "signup_plan": "free",
  "user_id": "12345",
  "signup_timestamp": "2024-01-15T10:30:00Z"
}
```

## Development Commands

```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f n8n

# Restart after .env changes
docker-compose restart n8n

# Check environment variables
docker exec n8n-lead-enrichment env | grep API_KEY

# Reset everything (deletes data)
docker-compose down -v
```

## Workflow Nodes Reference

| Node | Purpose |
|------|---------|
| New Signup Trigger | Webhook receiver |
| Extract Domain Info | Parse email/website domains |
| Hunter.io Domain Search | Company info from domain |
| Hunter.io Email Verify | Email validity check |
| Apollo Person Match | Person enrichment |
| Apollo Company Enrich | Company enrichment |
| PDL Company Enrich | Fallback company data |
| Fetch Website HTML | Tech stack detection (free) |
| Merge Enrichment Data | Combine all data sources |
| AI ICP Scoring | Gemini-powered scoring |
| Generate Email Variants | Gemini-powered emails |
| Notify Slack | High-value lead alerts |
| Save Gmail Draft | Draft creation |
| Add to Tracking Sheet | Google Sheets logging |

## Customization

- **ICP Scoring**: Edit `scripts/icp-scoring-prompt.md` or the AI ICP Scoring node
- **Email Templates**: Edit `scripts/email-generation-prompt.md` or the Generate Email Variants node
- **Slack Alerts**: Modify the IF node threshold (default: HIGH tier only)

## OAuth Credentials (UI Setup Required)

These cannot be configured via environment variables:

- Google Sheets OAuth2
- Gmail OAuth2
- Slack OAuth2

Configure in n8n: **Settings** → **Credentials** → **Add Credential**

## Free Tier Limits

| Service | Free Tier |
|---------|-----------|
| Gemini | 60 req/min |
| Hunter.io | 50/month |
| Apollo.io | 60 credits/month |
| PDL | 100 credits |
| HTML Scraping | Unlimited |
