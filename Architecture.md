# Architecture.md — FollowPulse System Design

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         iOS APP                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Instagram   │  │   Main UI   │  │  Push Notification      │  │
│  │ OAuth Login │  │  Dashboard  │  │  Handler                │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
└─────────┼────────────────┼─────────────────────┼─────────────────┘
          │                │                     │
          ▼                ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                       SUPABASE                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │    Auth     │  │ PostgreSQL  │  │    Edge Functions       │  │
│  │ (Instagram) │  │  Database   │  │  (webhook + scheduler)  │  │
│  └─────────────┘  └─────────────┘  └───────────┬─────────────┘  │
└────────────────────────────────────────────────┼─────────────────┘
                                                 │
          ┌──────────────────────────────────────┤
          │                                      │
          ▼                                      ▼
┌─────────────────┐                 ┌─────────────────┐
│   APIFY CLOUD   │                 │   Apple APNs    │
│                 │                 │                 │
│ instagram-      │                 │ Push Service    │
│ profile-scraper │                 │                 │
└─────────────────┘                 └─────────────────┘
```

---

## Data Flows

### Flow 1: User Onboarding
```
1. User opens app
2. Taps "Continue with Instagram"
3. App opens Instagram OAuth URL via ASWebAuthenticationSession:
   https://www.instagram.com/oauth/authorize?client_id=...&scope=instagram_business_basic
4. User logs in and authorizes (must have Professional Account)
5. Instagram redirects with authorization code
6. App sends code to Edge Function: /functions/v1/instagram-auth
7. Edge Function exchanges code for access token
8. Edge Function fetches username from Instagram Graph API
9. Edge Function creates user in public.users + generates Supabase session
10. App receives session and stores it
11. App requests push permission
12. App stores device token in database

Note: Supabase does NOT have built-in Instagram provider.
We implement custom OAuth flow via Edge Function.
```

### Flow 2: Scheduled Check
```
1. pg_cron triggers every N hours
2. Cron calls trigger-check Edge Function
3. Function queries users due for check
4. For each user: calls Apify Actor with webhookUrl
5. Apify scrapes followers
6. Apify calls webhook when done
```

### Flow 3: Process Results
```
1. apify-webhook Edge Function receives data
2. Fetches previous snapshot from database
3. Compares: previous_followers - current_followers = unfollows
4. If unfollows found:
   a. Inserts into unfollows table
   b. Sends push notification
5. Stores new snapshot
6. Updates last_checked_at
```

---

## Authentication (Instagram API with Instagram Login)

### Important Context
```
⚠️ Instagram Basic Display API was SHUT DOWN in 2024
⚠️ Supabase does NOT have built-in Instagram provider
⚠️ Users MUST have Instagram Professional Account (Business or Creator)

We use "Instagram API with Instagram Login" - the only official
Instagram authentication method available in 2025.
```

### Meta Developer Setup
```
1. Go to developers.facebook.com
2. Create App → Select "Business" type
3. Add Product → "Instagram API with Instagram Login"
4. Settings → Basic:
   - App Domains: your-project.supabase.co
   - Privacy Policy URL: (required)
   - Terms of Service URL: (optional but recommended)
5. Instagram API → Settings:
   - Valid OAuth Redirect URIs: 
     https://YOUR_PROJECT.supabase.co/functions/v1/instagram-callback
6. Add Instagram Tester accounts (for development)
7. For production: Submit App Review for instagram_business_basic permission
```

### OAuth URLs
```
Authorization:
https://www.instagram.com/oauth/authorize
  ?client_id=YOUR_APP_ID
  &redirect_uri=https://YOUR_PROJECT.supabase.co/functions/v1/instagram-callback
  &response_type=code
  &scope=instagram_business_basic

Token Exchange:
POST https://api.instagram.com/oauth/access_token
Content-Type: application/x-www-form-urlencoded

client_id=APP_ID
&client_secret=APP_SECRET
&grant_type=authorization_code
&redirect_uri=REDIRECT_URI
&code=AUTH_CODE

User Info:
GET https://graph.instagram.com/v21.0/me
  ?fields=user_id,username,account_type
  &access_token=ACCESS_TOKEN
```

### Permissions
```
instagram_business_basic (required):
- user_id: Instagram user ID
- username: Instagram handle
- account_type: BUSINESS or MEDIA_CREATOR
- media_count: Number of posts
- followers_count: Follower count (basic, not detailed list)
- follows_count: Following count

Note: This does NOT provide access to follower usernames.
We use Apify for detailed follower data.
```

### Professional Account Requirement
```
Your app should check and guide users:

1. If user has Personal Account:
   → Show instructions to convert to Professional
   → Instagram App → Settings → Account → Switch to Professional Account
   → Choose "Creator" or "Business"
   → This is FREE

2. If user has Professional Account:
   → Proceed with OAuth

Consider adding onboarding screen explaining this requirement.
```

### Custom Auth Implementation
```
Since Supabase lacks built-in Instagram:

Option A (Recommended): Custom JWT
1. Edge Function handles OAuth
2. Creates user in public.users
3. Generates custom Supabase JWT
4. Returns JWT to app

Option B: Magic Link Fallback
1. OAuth gets Instagram username + email (if available)
2. Send magic link to email
3. Link creates Supabase auth session

We use Option A for seamless UX.
```

---

## Database Schema

### Table: users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  auth_user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  instagram_username TEXT NOT NULL UNIQUE,
  instagram_user_id TEXT,
  account_type TEXT CHECK (account_type IN ('BUSINESS', 'MEDIA_CREATOR')),
  email TEXT,
  device_token TEXT,
  plan TEXT DEFAULT 'free' CHECK (plan IN ('free', 'premium')),
  check_frequency_hours INT DEFAULT 24,
  notifications_enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_checked_at TIMESTAMPTZ
);

-- Note: account_type must be BUSINESS or MEDIA_CREATOR
-- Personal accounts cannot use Instagram API with Instagram Login

-- RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users read own" ON users
  FOR SELECT USING (auth.uid() = auth_user_id);

CREATE POLICY "Users update own" ON users
  FOR UPDATE USING (auth.uid() = auth_user_id);
```

### Table: follower_snapshots
```sql
CREATE TABLE follower_snapshots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  follower_usernames TEXT[] NOT NULL,
  follower_count INT NOT NULL,
  captured_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_snapshots_user_time 
  ON follower_snapshots(user_id, captured_at DESC);

-- RLS
ALTER TABLE follower_snapshots ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users read own snapshots" ON follower_snapshots
  FOR SELECT USING (
    user_id IN (SELECT id FROM users WHERE auth_user_id = auth.uid())
  );
```

### Table: unfollows
```sql
CREATE TABLE unfollows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  username TEXT NOT NULL,
  detected_at TIMESTAMPTZ DEFAULT NOW(),
  notification_sent BOOLEAN DEFAULT false
);

CREATE INDEX idx_unfollows_user_time 
  ON unfollows(user_id, detected_at DESC);

-- RLS
ALTER TABLE unfollows ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users read own unfollows" ON unfollows
  FOR SELECT USING (
    user_id IN (SELECT id FROM users WHERE auth_user_id = auth.uid())
  );
```

### Useful Queries

**Get latest snapshot:**
```sql
SELECT * FROM follower_snapshots 
WHERE user_id = $1 
ORDER BY captured_at DESC 
LIMIT 1;
```

**Calculate unfollows (PostgreSQL array diff):**
```sql
SELECT unnest(previous.follower_usernames) 
EXCEPT 
SELECT unnest(current_followers);
```

**Get recent unfollows:**
```sql
SELECT * FROM unfollows 
WHERE user_id = $1 
ORDER BY detected_at DESC 
LIMIT 20;
```

**Users due for check:**
```sql
SELECT * FROM users 
WHERE notifications_enabled = true
  AND (
    last_checked_at IS NULL 
    OR last_checked_at < NOW() - (check_frequency_hours || ' hours')::INTERVAL
  );
```

---

## Project Structure

```
FollowPulse/
├── FollowPulse.xcodeproj
├── FollowPulse/
│   ├── App/
│   │   ├── FollowPulseApp.swift
│   │   └── ContentView.swift
│   ├── Features/
│   │   ├── Auth/
│   │   │   ├── LoginView.swift
│   │   │   └── AuthManager.swift
│   │   ├── Dashboard/
│   │   │   ├── DashboardView.swift
│   │   │   ├── UnfollowListView.swift
│   │   │   └── FollowerChartView.swift
│   │   └── Settings/
│   │       └── SettingsView.swift
│   ├── Services/
│   │   ├── SupabaseService.swift
│   │   └── NotificationService.swift
│   ├── Models/
│   │   ├── User.swift
│   │   └── Unfollow.swift
│   └── Resources/
│       └── Assets.xcassets
│
├── supabase/
│   ├── config.toml
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── functions/
│       ├── apify-webhook/
│       │   └── index.ts
│       └── trigger-check/
│           └── index.ts
│
└── docs/
    ├── Agent.md
    ├── Architecture.md (this file)
    ├── CodeSnippets.md
    ├── Payloads.md
    └── ApifySetup.md
```

---

## Design System

### Colors
```swift
extension Color {
    static let fpPrimary = Color(hex: "6366F1")    // Indigo
    static let fpUnfollow = Color(hex: "EF4444")   // Red
    static let fpNeutral = Color(hex: "6B7280")    // Gray
    static let fpBackground = Color(hex: "F9FAFB") // Light gray
    static let fpText = Color(hex: "111827")       // Near black
}
```

### Typography
```swift
// Use system fonts for consistency
.font(.largeTitle)      // Follower count
.font(.headline)        // Section headers
.font(.body)            // Regular text
.font(.caption)         // Timestamps, secondary
```

### Spacing
```swift
// Consistent spacing scale
let spacing4: CGFloat = 4
let spacing8: CGFloat = 8
let spacing12: CGFloat = 12
let spacing16: CGFloat = 16
let spacing24: CGFloat = 24
let spacing32: CGFloat = 32
```

### Components
```
LoginButton: Full-width, rounded, gradient optional
UnfollowCard: Username, avatar placeholder, timestamp
StatCard: Large number, label below, optional trend
EmptyState: Icon, title, subtitle, centered
```

---

## Screens

### 1. Login Screen
```
┌────────────────────────┐
│                        │
│      [App Logo]        │
│                        │
│      FollowPulse       │
│                        │
│  Know when someone     │
│  unfollows you         │
│                        │
│                        │
│ ┌────────────────────┐ │
│ │ Continue with      │ │
│ │ Instagram          │ │
│ └────────────────────┘ │
│                        │
│  We only access your   │
│  public profile        │
│                        │
│     Privacy Policy     │
└────────────────────────┘
```

### 2. Dashboard Screen
```
┌────────────────────────┐
│ FollowPulse    [Gear]  │
├────────────────────────┤
│                        │
│        12,847          │
│       followers        │
│        -3 today        │
│                        │
│  ┌──────────────────┐  │
│  │ [Chart 7 days]   │  │
│  └──────────────────┘  │
│                        │
│  Recent Unfollows      │
│  ────────────────────  │
│  @jane_doe             │
│  2 hours ago           │
│  ────────────────────  │
│  @john_smith           │
│  Yesterday             │
│  ────────────────────  │
│  @cool_brand           │
│  2 days ago            │
│                        │
└────────────────────────┘
```

### 3. Settings Screen
```
┌────────────────────────┐
│ ←  Settings            │
├────────────────────────┤
│                        │
│  ACCOUNT               │
│  ────────────────────  │
│  @your_username        │
│  Connected via Meta    │
│                        │
│  NOTIFICATIONS         │
│  ────────────────────  │
│  Push Alerts    [ON]   │
│                        │
│  PLAN                  │
│  ────────────────────  │
│  Free (daily checks)   │
│  [Upgrade to Premium]  │
│                        │
│  ────────────────────  │
│  [Rate this App]       │
│  [Delete My Data]      │
│  [Log Out]             │
│                        │
└────────────────────────┘
```

### 4. Empty State
```
┌────────────────────────┐
│                        │
│                        │
│          ✨            │
│                        │
│   No unfollows yet     │
│                        │
│   We'll notify you     │
│   when someone         │
│   unfollows            │
│                        │
│                        │
└────────────────────────┘
```

---

## Monetization

### Free Tier
- Check frequency: 24 hours
- Max followers: 10,000
- History: 7 days
- Notification: Daily summary

### Premium ($4.99/month)
- Check frequency: 6 hours
- Max followers: 100,000
- History: 90 days
- Notification: Instant per-unfollow
- Export CSV

### Implementation
Use RevenueCat for subscription management.

---

## Security

### Checklist
- [ ] RLS enabled on all tables
- [ ] API keys in Supabase Vault
- [ ] Device tokens not logged
- [ ] Webhook signature verification
- [ ] Rate limiting on Edge Functions
- [ ] Input sanitization

### Environment Variables (Edge Functions)
```
APIFY_TOKEN=apify_api_xxx
APNS_KEY_ID=ABC123
APNS_TEAM_ID=TEAM123
APNS_PRIVATE_KEY=-----BEGIN PRIVATE KEY-----...
```

---

*Part of FollowPulse documentation*
