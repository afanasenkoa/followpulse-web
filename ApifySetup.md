# ApifySetup.md — Apify Configuration Guide

## Overview

Your Actor: `afanasenko/instagram-profile-scraper`  
Purpose: Fetch follower list for a given Instagram username  
Integration: Webhook calls your Supabase Edge Function when complete

---

## Step 1: Get Your Apify Token

1. Go to [console.apify.com](https://console.apify.com)
2. Click your profile icon (top right)
3. Select **Settings**
4. Go to **Integrations** tab
5. Copy your **Personal API Token**

```
Token format: apify_api_XXXXXXXXXXXXXXXXXXXXXXXXX
```

⚠️ Keep this secret! Never commit to git.

---

## Step 2: Install and Configure CLI

```bash
# Install
npm install -g apify-cli

# Login with your token
apify login

# Verify
apify whoami
```

---

## Step 3: Test Actor Manually

Create test input file:
```bash
cat > test-input.json << 'EOF'
{
  "operationMode": "analyzeFollowersFollowing",
  "username": "instagram",
  "analyzeFollowers": true,
  "analyzeFollowing": false,
  "maxCount": 20,
  "extractEmail": false,
  "extractPhoneNumber": false,
  "analyzeQuality": false
}
EOF
```

Run the Actor:
```bash
apify call afanasenko/instagram-profile-scraper --input-file=test-input.json
```

Check results:
```bash
# Wait for completion, then:
apify datasets get-items --limit=10
```

Expected output: List of follower objects with `username` field.

---

## Step 4: Configure Webhook

### Option A: Via API (Recommended for Production)

When triggering the Actor via API, include webhook in the request:

```bash
curl -X POST "https://api.apify.com/v2/acts/afanasenko~instagram-profile-scraper/runs" \
  -H "Authorization: Bearer YOUR_APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "operationMode": "analyzeFollowersFollowing",
      "username": "TARGET_USERNAME",
      "analyzeFollowers": true,
      "maxCount": 10000
    },
    "webhooks": [{
      "eventTypes": ["ACTOR.RUN.SUCCEEDED"],
      "requestUrl": "https://YOUR_PROJECT.supabase.co/functions/v1/apify-webhook",
      "payloadTemplate": "{\"resource\": {{resource}}, \"customData\": {\"userId\": \"USER_ID_HERE\"}}"
    }]
  }'
```

### Option B: Via Apify Console (For Testing)

1. Go to [console.apify.com](https://console.apify.com)
2. Navigate to your Actor runs or create new run
3. Click **Integrations** tab (or **Webhooks** in Actor settings)
4. Click **Add webhook**
5. Configure:
   - **Event types**: `ACTOR.RUN.SUCCEEDED`
   - **Request URL**: `https://YOUR_PROJECT.supabase.co/functions/v1/apify-webhook`
   - **Payload template**: (see below)

Payload Template:
```json
{
  "eventType": {{eventType}},
  "createdAt": {{createdAt}},
  "resource": {{resource}},
  "customData": {
    "userId": "YOUR_USER_ID"
  }
}
```

---

## Step 5: Test Webhook Locally

### Using ngrok

1. Install ngrok:
```bash
brew install ngrok  # macOS
# or download from ngrok.com
```

2. Start your local Edge Function emulator:
```bash
supabase functions serve apify-webhook
```

3. Expose to internet:
```bash
ngrok http 54321
```

4. Copy the ngrok URL (e.g., `https://abc123.ngrok.io`)

5. Run Actor with ngrok webhook:
```bash
curl -X POST "https://api.apify.com/v2/acts/afanasenko~instagram-profile-scraper/runs" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "operationMode": "analyzeFollowersFollowing",
      "username": "instagram",
      "analyzeFollowers": true,
      "maxCount": 10
    },
    "webhooks": [{
      "eventTypes": ["ACTOR.RUN.SUCCEEDED"],
      "requestUrl": "https://abc123.ngrok.io/functions/v1/apify-webhook"
    }]
  }'
```

6. Watch ngrok terminal for incoming webhook

---

## Step 6: Deploy to Production

1. Deploy Edge Function:
```bash
supabase functions deploy apify-webhook
```

2. Set Apify token as secret:
```bash
supabase secrets set APIFY_TOKEN=apify_api_xxx
```

3. Get your function URL:
```
https://YOUR_PROJECT.supabase.co/functions/v1/apify-webhook
```

4. Update trigger-check function to use this URL in webhook config.

---

## Webhook Payload Reference

### What Apify Sends

```json
{
  "eventType": "ACTOR.RUN.SUCCEEDED",
  "createdAt": "2025-01-02T10:30:00.000Z",
  "resource": {
    "id": "run_abc123",
    "actId": "afanasenko/instagram-profile-scraper",
    "status": "SUCCEEDED",
    "defaultDatasetId": "dataset_xyz789"
  },
  "customData": {
    "userId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Fetching Results

The webhook doesn't contain the actual data. You need to fetch it:

```typescript
// In your Edge Function
const datasetId = payload.resource.defaultDatasetId

const response = await fetch(
  `https://api.apify.com/v2/datasets/${datasetId}/items?token=${APIFY_TOKEN}`
)

const items = await response.json()
const followerUsernames = items.map(item => item.username)
```

---

## Cost Control

### Limit Followers Scraped

```json
{
  "maxCount": 1000  // Limit to 1000 followers
}
```

Cost per run ≈ $0.10 (list fetch) + $0.01 × profiles analyzed

### Skip Expensive Analysis

```json
{
  "extractEmail": false,
  "extractPhoneNumber": false,
  "analyzeQuality": false,
  "extractPosts": false
}
```

This reduces cost to just the list fetch ($0.10).

### Free Tier Limits

For free users (10k follower limit):
```json
{
  "maxCount": 10000
}
```

For premium users (100k follower limit):
```json
{
  "maxCount": 100000
}
```

---

## Troubleshooting

### Webhook Not Received

1. Check Actor run succeeded:
```bash
apify runs ls --limit=1
```

2. Check webhook was configured:
```bash
apify runs info RUN_ID --json | jq '.webhooks'
```

3. Check Supabase function logs:
```bash
supabase functions logs apify-webhook
```

### Empty Results

1. Check if account is public
2. Check if account exists
3. Try with a known account like `instagram`

### Rate Limits

Apify may rate limit requests. If you see 429 errors:
- Wait 15-30 minutes
- Reduce concurrent runs
- Upgrade Apify plan if needed

### Actor Failed

```bash
# Check run status
apify runs info RUN_ID

# Check logs
apify runs log RUN_ID
```

Common issues:
- Instagram blocking (try later)
- Account is private
- Account doesn't exist
- Rate limited

---

## Quick Reference

### CLI Commands

```bash
# Run Actor
apify call afanasenko/instagram-profile-scraper --input-file=test-input.json

# Check status
apify runs ls --limit=5

# Get results
apify datasets get-items --limit=100

# View logs
apify runs log RUN_ID
```

### API Endpoints

```
# Run Actor
POST https://api.apify.com/v2/acts/afanasenko~instagram-profile-scraper/runs

# Get run info
GET https://api.apify.com/v2/actor-runs/RUN_ID

# Get dataset items
GET https://api.apify.com/v2/datasets/DATASET_ID/items
```

### Environment Variables

```bash
# For Supabase Edge Functions
APIFY_TOKEN=apify_api_xxx

# Set via CLI
supabase secrets set APIFY_TOKEN=apify_api_xxx
```

---

*Part of FollowPulse documentation*
