# Lead Profile Enrichment Automation for n8n

Automatically enrich new signup leads, score them against your ICP, and generate personalized outreach emails using **Google Gemini AI**.

## Features

✅ **Environment-based API keys** - No manual UI configuration for most services
✅ **Multi-source enrichment** - Hunter.io, Clearbit, Wappalyzer, PDL
✅ **AI-powered ICP scoring** - Using Google Gemini (free tier available)
✅ **Personalized emails** - 3 variants per lead
✅ **Automated workflows** - Slack, Gmail, Google Sheets integration

## Quick Start

### 1. Configure Environment Variables

Copy the example file and add your API keys:

```bash
cp .env.example .env
```

Edit `.env` and add your API keys:

```env
# AI Service
GEMINI_API_KEY=your_gemini_api_key_here

# Enrichment Services
HUNTER_API_KEY=your_hunter_api_key_here
CLEARBIT_API_KEY=your_clearbit_api_key_here
WAPPALYZER_API_KEY=your_wappalyzer_api_key_here  # Optional
PDL_API_KEY=your_pdl_api_key_here  # Optional

# Google Services
GOOGLE_SHEET_ID=your_google_sheet_id_here

# n8n Configuration (change these!)
N8N_DEFAULT_USER_EMAIL=admin@yourdomain.com
N8N_DEFAULT_USER_PASSWORD=your_secure_password
N8N_ENCRYPTION_KEY=generate_a_random_32_character_string
DB_PASSWORD=your_database_password
```

**Get API Keys:**
- **Gemini**: https://makersuite.google.com/app/apikey (Free: 60 req/min)
- **Hunter.io**: https://hunter.io (Free: 50/month)
- **Clearbit**: https://clearbit.com (Free tier available)
- **Wappalyzer**: https://www.wappalyzer.com/api (Optional)
- **PDL**: https://www.peopledatalabs.com (Optional)

### 2. Start n8n

```bash
docker-compose up -d
```

Access n8n at: http://localhost:5678

Login with credentials from your `.env` file.

### 3. Setup OAuth Credentials (in n8n UI)

Some services require OAuth (these cannot be configured via .env):

**Google Sheets OAuth2:**
1. In n8n: **Settings** → **Credentials** → **Add Credential**
2. Search for "Google Sheets OAuth2 API"
3. Follow OAuth flow, name it "Google Sheets OAuth"

**Gmail OAuth2** (Optional):
1. **Settings** → **Credentials** → **Add Credential**
2. Search for "Gmail OAuth2"
3. Follow OAuth flow, name it "Gmail OAuth"

**Slack OAuth2** (Optional):
1. **Settings** → **Credentials** → **Add Credential**
2. Search for "Slack OAuth2 API"
3. Follow OAuth flow, name it "Slack OAuth"

### 4. Import Workflow

1. In n8n, go to **Workflows** → **Import from File**
2. Select `workflows/lead-enrichment.json`
3. Click **Import**
4. **Activate** the workflow

### 5. Create Google Sheet

1. Create a new Google Sheet
2. Add column headers: Timestamp, Email, Company, Website, Platform, Industry, Employees, Traffic Level, ICP Score, ICP Tier, Uses Competitor, Recommended Variant, Outreach Status, Reply Date, Outcome, Notes, AI Recommendation, Country, User Name
3. Copy the Sheet ID from URL (e.g., `https://docs.google.com/spreadsheets/d/SHEET_ID/edit`)
4. Add to `.env`: `GOOGLE_SHEET_ID=SHEET_ID`
5. Restart n8n: `docker-compose restart n8n`

### 6. Test the Workflow

Ensure workflow is **Active**, then test:

```bash
curl -X POST http://localhost:5678/webhook/lead-enrichment \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@shopify-store.com",
    "website_url": "https://example-store.com",
    "signup_plan": "free",
    "user_id": "12345",
    "signup_timestamp": "2024-01-15T10:30:00Z"
  }'
```

Expected response:
```json
{
  "success": true,
  "lead_id": "12345",
  "email": "john@shopify-store.com",
  "company": "Example Store",
  "icp_score": 85,
  "icp_tier": "HIGH",
  "enrichment_quality": 90,
  "processed_at": "2024-01-15T10:31:23Z"
}
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

## How Environment Variables Work

### In the Workflow
The workflow uses `$env` to access environment variables:

```javascript
"value": "={{ $env.HUNTER_API_KEY }}"
"value": "={{ $env.GEMINI_API_KEY }}"
```

### In Docker
The `docker-compose.yml` passes variables from `.env` to the container:

```yaml
environment:
  - HUNTER_API_KEY=${HUNTER_API_KEY}
  - GEMINI_API_KEY=${GEMINI_API_KEY}
```

### Benefits
✅ No manual credential configuration for API keys
✅ Easy to manage across environments
✅ Secure - keep `.env` out of version control
✅ Easy to rotate keys

**Note:** OAuth credentials (Google Sheets, Gmail, Slack) still need UI configuration.

## Free Services Used

| Service | Free Tier | Purpose |
|---------|-----------|---------|
| Google Gemini | 60 req/min | ICP scoring, email generation |
| Hunter.io | 50/month | Email verification, domain search |
| Clearbit | Limited | Person & company enrichment |
| Wappalyzer | Limited | Tech stack detection |
| PDL | 100 credits | Company enrichment fallback |

## Files Structure

```
lead-profile-enrichment-n8n/
├── docker-compose.yml          # n8n + PostgreSQL setup
├── .env.example                # Environment variables template
├── .gitignore                  # Excludes .env from git
├── README.md                   # This file
├── CLAUDE.md                   # Detailed specifications
├── workflows/
│   └── lead-enrichment.json    # Main n8n workflow (uses $env)
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

### Environment Variables Not Working
**Symptom:** Workflow fails with undefined API keys

**Solutions:**
1. Verify `.env` file exists in root directory
2. Restart n8n: `docker-compose restart n8n`
3. Check environment: `docker exec n8n-lead-enrichment env | grep API_KEY`

### Gemini API Errors
**Solutions:**
- Verify API key at https://makersuite.google.com/app/apikey
- Check usage limits (60 requests/minute on free tier)
- Ensure model name is `gemini-1.5-flash`

### Docker Issues
```bash
# View logs
docker-compose logs -f n8n

# Restart after .env changes
docker-compose restart n8n

# Reset everything (⚠️ deletes data)
docker-compose down -v
```

See [docs/troubleshooting.md](docs/troubleshooting.md) for more solutions.

## Cost Estimate

**Free Tier (50 leads/month):**
- Hunter.io: Free
- Gemini: Free
- **Total: $0/month**

**Paid Tier (500 leads/month):**
- Hunter.io: $49/month
- Clearbit: $99/month
- Gemini: Free
- **Total: ~$150/month**

| Volume | Hunter.io | Clearbit | Gemini | Total/month |
|--------|-----------|----------|--------|-------------|
| 50 leads | Free | Free tier | Free | $0 |
| 500 leads | $49 | $99 | Free | ~$150 |
| 2000 leads | $99 | $99+ | Free | ~$200+ |

## License

MIT
