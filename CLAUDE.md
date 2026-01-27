# Lead Profile Enrichment Automation for n8n

## Project Overview

Build an n8n automation workflow that enriches new signup leads, classifies them against ICP (Ideal Customer Profile), and enables personalized 1:1 onboarding outreach.

**Input Data:** Email Address + Website URLs(from signup)

---

## Architecture Overview

```
┌─────────────────┐
│  Trigger        │  Webhook/Database trigger on new signup
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Data Fetching  │  Parallel enrichment from multiple sources
│  (Split Branch) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Data Merge     │  Combine all enrichment data
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  AI Processing  │  ICP scoring + Email generation
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Output Actions │  Storage, Notifications, Tracking
└─────────────────┘
```

---

## Data Points to Collect

### 1. User Profile
| Data Point | Source | n8n Node |
|------------|--------|----------|
| Country | IP Geolocation / Clearbit | HTTP Request |
| Currency | Based on country mapping | Code Node |
| Stripe Data | Stripe API | Stripe Node |
| Signup Plan | Internal Database/Webhook payload | Postgres/MySQL/Webhook |
| Related users (matching email domain) | Internal Database | Postgres/MySQL Query |

### 2. Site Profile
| Data Point | Source | n8n Node |
|------------|--------|----------|
| Category | BuiltWith / Wappalyzer | HTTP Request |
| Industry | Clearbit / Apollo | HTTP Request |
| Platform (WooCommerce, Shopify, WordPress) | BuiltWith / Wappalyzer API | HTTP Request |
| Tech Stack | BuiltWith / Wappalyzer | HTTP Request |
| Related domains | BuiltWith | HTTP Request |

### 3. Company Profile
| Data Point | Source | n8n Node |
|------------|--------|----------|
| Industry | Clearbit Company API / Apollo | HTTP Request |
| Number of Employees | Clearbit / Apollo / LinkedIn | HTTP Request |
| Country | Clearbit Company API | HTTP Request |
| Revenue | Clearbit / Apollo / ZoomInfo | HTTP Request |

### 4. Site Traffic Details
| Data Point | Source | n8n Node |
|------------|--------|----------|
| Main site + subdomains | SimilarWeb API / SEMrush | HTTP Request |
| Traffic ranking | SimilarWeb / Alexa / SEMrush | HTTP Request |

### 5. Social Media Profiles
| Data Point | Source | n8n Node |
|------------|--------|----------|
| Twitter/X | Clearbit / Hunter / Manual scrape | HTTP Request |
| Facebook | Clearbit / Company website scrape | HTTP Request |
| YouTube | Google Search API / Website scrape | HTTP Request |
| LinkedIn | Clearbit / Apollo | HTTP Request |

### 6. Competitor Tool Usage
| Data Point | Source | n8n Node |
|------------|--------|----------|
| OneSignal detection | BuiltWith / Wappalyzer | HTTP Request |
| Other push notification tools | BuiltWith / Wappalyzer | HTTP Request |

---

## Required API Services & Credentials

### Primary Enrichment APIs (Choose based on budget)

```yaml
# Free/Freemium Options
- Hunter.io: Email verification, company info (50 free/month)
- Clearbit (free tier): Basic company enrichment
- BuiltWith Free: Basic tech detection
- Wappalyzer: Tech stack detection

# Paid/Premium Options
- Clearbit Enrichment API: $99+/month - Best for company data
- Apollo.io API: $49+/month - Contact + company data
- BuiltWith Pro: $295+/month - Comprehensive tech data
- SimilarWeb API: Enterprise pricing - Traffic data
- ZoomInfo: Enterprise pricing - Revenue data
```

### Required n8n Credentials to Configure

```yaml
credentials_needed:
  # Enrichment
  - clearbit_api_key
  - apollo_api_key
  - hunter_api_key
  - builtwith_api_key
  - similarweb_api_key
  
  # Internal
  - stripe_api_key
  - database_credentials (postgres/mysql)
  
  # AI
  - openai_api_key OR anthropic_api_key
  
  # Output
  - slack_oauth_token
  - google_sheets_oauth OR airtable_api_key
  - gmail_oauth OR smtp_credentials
```

---

## Workflow Implementation

### STAGE 1: Trigger Node

```yaml
node: Webhook
name: "New Signup Trigger"
config:
  http_method: POST
  path: /lead-enrichment
  response_mode: "Last Node"
expected_payload:
  email: "user@company.com"
  website_url: "https://company.com"
  signup_plan: "free|pro|enterprise"
  signup_timestamp: "ISO8601"
  user_id: "internal_id"
```

**Alternative Triggers:**
- Postgres Trigger (on new row)
- MySQL Trigger
- Stripe Node (on customer.created event)
- Segment Webhook

---

### STAGE 2: Data Enrichment (Parallel Branches)

#### Branch 2A: User Profile Enrichment

```yaml
# Node 2A-1: Extract Email Domain
node: Code
name: "Extract Domain from Email"
code: |
  const email = $input.first().json.email;
  const domain = email.split('@')[1];
  const isPersonalEmail = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com'].includes(domain);
  return {
    email,
    domain,
    isPersonalEmail,
    companyDomain: isPersonalEmail ? null : domain
  };

# Node 2A-2: Fetch Stripe Customer Data
node: Stripe
name: "Get Stripe Customer"
operation: "Search Customers"
config:
  query: "email:'{{$json.email}}'"
  
# Node 2A-3: Query Related Users by Domain
node: Postgres/MySQL
name: "Find Related Users"
operation: "Execute Query"
query: |
  SELECT user_id, email, signup_date, plan
  FROM users 
  WHERE email LIKE '%@{{$json.domain}}'
  AND email != '{{$json.email}}'
  LIMIT 10;

# Node 2A-4: Get User Country (IP-based or Clearbit)
node: HTTP Request
name: "Clearbit Person Enrichment"
config:
  method: GET
  url: "https://person.clearbit.com/v2/people/find"
  query_params:
    email: "={{$json.email}}"
  headers:
    Authorization: "Bearer {{$credentials.clearbit_api_key}}"
```

#### Branch 2B: Site Profile Enrichment

```yaml
# Node 2B-1: BuiltWith Technology Lookup
node: HTTP Request
name: "BuiltWith Tech Stack"
config:
  method: GET
  url: "https://api.builtwith.com/v21/api.json"
  query_params:
    KEY: "={{$credentials.builtwith_api_key}}"
    LOOKUP: "={{$json.website_url}}"
output_mapping:
  platform: "Extract WooCommerce|Shopify|WordPress from technologies"
  tech_stack: "Full technology list"
  competitor_tools: "Filter for OneSignal, PushEngage, etc."

# Node 2B-2: Alternative - Wappalyzer (if BuiltWith unavailable)
node: HTTP Request
name: "Wappalyzer Lookup"
config:
  method: GET
  url: "https://api.wappalyzer.com/v2/lookup/"
  query_params:
    urls: "={{$json.website_url}}"
  headers:
    x-api-key: "={{$credentials.wappalyzer_api_key}}"
```

#### Branch 2C: Company Profile Enrichment

```yaml
# Node 2C-1: Clearbit Company Enrichment
node: HTTP Request
name: "Clearbit Company"
config:
  method: GET
  url: "https://company.clearbit.com/v2/companies/find"
  query_params:
    domain: "={{$json.domain}}"
  headers:
    Authorization: "Bearer {{$credentials.clearbit_api_key}}"
output_fields:
  - name
  - industry
  - employeesRange
  - estimatedAnnualRevenue
  - country
  - city
  - linkedin_handle
  - twitter_handle
  - facebook_handle

# Node 2C-2: Apollo Company Enrichment (Alternative/Supplement)
node: HTTP Request
name: "Apollo Company Lookup"
config:
  method: POST
  url: "https://api.apollo.io/v1/organizations/enrich"
  headers:
    Content-Type: "application/json"
    Cache-Control: "no-cache"
  body:
    api_key: "={{$credentials.apollo_api_key}}"
    domain: "={{$json.domain}}"
```

#### Branch 2D: Traffic Data Enrichment

```yaml
# Node 2D-1: SimilarWeb Traffic Data
node: HTTP Request
name: "SimilarWeb Traffic"
config:
  method: GET
  url: "https://api.similarweb.com/v1/website/{{$json.domain}}/total-traffic-and-engagement/visits"
  query_params:
    api_key: "={{$credentials.similarweb_api_key}}"
    start_date: "2024-01"
    end_date: "2024-12"
    country: "world"
    granularity: "monthly"
    main_domain_only: "false"
output_fields:
  - visits
  - global_rank
  - country_rank
  - category_rank

# Node 2D-2: Alternative - SEMrush Domain Overview
node: HTTP Request  
name: "SEMrush Domain"
config:
  method: GET
  url: "https://api.semrush.com/"
  query_params:
    type: "domain_rank"
    key: "={{$credentials.semrush_api_key}}"
    domain: "={{$json.domain}}"
    database: "us"
```

---

### STAGE 3: Merge Enrichment Data

```yaml
node: Merge
name: "Combine All Enrichment Data"
mode: "Combine"
merge_by_position: true

# Then use Code node to structure
node: Code
name: "Structure Enriched Data"
code: |
  // Combine all inputs into unified lead profile
  const userProfile = $('Clearbit Person Enrichment').first()?.json || {};
  const stripeData = $('Get Stripe Customer').first()?.json || {};
  const relatedUsers = $('Find Related Users').all() || [];
  const techStack = $('BuiltWith Tech Stack').first()?.json || {};
  const companyData = $('Clearbit Company').first()?.json || {};
  const trafficData = $('SimilarWeb Traffic').first()?.json || {};
  const triggerData = $('New Signup Trigger').first().json;

  return {
    // Original Input
    email: triggerData.email,
    website_url: triggerData.website_url,
    signup_plan: triggerData.signup_plan,
    user_id: triggerData.user_id,
    
    // User Profile
    user_profile: {
      country: userProfile.geo?.country || companyData.geo?.country || 'Unknown',
      currency: mapCountryToCurrency(userProfile.geo?.country),
      stripe_customer_id: stripeData.id || null,
      stripe_subscription: stripeData.subscriptions?.data?.[0] || null,
      related_users_count: relatedUsers.length,
      related_users: relatedUsers
    },
    
    // Site Profile
    site_profile: {
      platform: detectPlatform(techStack),
      tech_stack: techStack.technologies || [],
      category: companyData.category?.sector || 'Unknown',
      industry: companyData.category?.industry || 'Unknown'
    },
    
    // Company Profile
    company_profile: {
      name: companyData.name || 'Unknown',
      industry: companyData.category?.industry || 'Unknown',
      employee_count: companyData.metrics?.employees || 'Unknown',
      employee_range: companyData.metrics?.employeesRange || 'Unknown',
      country: companyData.geo?.country || 'Unknown',
      city: companyData.geo?.city || 'Unknown',
      estimated_revenue: companyData.metrics?.estimatedAnnualRevenue || 'Unknown',
      founded_year: companyData.foundedYear || 'Unknown'
    },
    
    // Traffic
    traffic: {
      monthly_visits: trafficData.visits || 'Unknown',
      global_rank: trafficData.global_rank || 'Unknown',
      country_rank: trafficData.country_rank || 'Unknown'
    },
    
    // Social Media
    social_profiles: {
      twitter: companyData.twitter?.handle ? `https://twitter.com/${companyData.twitter.handle}` : null,
      facebook: companyData.facebook?.handle ? `https://facebook.com/${companyData.facebook.handle}` : null,
      linkedin: companyData.linkedin?.handle ? `https://linkedin.com/company/${companyData.linkedin.handle}` : null,
      youtube: null // Requires separate lookup
    },
    
    // Competitor Analysis
    competitor_tools: {
      uses_onesignal: techStack.technologies?.some(t => t.name?.toLowerCase().includes('onesignal')) || false,
      uses_pushengage: techStack.technologies?.some(t => t.name?.toLowerCase().includes('pushengage')) || false,
      uses_pushowl: techStack.technologies?.some(t => t.name?.toLowerCase().includes('pushowl')) || false,
      all_marketing_tools: filterMarketingTools(techStack.technologies)
    },
    
    enrichment_timestamp: new Date().toISOString()
  };

  // Helper functions
  function mapCountryToCurrency(country) {
    const currencyMap = {
      'United States': 'USD', 'United Kingdom': 'GBP', 'Germany': 'EUR',
      'France': 'EUR', 'India': 'INR', 'Canada': 'CAD', 'Australia': 'AUD'
    };
    return currencyMap[country] || 'USD';
  }

  function detectPlatform(techStack) {
    const techs = techStack.technologies?.map(t => t.name?.toLowerCase()) || [];
    if (techs.some(t => t?.includes('shopify'))) return 'Shopify';
    if (techs.some(t => t?.includes('woocommerce'))) return 'WooCommerce';
    if (techs.some(t => t?.includes('magento'))) return 'Magento';
    if (techs.some(t => t?.includes('wordpress'))) return 'WordPress';
    if (techs.some(t => t?.includes('bigcommerce'))) return 'BigCommerce';
    return 'Custom/Unknown';
  }

  function filterMarketingTools(technologies) {
    const marketingKeywords = ['analytics', 'marketing', 'push', 'notification', 'email', 'sms'];
    return (technologies || []).filter(t => 
      marketingKeywords.some(kw => t.name?.toLowerCase().includes(kw))
    );
  }
```

---

### STAGE 4: AI ICP Scoring

```yaml
node: OpenAI (or Anthropic)
name: "AI ICP Scoring"
operation: "Message a Model"
config:
  model: "gpt-4o" # or "claude-3-5-sonnet-20241022"
  temperature: 0.3
  system_prompt: |
    You are an ICP (Ideal Customer Profile) scoring expert. Analyze the provided lead data and return a JSON response with scoring.
    
    Our ICP criteria:
    - E-commerce platforms (Shopify, WooCommerce preferred) = High value
    - Employee count 10-500 = Sweet spot
    - Monthly traffic > 10,000 visits = Good
    - Uses competitor tools (OneSignal, etc.) = High intent
    - B2C retail/e-commerce industry = Best fit
    - Revenue $1M-$100M = Target range
    
    Return ONLY valid JSON:
    {
      "icp_score": 0-100,
      "icp_tier": "HIGH|MEDIUM|LOW",
      "scoring_breakdown": {
        "platform_fit": 0-25,
        "company_size_fit": 0-20,
        "traffic_fit": 0-20,
        "competitor_usage": 0-20,
        "industry_fit": 0-15
      },
      "key_insights": ["insight1", "insight2"],
      "recommended_approach": "string",
      "urgency": "HIGH|MEDIUM|LOW"
    }
  user_message: |
    Analyze this lead for ICP fit:
    {{JSON.stringify($json, null, 2)}}
```

---

### STAGE 5: AI Email Generation

```yaml
node: OpenAI (or Anthropic)
name: "Generate Email Variants"
operation: "Message a Model"
config:
  model: "gpt-4o"
  temperature: 0.7
  system_prompt: |
    You are an expert sales copywriter. Create personalized outreach emails based on lead data.
    
    Our product: [YOUR PRODUCT NAME] - [Brief description of what you do]
    Key value props:
    - [Value prop 1]
    - [Value prop 2]
    - [Value prop 3]
    
    Create 3 email variants:
    1. Direct value pitch (focus on immediate ROI)
    2. Problem-aware (acknowledge their current tools/challenges)
    3. Social proof (case study focused)
    
    Each email should:
    - Be personalized to their platform/industry
    - Reference specific data points about their business
    - Include a clear CTA for a call
    - Be under 150 words
    - Sound human, not templated
    
    Return as JSON:
    {
      "variant_1": {
        "subject": "string",
        "body": "string",
        "approach": "direct_value"
      },
      "variant_2": {
        "subject": "string", 
        "body": "string",
        "approach": "problem_aware"
      },
      "variant_3": {
        "subject": "string",
        "body": "string",
        "approach": "social_proof"
      },
      "recommended_variant": 1|2|3,
      "personalization_notes": "string"
    }
  user_message: |
    Create outreach emails for this lead:
    
    Lead Data:
    {{JSON.stringify($json.enriched_data, null, 2)}}
    
    ICP Analysis:
    {{JSON.stringify($json.icp_analysis, null, 2)}}
```

---

### STAGE 6: Output Actions

#### 6A: Save Email to Draft (Gmail)

```yaml
node: Gmail
name: "Save Email Draft"
operation: "Create Draft"
config:
  to: "={{$json.email}}"
  subject: "={{$json.email_variants.variant_1.subject}}"
  message: "={{$json.email_variants.variant_1.body}}"
  # Add all 3 variants as separate drafts using a loop
```

#### 6B: Save to File (Google Drive)

```yaml
node: Google Drive
name: "Save Pitch Document"
operation: "Upload File"
config:
  folder_id: "[YOUR_FOLDER_ID]"
  file_name: "Lead_{{$json.company_profile.name}}_{{$now.format('YYYY-MM-DD')}}.json"
  file_content: |
    {
      "lead_data": {{JSON.stringify($json.enriched_data)}},
      "icp_analysis": {{JSON.stringify($json.icp_analysis)}},
      "email_variants": {{JSON.stringify($json.email_variants)}},
      "generated_at": "{{$now.toISO()}}"
    }
```

#### 6C: Slack Notification (High Value Only)

```yaml
node: IF
name: "Check if High Value"
conditions:
  - value1: "={{$json.icp_analysis.icp_tier}}"
    operation: "equals"
    value2: "HIGH"

# If TRUE branch:
node: Slack
name: "Notify High Value Lead"
operation: "Send Message"
config:
  channel: "#high-value-leads"  # or use channel ID
  blocks:
    - type: "header"
      text: "🔥 High Value Lead Alert!"
    - type: "section"
      fields:
        - type: "mrkdwn"
          text: "*Company:* {{$json.company_profile.name}}"
        - type: "mrkdwn"
          text: "*Email:* {{$json.email}}"
        - type: "mrkdwn"
          text: "*ICP Score:* {{$json.icp_analysis.icp_score}}/100"
        - type: "mrkdwn"
          text: "*Platform:* {{$json.site_profile.platform}}"
        - type: "mrkdwn"
          text: "*Employees:* {{$json.company_profile.employee_range}}"
        - type: "mrkdwn"
          text: "*Traffic:* {{$json.traffic.monthly_visits}}/month"
    - type: "section"
      text: "*Key Insights:*\n{{$json.icp_analysis.key_insights.join('\n• ')}}"
    - type: "section"
      text: "*Uses Competitor Tools:* {{$json.competitor_tools.uses_onesignal ? 'OneSignal ✓' : 'None detected'}}"
    - type: "actions"
      elements:
        - type: "button"
          text: "View Lead Details"
          url: "{{$json.tracking_sheet_url}}"
        - type: "button"
          text: "Open Gmail Draft"
          url: "https://mail.google.com/mail/u/0/#drafts"
```

#### 6D: Google Sheets Tracking

```yaml
node: Google Sheets
name: "Add to Tracking Sheet"
operation: "Append Row"
config:
  spreadsheet_id: "[YOUR_SPREADSHEET_ID]"
  sheet_name: "Lead Outreach Tracker"
  columns:
    A: "={{$now.format('YYYY-MM-DD HH:mm')}}"  # Timestamp
    B: "={{$json.email}}"                        # Email
    C: "={{$json.company_profile.name}}"         # Company
    D: "={{$json.website_url}}"                  # Website
    E: "={{$json.site_profile.platform}}"        # Platform
    F: "={{$json.company_profile.industry}}"     # Industry
    G: "={{$json.company_profile.employee_range}}" # Employees
    H: "={{$json.traffic.monthly_visits}}"       # Traffic
    I: "={{$json.icp_analysis.icp_score}}"       # ICP Score
    J: "={{$json.icp_analysis.icp_tier}}"        # ICP Tier
    K: "={{$json.competitor_tools.uses_onesignal}}" # Uses OneSignal
    L: "={{$json.email_variants.recommended_variant}}" # Recommended Variant
    M: "Pending"                                  # Outreach Status
    N: ""                                         # Reply Date
    O: ""                                         # Outcome
    P: ""                                         # Notes
    Q: "={{$json.icp_analysis.recommended_approach}}" # AI Recommendation
```

---

## Google Sheets Template Structure

Create a Google Sheet with these columns:

| Column | Header | Purpose |
|--------|--------|---------|
| A | Timestamp | When lead was processed |
| B | Email | Lead email |
| C | Company | Company name |
| D | Website | Website URL |
| E | Platform | Shopify/WooCommerce/etc |
| F | Industry | Business industry |
| G | Employees | Employee range |
| H | Monthly Traffic | Site visits |
| I | ICP Score | 0-100 |
| J | ICP Tier | HIGH/MEDIUM/LOW |
| K | Uses Competitor | TRUE/FALSE |
| L | Recommended Variant | 1/2/3 |
| M | Outreach Status | Pending/Sent/Replied/Converted/Lost |
| N | Reply Date | When they replied |
| O | Outcome | Meeting Booked/Not Interested/etc |
| P | Notes | Manual notes |
| Q | AI Recommendation | Approach suggestion |

---

## Error Handling

```yaml
# Wrap each enrichment branch in Error Catcher
node: Error Trigger
name: "Handle Enrichment Errors"
config:
  continue_on_fail: true

# Add fallback values for failed enrichments
node: Code
name: "Apply Fallback Values"
code: |
  const data = $json;
  
  // Apply defaults for missing data
  return {
    ...data,
    company_profile: {
      name: data.company_profile?.name || 'Unknown Company',
      industry: data.company_profile?.industry || 'Unknown',
      employee_count: data.company_profile?.employee_count || 'Unknown',
      country: data.company_profile?.country || 'Unknown',
      estimated_revenue: data.company_profile?.estimated_revenue || 'Unknown'
    },
    traffic: {
      monthly_visits: data.traffic?.monthly_visits || 0,
      global_rank: data.traffic?.global_rank || 'Unknown'
    },
    enrichment_quality: calculateEnrichmentQuality(data)
  };
  
  function calculateEnrichmentQuality(data) {
    let score = 0;
    if (data.company_profile?.name !== 'Unknown Company') score += 20;
    if (data.company_profile?.industry !== 'Unknown') score += 20;
    if (data.traffic?.monthly_visits > 0) score += 20;
    if (data.site_profile?.platform !== 'Custom/Unknown') score += 20;
    if (data.social_profiles?.linkedin) score += 20;
    return score;
  }
```

---

## Deployment Checklist

### Pre-Deployment
- [ ] Set up all API credentials in n8n
- [ ] Create Google Sheet with tracking template
- [ ] Create Gmail drafts folder
- [ ] Create Slack channel for notifications
- [ ] Set up webhook endpoint security (if needed)
- [ ] Configure Stripe webhook (if using Stripe trigger)

### API Rate Limits to Consider
- Clearbit: 600 requests/minute
- Apollo: Varies by plan
- BuiltWith: 5 requests/second
- SimilarWeb: Varies by plan
- Add delays between enrichment calls if needed

### Testing
- [ ] Test with personal email first
- [ ] Verify each enrichment API returns data
- [ ] Test AI scoring with various lead profiles
- [ ] Verify Slack notifications fire correctly
- [ ] Confirm Google Sheets updates properly
- [ ] Test Gmail draft creation

---

## Cost Estimation (Monthly)

| Service | Free Tier | Paid Estimate |
|---------|-----------|---------------|
| Clearbit | 50 enrichments | $99-199/mo |
| Apollo | 50 credits | $49-99/mo |
| BuiltWith | 5 lookups | $295/mo |
| SimilarWeb | N/A | Enterprise |
| OpenAI GPT-4 | N/A | ~$0.03/lead |
| n8n | Self-hosted free | $20+/mo cloud |

**Budget-Friendly Alternative Stack:**
- Hunter.io (free 50/mo) + Clearbit free tier
- Wappalyzer free API for tech detection
- Skip traffic data or use free alternatives
- OpenAI GPT-3.5-turbo (~$0.002/lead)

---

## Customization Points

### ICP Scoring Weights
Modify in the AI scoring prompt:
- Platform fit weight
- Company size preferences
- Traffic thresholds
- Industry priorities
- Revenue ranges

### Email Templates
Customize in the email generation prompt:
- Your product value props
- Tone and style
- CTA variations
- Case study references

### Slack Alert Thresholds
Adjust the IF node condition:
- Score threshold for alerts
- Additional conditions (e.g., competitor tool usage)
- Multiple channels for different tiers

---

## File Structure for Claude Code

```
/lead-enrichment-n8n/
├── CLAUDE.md                 # This file
├── workflows/
│   └── lead-enrichment.json  # Exported n8n workflow
├── templates/
│   ├── google-sheet-template.csv
│   └── email-templates.md
├── scripts/
│   ├── icp-scoring-prompt.md
│   └── email-generation-prompt.md
└── docs/
    ├── api-setup-guide.md
    └── troubleshooting.md
```

---

## Quick Start Commands

```bash
# If using n8n CLI
n8n import:workflow --input=workflows/lead-enrichment.json

# Set credentials via CLI (if supported)
n8n credentials:set clearbit_api_key YOUR_KEY
```

---

## Support & Iteration

After initial deployment, iterate based on:
1. Enrichment success rate (add fallback APIs)
2. ICP scoring accuracy (refine prompts)
3. Email reply rates (A/B test variants)
4. Team feedback on Slack notifications
5. Conversion tracking in Google Sheets
