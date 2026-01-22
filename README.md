# Lead Profile Enrichment Automation for n8n

Automatically enrich new signup leads, score them against your ICP, and generate personalized outreach emails.

## Quick Start

### 1. Start n8n

```bash
docker-compose up -d
```

Access n8n at: http://localhost:5678
- Username: `admin`
- Password: `changeme123`

### 2. Import Workflow

1. In n8n, go to **Workflows** → **Import from File**
2. Select `workflows/lead-enrichment.json`
3. Click **Import**

### 3. Configure Credentials

Set up these credentials in n8n (**Credentials** → **Add Credential**):

| Service | Type | Required |
|---------|------|----------|
| Hunter.io | HTTP Query Auth | Recommended |
| Clearbit | Header Auth | Recommended |
| OpenAI | OpenAI API | Required |
| Google Sheets | Google Sheets OAuth2 | Required |
| Gmail | Gmail OAuth2 | Optional |
| Slack | Slack OAuth2 | Optional |

See [docs/api-setup-guide.md](docs/api-setup-guide.md) for detailed instructions.

### 4. Create Google Sheet

1. Create a new Google Sheet
2. Import headers from `templates/google-sheet-template.csv`
3. Copy the spreadsheet ID and update the workflow node

### 5. Activate & Test

1. Toggle the workflow to **Active**
2. Test with:

```bash
curl -X POST http://localhost:5678/webhook/lead-enrichment \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@shopify-store.com",
    "website_url": "https://example-store.com",
    "signup_plan": "free",
    "user_id": "12345"
  }'
```

## Architecture

```
Webhook → Extract Domain → Parallel Enrichment → Merge Data
                                ↓
                         AI ICP Scoring
                                ↓
                     AI Email Generation
                                ↓
              ┌─────────────────┼─────────────────┐
        Slack Notify     Gmail Draft      Google Sheets
        (High Value)    (High Value)       (All Leads)
```

## Free Services Used

| Service | Free Tier | Purpose |
|---------|-----------|---------|
| Hunter.io | 50/month | Email verification, domain search |
| Clearbit | Limited | Person & company enrichment |
| Wappalyzer | Limited | Tech stack detection |
| PDL | 100 credits | Company enrichment fallback |
| OpenAI | Pay-per-use (~$0.002/lead) | ICP scoring, email generation |

## Files Structure

```
lead-profile-enrichment-n8n/
├── docker-compose.yml          # n8n + PostgreSQL setup
├── README.md                   # This file
├── CLAUDE.md                   # Detailed specifications
├── workflows/
│   └── lead-enrichment.json    # Main n8n workflow
├── templates/
│   └── google-sheet-template.csv
├── scripts/
│   ├── icp-scoring-prompt.md   # Customizable AI prompt
│   └── email-generation-prompt.md
└── docs/
    ├── api-setup-guide.md      # API credential setup
    └── troubleshooting.md      # Common issues & fixes
```

## Customization

### ICP Scoring Criteria
Edit the AI prompt in the "AI ICP Scoring" node or customize via [scripts/icp-scoring-prompt.md](scripts/icp-scoring-prompt.md).

### Email Templates
Edit the AI prompt in the "Generate Email Variants" node or customize via [scripts/email-generation-prompt.md](scripts/email-generation-prompt.md).

### Slack Notifications
Change the channel name and notification format in the "Notify Slack - High Value" node.

## Webhook Payload

```json
{
  "email": "user@company.com",        // Required
  "website_url": "https://company.com", // Required
  "signup_plan": "free",              // Optional
  "user_id": "12345",                 // Optional
  "signup_timestamp": "2024-01-15T10:30:00Z" // Optional
}
```

## Response

```json
{
  "success": true,
  "lead_id": "12345",
  "email": "user@company.com",
  "company": "Company Name",
  "icp_score": 85,
  "icp_tier": "HIGH",
  "enrichment_quality": 80,
  "processed_at": "2024-01-15T10:30:05Z"
}
```

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md) for common issues and solutions.

## Cost Estimate

| Volume | Hunter.io | OpenAI | Total/month |
|--------|-----------|--------|-------------|
| 50 leads | Free | ~$0.10 | ~$0.10 |
| 500 leads | $49 | ~$1.00 | ~$50 |
| 2000 leads | $99 | ~$4.00 | ~$103 |

## License

MIT
