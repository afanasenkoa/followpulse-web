# Payloads.md — JSON Examples for All Integrations

## Apify

### Actor Input (test-input.json)
```json
{
  "operationMode": "analyzeFollowersFollowing",
  "username": "your_test_account",
  "analyzeFollowers": true,
  "analyzeFollowing": false,
  "maxCount": 100,
  "extractEmail": false,
  "extractPhoneNumber": false,
  "extractWebsiteUrl": false,
  "extractBusinessCategory": false,
  "analyzeQuality": false,
  "extractPosts": false
}
```

### Actor Output (Dataset Items)
```json
[
  {
    "username": "follower_1",
    "fullName": "John Doe",
    "profilePicUrl": "https://...",
    "followersCount": 1234,
    "followingCount": 567,
    "isVerified": false,
    "isPrivate": false,
    "biography": "..."
  },
  {
    "username": "follower_2",
    "fullName": "Jane Smith",
    "profilePicUrl": "https://...",
    "followersCount": 890,
    "followingCount": 432,
    "isVerified": false,
    "isPrivate": false,
    "biography": "..."
  }
]
```

### Webhook Payload (ACTOR.RUN.SUCCEEDED)
```json
{
  "eventType": "ACTOR.RUN.SUCCEEDED",
  "createdAt": "2025-01-02T10:30:00.000Z",
  "resource": {
    "id": "abc123xyz",
    "actId": "afanasenko/instagram-profile-scraper",
    "status": "SUCCEEDED",
    "startedAt": "2025-01-02T10:25:00.000Z",
    "finishedAt": "2025-01-02T10:30:00.000Z",
    "defaultDatasetId": "dataset_xyz789",
    "defaultDataset": {
      "itemCount": 1234
    }
  },
  "customData": {
    "userId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Fetching Dataset Items
```bash
# Via CLI
apify datasets get-items dataset_xyz789 --format=json

# Via API
curl "https://api.apify.com/v2/datasets/dataset_xyz789/items?token=YOUR_TOKEN"
```

### Dataset Items Response
```json
{
  "items": [
    { "username": "follower_1", ... },
    { "username": "follower_2", ... }
  ],
  "total": 1234,
  "offset": 0,
  "count": 100,
  "limit": 100
}
```

---

## Supabase

### Custom JWT Session (from instagram-callback Edge Function)
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "instagram_username": "johndoe",
    "instagram_user_id": "12345678",
    "account_type": "BUSINESS"
  },
  "expires_at": "2025-02-01T10:00:00.000Z"
}
```

Note: Supabase does NOT have built-in Instagram provider.
We use custom OAuth flow with Edge Function that returns JWT.

### User Record (public.users)
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "auth_user_id": "auth0|123456789",
  "instagram_username": "johndoe",
  "instagram_user_id": "12345678",
  "account_type": "BUSINESS",
  "email": "john@example.com",
  "device_token": "abc123devicetoken456...",
  "plan": "free",
  "check_frequency_hours": 24,
  "notifications_enabled": true,
  "created_at": "2025-01-02T10:00:00.000Z",
  "updated_at": "2025-01-02T10:00:00.000Z",
  "last_checked_at": "2025-01-02T18:00:00.000Z"
}
```

### Follower Snapshot Record
```json
{
  "id": "660e8400-e29b-41d4-a716-446655440001",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "follower_usernames": [
    "follower_1",
    "follower_2",
    "follower_3",
    "follower_4"
  ],
  "follower_count": 4,
  "captured_at": "2025-01-02T18:00:00.000Z"
}
```

### Unfollow Record
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "former_follower",
  "detected_at": "2025-01-02T18:00:00.000Z",
  "notification_sent": true
}
```

### Edge Function Request Headers
```json
{
  "Authorization": "Bearer service_role_key_here",
  "Content-Type": "application/json",
  "x-client-info": "supabase-js/2.0.0"
}
```

### Edge Function Error Response
```json
{
  "error": "Error message here",
  "code": "ERROR_CODE",
  "details": "Additional details if available"
}
```

---

## Apple Push Notification Service (APNs)

### Push Payload — Single Unfollow (Premium)
```json
{
  "aps": {
    "alert": {
      "title": "FollowPulse",
      "body": "@jane_doe unfollowed you"
    },
    "badge": 1,
    "sound": "default"
  },
  "data": {
    "type": "unfollow",
    "username": "jane_doe",
    "detected_at": "2025-01-02T18:00:00.000Z"
  }
}
```

### Push Payload — Summary (Free)
```json
{
  "aps": {
    "alert": {
      "title": "FollowPulse",
      "body": "3 people unfollowed you today"
    },
    "badge": 3,
    "sound": "default"
  },
  "data": {
    "type": "daily_summary",
    "unfollow_count": 3,
    "date": "2025-01-02"
  }
}
```

### APNs Request Headers
```
:method: POST
:path: /3/device/<device_token>
authorization: bearer <jwt_token>
apns-topic: com.yourcompany.followpulse
apns-push-type: alert
apns-priority: 10
apns-expiration: 0
```

### Device Token Format
```
# Raw data from iOS (64 bytes)
<32bytes><32bytes>

# Hex string format (128 characters)
a1b2c3d4e5f6... (128 hex characters)
```

---

## Instagram API with Instagram Login (Official 2025)

### Important Notes
```
⚠️ Instagram Basic Display API was SHUT DOWN in 2024
⚠️ This is the ONLY official Instagram auth method available
⚠️ Requires Instagram Professional Account (Business or Creator)
⚠️ Supabase does NOT have built-in Instagram provider
```

### OAuth Authorization URL
```
https://www.instagram.com/oauth/authorize
  ?client_id=YOUR_APP_ID
  &redirect_uri=https://YOUR_PROJECT.supabase.co/functions/v1/instagram-callback
  &response_type=code
  &scope=instagram_business_basic
```

### Token Exchange Request
```
POST https://api.instagram.com/oauth/access_token
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_APP_ID
&client_secret=YOUR_APP_SECRET
&grant_type=authorization_code
&redirect_uri=https://YOUR_PROJECT.supabase.co/functions/v1/instagram-callback
&code=AUTH_CODE_FROM_REDIRECT
```

Note: Must be `application/x-www-form-urlencoded`, NOT JSON.

### Token Exchange Response
```json
{
  "access_token": "IGQVJYeU...",
  "user_id": 12345678
}
```

### User Profile Request
```
GET https://graph.instagram.com/v21.0/me
  ?fields=user_id,username,account_type
  &access_token=IGQVJYeU...
```

### User Profile Response
```json
{
  "user_id": "12345678",
  "username": "johndoe",
  "account_type": "BUSINESS"
}
```

### Account Types
```
BUSINESS     — Business accounts
MEDIA_CREATOR — Creator accounts
PERSONAL     — Will NOT work (OAuth will fail)
```

### OAuth Error Response
```json
{
  "error_type": "OAuthException",
  "code": 400,
  "error_message": "Invalid OAuth access token."
}
```

### Common OAuth Errors
```
"Invalid redirect_uri"  → Check exact match in Meta App settings
"Invalid scope"         → User doesn't have Professional Account
"Access token expired"  → Token is short-lived, exchange for long-lived
```

---

## Supabase Realtime (Optional)

### Subscribe to Unfollows
```json
{
  "event": "postgres_changes",
  "schema": "public",
  "table": "unfollows",
  "filter": "user_id=eq.550e8400-e29b-41d4-a716-446655440000"
}
```

### Realtime Payload
```json
{
  "type": "INSERT",
  "table": "unfollows",
  "schema": "public",
  "record": {
    "id": "770e8400-e29b-41d4-a716-446655440002",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "former_follower",
    "detected_at": "2025-01-02T18:00:00.000Z",
    "notification_sent": false
  },
  "old_record": null
}
```

---

## RevenueCat (Subscriptions)

### Customer Info Response
```json
{
  "subscriber": {
    "original_app_user_id": "550e8400-e29b-41d4-a716-446655440000",
    "subscriptions": {
      "premium_monthly": {
        "expires_date": "2025-02-02T10:00:00.000Z",
        "purchase_date": "2025-01-02T10:00:00.000Z",
        "is_active": true,
        "will_renew": true
      }
    },
    "entitlements": {
      "premium": {
        "is_active": true,
        "product_identifier": "premium_monthly",
        "expires_date": "2025-02-02T10:00:00.000Z"
      }
    }
  }
}
```

### Webhook Event (Subscription Started)
```json
{
  "event": {
    "type": "INITIAL_PURCHASE",
    "app_user_id": "550e8400-e29b-41d4-a716-446655440000",
    "product_id": "premium_monthly",
    "price": 4.99,
    "currency": "USD",
    "purchased_at_ms": 1704189600000
  }
}
```

---

## Mock Data Files

### /mocks/apify-input.json
```json
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
```

### /mocks/followers-sample.json
```json
[
  { "username": "user_001" },
  { "username": "user_002" },
  { "username": "user_003" },
  { "username": "user_004" },
  { "username": "user_005" }
]
```

### /mocks/unfollows-sample.json
```json
[
  {
    "id": "mock-001",
    "username": "former_follower_1",
    "detected_at": "2025-01-02T18:00:00.000Z"
  },
  {
    "id": "mock-002",
    "username": "former_follower_2",
    "detected_at": "2025-01-01T12:00:00.000Z"
  },
  {
    "id": "mock-003",
    "username": "former_follower_3",
    "detected_at": "2024-12-30T09:00:00.000Z"
  }
]
```

---

*Part of FollowPulse documentation*
