# API Setup Guide

This guide walks you through setting up all the required API credentials for the Lead Profile Enrichment workflow.

## Table of Contents
1. [Hunter.io](#1-hunterio-free-50month)
2. [Clearbit](#2-clearbit-free-tier-available)
3. [Wappalyzer](#3-wappalyzer-optional)
4. [People Data Labs](#4-people-data-labs-optional)
5. [OpenAI](#5-openai-required)
6. [Google Sheets](#6-google-sheets-required)
7. [Gmail](#7-gmail-optional)
8. [Slack](#8-slack-optional)

---

## 1. Hunter.io (Free: 50/month)

Hunter.io provides email verification and domain search capabilities.

### Getting API Key
1. Sign up at [https://hunter.io](https://hunter.io)
2. Go to **API** section in your dashboard
3. Copy your API key

### Free Tier Limits
- 25 email verifications/month
- 25 domain searches/month
- Email finder available

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "HTTP Query Auth" or use the HTTP Request node
3. Configure:
   - **Name**: `Hunter API`
   - **Parameter Name**: `api_key`
   - **Value**: Your Hunter.io API key

### Alternative: Direct HTTP Configuration
In the HTTP Request node, add query parameter:
```
name: api_key
value: YOUR_HUNTER_API_KEY
```

---

## 2. Clearbit (Free Tier Available)

Clearbit provides person and company enrichment data.

### Getting API Key
1. Sign up at [https://clearbit.com](https://clearbit.com)
2. Navigate to **API** in settings
3. Copy your API key (starts with `sk_`)

### Free Tier Limits
- Limited enrichments per month
- Basic data only
- Check current limits at signup

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "Header Auth"
3. Configure:
   - **Name**: `Clearbit API`
   - **Header Name**: `Authorization`
   - **Header Value**: `Bearer YOUR_CLEARBIT_API_KEY`

### Endpoints Used
- Person Lookup: `https://person.clearbit.com/v2/people/find?email=EMAIL`
- Company Lookup: `https://company.clearbit.com/v2/companies/find?domain=DOMAIN`

---

## 3. Wappalyzer (Optional)

Wappalyzer detects technologies used on websites.

### Getting API Key
1. Sign up at [https://www.wappalyzer.com](https://www.wappalyzer.com)
2. Go to **API** section
3. Generate an API key

### Free Tier Limits
- Limited lookups (check current offering)
- Basic technology detection

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "Header Auth"
3. Configure:
   - **Name**: `Wappalyzer API`
   - **Header Name**: `x-api-key`
   - **Header Value**: Your Wappalyzer API key

### Alternative: BuiltWith
If Wappalyzer isn't available, use BuiltWith:
1. Sign up at [https://builtwith.com/api](https://builtwith.com/api)
2. Free tier: 5 lookups/day
3. Endpoint: `https://api.builtwith.com/v21/api.json?KEY=YOUR_KEY&LOOKUP=DOMAIN`

---

## 4. People Data Labs (Optional)

PDL provides company enrichment as a fallback data source.

### Getting API Key
1. Sign up at [https://www.peopledatalabs.com](https://www.peopledatalabs.com)
2. Navigate to API section
3. Copy your API key

### Free Tier Limits
- 100 free credits on signup
- Limited enrichments

### n8n Credential Setup
Use as query parameter:
```
name: api_key
value: YOUR_PDL_API_KEY
```

### Endpoint Used
- Company Enrich: `https://api.peopledatalabs.com/v5/company/enrich?website=DOMAIN&api_key=KEY`

---

## 5. OpenAI (Required)

OpenAI powers the ICP scoring and email generation.

### Getting API Key
1. Sign up at [https://platform.openai.com](https://platform.openai.com)
2. Go to **API Keys** section
3. Create a new API key

### Cost Estimate
Using GPT-4o-mini:
- Input: $0.15 per 1M tokens
- Output: $0.60 per 1M tokens
- Estimated cost per lead: ~$0.002-0.005

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "OpenAI"
3. Configure:
   - **API Key**: Your OpenAI API key

### Model Selection
- **GPT-4o-mini**: Best cost/performance ratio (recommended)
- **GPT-4o**: Higher quality, higher cost
- **GPT-3.5-turbo**: Lower cost, lower quality

---

## 6. Google Sheets (Required)

Google Sheets stores your lead tracking data.

### Setting Up OAuth
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project or select existing
3. Enable Google Sheets API:
   - Go to **APIs & Services** → **Library**
   - Search "Google Sheets API"
   - Click **Enable**
4. Create OAuth credentials:
   - Go to **APIs & Services** → **Credentials**
   - Click **Create Credentials** → **OAuth client ID**
   - Select **Web application**
   - Add authorized redirect URI: `https://YOUR_N8N_URL/rest/oauth2-credential/callback`
5. Copy Client ID and Client Secret

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "Google Sheets OAuth2 API"
3. Enter your Client ID and Client Secret
4. Click **Connect** and authorize

### Creating the Tracking Sheet
1. Create a new Google Sheet
2. Import the template from `templates/google-sheet-template.csv`
3. Copy the spreadsheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_HERE/edit
   ```
4. Add this ID to the "Add to Tracking Sheet" node

---

## 7. Gmail (Optional)

Gmail creates draft emails for outreach.

### Setting Up OAuth
1. Use the same Google Cloud project
2. Enable Gmail API:
   - Go to **APIs & Services** → **Library**
   - Search "Gmail API"
   - Click **Enable**
3. Use the same OAuth credentials

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "Gmail OAuth2 API"
3. Enter your Client ID and Client Secret
4. Click **Connect** and authorize

### Required Scopes
- `https://www.googleapis.com/auth/gmail.compose`
- `https://www.googleapis.com/auth/gmail.modify`

---

## 8. Slack (Optional)

Slack sends notifications for high-value leads.

### Creating Slack App
1. Go to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. Name it (e.g., "Lead Enrichment Bot")
4. Select your workspace

### Configuring Permissions
1. Go to **OAuth & Permissions**
2. Add Bot Token Scopes:
   - `chat:write`
   - `chat:write.public`
3. Click **Install to Workspace**
4. Copy the **Bot User OAuth Token**

### n8n Credential Setup
1. In n8n, go to **Credentials** → **Add Credential**
2. Search for "Slack OAuth2 API"
3. Enter your credentials:
   - **Client ID**: From app settings
   - **Client Secret**: From app settings
   - **Access Token**: Bot User OAuth Token

### Creating the Channel
1. Create a new Slack channel (e.g., `#high-value-leads`)
2. Invite the bot to the channel: `/invite @YourBotName`
3. Update the channel name in the Slack node

---

## Credential Checklist

| Service | Required | Free Tier | Status |
|---------|----------|-----------|--------|
| Hunter.io | Recommended | 50/month | ☐ |
| Clearbit | Recommended | Limited | ☐ |
| Wappalyzer | Optional | Limited | ☐ |
| PDL | Optional | 100 credits | ☐ |
| OpenAI | Required | Pay-per-use | ☐ |
| Google Sheets | Required | Free | ☐ |
| Gmail | Optional | Free | ☐ |
| Slack | Optional | Free | ☐ |

---

## Testing Your Credentials

### Quick Test Workflow
Create a simple test workflow:

1. **Manual Trigger** node
2. **HTTP Request** node to test each API:

```json
// Hunter.io Test
GET https://api.hunter.io/v2/account?api_key=YOUR_KEY

// Clearbit Test
GET https://person.clearbit.com/v2/people/find?email=test@example.com
Headers: Authorization: Bearer YOUR_KEY

// OpenAI Test
POST https://api.openai.com/v1/chat/completions
Headers: Authorization: Bearer YOUR_KEY
Body: {"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "test"}]}
```

### Expected Responses
- **Hunter.io**: Returns account info and remaining credits
- **Clearbit**: Returns 404 if no data (normal), 200 with data if found
- **OpenAI**: Returns a chat completion response

---

## Troubleshooting

### Common Issues

**401 Unauthorized**
- API key is incorrect or expired
- Check for extra spaces in the key

**403 Forbidden**
- API key doesn't have required permissions
- Free tier limits exceeded

**429 Too Many Requests**
- Rate limit exceeded
- Add delays between API calls

**CORS Errors**
- Only affects browser-based requests
- n8n server-side calls should work fine

### Rate Limiting
Add a "Wait" node between parallel enrichment calls if needed:
- Hunter.io: 10 requests/second
- Clearbit: 600 requests/minute
- Wappalyzer: Varies by plan

---

## Security Best Practices

1. **Never commit API keys** to version control
2. **Use n8n's credential system** instead of hardcoding
3. **Rotate keys periodically** (every 90 days recommended)
4. **Monitor usage** to detect unauthorized access
5. **Use environment variables** in production:
   ```yaml
   environment:
     - HUNTER_API_KEY=${HUNTER_API_KEY}
     - CLEARBIT_API_KEY=${CLEARBIT_API_KEY}
   ```
