# Troubleshooting Guide

Common issues and solutions for the Lead Profile Enrichment workflow.

## Table of Contents
1. [Workflow Not Triggering](#1-workflow-not-triggering)
2. [Enrichment API Errors](#2-enrichment-api-errors)
3. [AI Processing Issues](#3-ai-processing-issues)
4. [Google Sheets Errors](#4-google-sheets-errors)
5. [Slack Notification Failures](#5-slack-notification-failures)
6. [Gmail Draft Issues](#6-gmail-draft-issues)
7. [Performance Issues](#7-performance-issues)
8. [Data Quality Issues](#8-data-quality-issues)

---

## 1. Workflow Not Triggering

### Webhook Not Receiving Data

**Symptom**: Workflow doesn't start when webhook is called.

**Solutions**:
1. **Check webhook URL**: Ensure you're using the correct URL
   ```
   http://YOUR_N8N_HOST:5678/webhook/lead-enrichment
   ```

2. **Verify workflow is active**: The workflow must be activated (toggle ON)

3. **Check n8n logs**:
   ```bash
   docker logs n8n-lead-enrichment
   ```

4. **Test webhook manually**:
   ```bash
   curl -X POST http://localhost:5678/webhook/lead-enrichment \
     -H "Content-Type: application/json" \
     -d '{
       "email": "test@example.com",
       "website_url": "https://example.com",
       "signup_plan": "free",
       "user_id": "test123"
     }'
   ```

### Webhook Returns 404

**Solutions**:
1. Ensure workflow is activated
2. Check the webhook path matches exactly
3. Restart n8n container:
   ```bash
   docker-compose restart n8n
   ```

---

## 2. Enrichment API Errors

### Hunter.io Errors

**Error**: `401 Unauthorized`
- **Cause**: Invalid API key
- **Solution**: Verify API key at [hunter.io/api](https://hunter.io/api)

**Error**: `429 Too Many Requests`
- **Cause**: Rate limit exceeded (50 free/month)
- **Solution**:
  - Wait for quota reset (monthly)
  - Upgrade to paid plan
  - Skip Hunter.io enrichment for low-priority leads

### Clearbit Errors

**Error**: `404 Not Found`
- **Cause**: No data found for email/domain
- **This is normal** - not all emails/domains have data
- The workflow handles this with `continueOnFail: true`

**Error**: `402 Payment Required`
- **Cause**: Free tier exhausted
- **Solution**: Check usage at [clearbit.com](https://clearbit.com)

### Wappalyzer Errors

**Error**: `401 Unauthorized`
- **Cause**: Invalid or expired API key
- **Solution**: Generate new key at Wappalyzer dashboard

**Error**: Timeout
- **Cause**: Website took too long to analyze
- **Solution**: Increase timeout in node settings (default: 60s)

### Handling Missing Data

The workflow is designed to handle missing data gracefully:
```javascript
// In "Structure Enriched Data" node
const companyName = clearbitCompany?.name ||
                   hunterDomain?.organization ||
                   pdlCompany?.name ||
                   'Unknown';
```

---

## 3. AI Processing Issues

### OpenAI API Errors

**Error**: `401 Unauthorized`
- **Cause**: Invalid API key
- **Solution**:
  1. Check key at [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
  2. Ensure key hasn't been rotated

**Error**: `429 Rate Limit`
- **Cause**: Too many requests or quota exceeded
- **Solution**:
  - Add delays between AI calls
  - Check billing at [platform.openai.com/usage](https://platform.openai.com/usage)

**Error**: `500 Internal Server Error`
- **Cause**: OpenAI service issue
- **Solution**: Retry or wait for service recovery

### JSON Parsing Errors

**Symptom**: "Parse ICP Analysis" or "Parse Email Variants" node fails

**Solutions**:

1. **Check AI output in execution log**
   - Look for markdown code blocks in response
   - The parse nodes handle this, but sometimes AI adds extra text

2. **Adjust temperature**
   - Lower temperature (0.2-0.3) for more consistent JSON output

3. **Update prompt**
   - Add: "Return ONLY valid JSON. No explanations, no markdown."

4. **Manual fallback**
   The parse nodes include fallback values:
   ```javascript
   } catch (e) {
     // Default fallback if parsing fails
     icpAnalysis = {
       icp_score: 50,
       icp_tier: 'MEDIUM',
       // ...
     };
   }
   ```

### Poor Quality AI Output

**Symptom**: Generic or irrelevant emails/scoring

**Solutions**:
1. **Provide more context** in prompts
2. **Use GPT-4o** instead of GPT-4o-mini for better quality
3. **Customize prompts** for your specific business
4. **Review and iterate** on prompt templates in `/scripts/`

---

## 4. Google Sheets Errors

### Authentication Errors

**Error**: `Invalid Credentials`
- **Cause**: OAuth token expired
- **Solution**:
  1. Go to n8n Credentials
  2. Delete and recreate Google Sheets credential
  3. Re-authorize

### Sheet Not Found

**Error**: `Spreadsheet not found`
- **Solutions**:
  1. Verify spreadsheet ID is correct
  2. Ensure sheet is shared with the service account
  3. Check sheet name matches exactly: "Lead Outreach Tracker"

### Permission Denied

**Error**: `The caller does not have permission`
- **Cause**: OAuth doesn't have edit access
- **Solution**: Re-authorize with correct scopes

### Column Mismatch

**Error**: Data not appearing in correct columns
- **Solution**:
  1. Ensure sheet has exact column headers from template
  2. Check for hidden characters in header names
  3. Use column letters instead of names if issues persist

---

## 5. Slack Notification Failures

### Bot Not in Channel

**Error**: `channel_not_found` or `not_in_channel`
- **Solution**:
  ```
  /invite @YourBotName
  ```
  Run this command in the target Slack channel

### Invalid Blocks

**Error**: `invalid_blocks`
- **Cause**: Block formatting error
- **Solution**:
  1. Validate blocks at [Slack Block Kit Builder](https://app.slack.com/block-kit-builder)
  2. Check for undefined values in dynamic fields

### Missing Scopes

**Error**: `missing_scope`
- **Solution**: Add required scopes to your Slack app:
  - `chat:write`
  - `chat:write.public`

---

## 6. Gmail Draft Issues

### Draft Not Created

**Error**: `Invalid grant`
- **Cause**: OAuth token revoked or expired
- **Solution**: Re-authorize Gmail credentials

### Wrong Account

**Symptom**: Drafts appear in different Gmail account
- **Solution**: Ensure correct Google account is authorized in n8n

### Formatting Issues

**Symptom**: Email body has escaped characters
- **Solution**: Use `emailType: "text"` for plain text or properly format HTML

---

## 7. Performance Issues

### Workflow Running Slowly

**Causes & Solutions**:

1. **Too many API calls**
   - Add caching for repeated domains
   - Skip optional enrichment for low-value leads

2. **Large data payloads**
   - Reduce fields passed between nodes
   - Use `$json.field` instead of `$json` where possible

3. **Sequential execution**
   - Ensure parallel enrichment nodes run simultaneously
   - Check "Execute in Parallel" settings

### Timeout Errors

**Solution**: Increase timeouts in docker-compose.yml:
```yaml
environment:
  - EXECUTIONS_TIMEOUT=300  # 5 minutes
  - EXECUTIONS_TIMEOUT_MAX=600  # 10 minutes
```

### Memory Issues

**Symptom**: Container crashes or restarts
**Solution**: Increase memory limits:
```yaml
services:
  n8n:
    deploy:
      resources:
        limits:
          memory: 2G
```

---

## 8. Data Quality Issues

### Low Enrichment Quality Score

**Causes**:
1. Personal email addresses (Gmail, Yahoo, etc.)
2. New/small companies not in databases
3. Non-English websites

**Solutions**:
1. **Prioritize business emails** in your signup flow
2. **Use website domain** as fallback for personal emails
3. **Add manual enrichment queue** for important leads

### Missing Tech Stack Data

**Cause**: Website blocks scraping or uses unusual tech
**Solutions**:
1. Increase Wappalyzer timeout
2. Use BuiltWith as alternative
3. Add manual tech detection for key accounts

### Incorrect Company Matching

**Symptom**: Wrong company data for email
**Causes**:
- Email domain != company domain
- Freelancers using personal domains
- Subsidiary companies

**Solutions**:
1. Always include website_url in webhook payload
2. Prioritize website domain over email domain
3. Add manual review for mismatches

---

## Debug Mode

### Enabling Debug Logs

In docker-compose.yml:
```yaml
environment:
  - N8N_LOG_LEVEL=debug
```

### Viewing Execution History

1. In n8n UI, go to **Executions**
2. Click on failed execution
3. Review each node's input/output

### Testing Individual Nodes

1. Open workflow editor
2. Click on node
3. Click "Execute Node" to test with sample data

---

## Common Error Reference

| Error Code | Service | Meaning | Action |
|------------|---------|---------|--------|
| 401 | Any | Authentication failed | Check API key |
| 402 | Clearbit | Payment required | Check quota |
| 403 | Any | Forbidden | Check permissions |
| 404 | Any | Not found | Resource doesn't exist |
| 429 | Any | Rate limited | Wait or upgrade |
| 500 | Any | Server error | Retry later |
| 502 | Any | Bad gateway | Service down |
| 503 | Any | Service unavailable | Retry later |

---

## Getting Help

1. **n8n Community**: [community.n8n.io](https://community.n8n.io)
2. **n8n Documentation**: [docs.n8n.io](https://docs.n8n.io)
3. **GitHub Issues**: For workflow-specific issues

### Reporting Issues

When reporting issues, include:
1. n8n version
2. Error message (full text)
3. Node configuration (screenshot)
4. Execution log (if available)
