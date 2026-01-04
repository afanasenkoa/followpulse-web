# CodeSnippets.md — Ready-to-Use Code

## iOS / SwiftUI

### Supabase Client Setup
```swift
// SupabaseService.swift
import Supabase

class SupabaseService {
    static let shared = SupabaseService()
    
    let client: SupabaseClient
    
    private init() {
        client = SupabaseClient(
            supabaseURL: URL(string: "https://YOUR_PROJECT.supabase.co")!,
            supabaseKey: "YOUR_ANON_KEY"
        )
    }
}

// Usage anywhere:
// let supabase = SupabaseService.shared.client
```

### Auth Manager (Custom Instagram OAuth)
```swift
// AuthManager.swift
import SwiftUI
import AuthenticationServices

@MainActor
class AuthManager: NSObject, ObservableObject {
    @Published var isAuthenticated = false
    @Published var currentUser: AppUser?
    @Published var isLoading = false
    @Published var error: String?
    
    private let supabase = SupabaseService.shared.client
    
    // Your Meta App credentials
    private let clientId = "YOUR_INSTAGRAM_APP_ID"
    private let redirectUri = "https://YOUR_PROJECT.supabase.co/functions/v1/instagram-callback"
    
    override init() {
        super.init()
        Task {
            await checkSession()
        }
    }
    
    func checkSession() async {
        // Check if we have a stored session
        if let token = UserDefaults.standard.string(forKey: "supabase_token") {
            do {
                // Verify token is still valid
                let user = try await supabase
                    .from("users")
                    .select()
                    .single()
                    .execute()
                    .value as AppUser
                
                currentUser = user
                isAuthenticated = true
            } catch {
                // Token invalid, clear it
                UserDefaults.standard.removeObject(forKey: "supabase_token")
                isAuthenticated = false
            }
        }
    }
    
    func signInWithInstagram() async {
        isLoading = true
        error = nil
        
        // Build Instagram OAuth URL
        let scope = "instagram_business_basic"
        let authURL = URL(string: """
            https://www.instagram.com/oauth/authorize\
            ?client_id=\(clientId)\
            &redirect_uri=\(redirectUri.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)!)\
            &response_type=code\
            &scope=\(scope)
            """)!
        
        // Start OAuth session
        await startOAuthSession(url: authURL)
    }
    
    private func startOAuthSession(url: URL) async {
        await withCheckedContinuation { continuation in
            let session = ASWebAuthenticationSession(
                url: url,
                callbackURLScheme: "followpulse"
            ) { [weak self] callbackURL, error in
                Task { @MainActor in
                    if let error = error {
                        self?.error = error.localizedDescription
                        self?.isLoading = false
                        continuation.resume()
                        return
                    }
                    
                    guard let callbackURL = callbackURL,
                          let token = self?.extractToken(from: callbackURL) else {
                        self?.error = "Failed to get authentication token"
                        self?.isLoading = false
                        continuation.resume()
                        return
                    }
                    
                    // Store token and load user
                    UserDefaults.standard.set(token, forKey: "supabase_token")
                    await self?.loadUser()
                    self?.isAuthenticated = true
                    self?.isLoading = false
                    continuation.resume()
                }
            }
            
            session.presentationContextProvider = self
            session.prefersEphemeralWebBrowserSession = false
            session.start()
        }
    }
    
    private func extractToken(from url: URL) -> String? {
        // The Edge Function will redirect with token in URL
        // followpulse://auth/callback?token=xxx
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false)
        return components?.queryItems?.first(where: { $0.name == "token" })?.value
    }
    
    private func loadUser() async {
        do {
            let user: AppUser = try await supabase
                .from("users")
                .select()
                .single()
                .execute()
                .value
            
            currentUser = user
        } catch {
            self.error = "Failed to load user profile"
        }
    }
    
    func signOut() async {
        UserDefaults.standard.removeObject(forKey: "supabase_token")
        isAuthenticated = false
        currentUser = nil
    }
}

// Required for ASWebAuthenticationSession
extension AuthManager: ASWebAuthenticationPresentationContextProviding {
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        guard let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene,
              let window = scene.windows.first else {
            return ASPresentationAnchor()
        }
        return window
    }
}
```

### Instagram Auth Edge Function
```typescript
// supabase/functions/instagram-callback/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"
import { SignJWT } from "https://deno.land/x/jose@v4.14.4/index.ts"

const supabaseUrl = Deno.env.get("SUPABASE_URL")!
const supabaseServiceKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
const instagramAppId = Deno.env.get("INSTAGRAM_APP_ID")!
const instagramAppSecret = Deno.env.get("INSTAGRAM_APP_SECRET")!
const jwtSecret = Deno.env.get("JWT_SECRET")!
const redirectUri = `${supabaseUrl}/functions/v1/instagram-callback`

serve(async (req) => {
  const url = new URL(req.url)
  const code = url.searchParams.get("code")
  const error = url.searchParams.get("error")
  
  // Handle OAuth errors
  if (error) {
    return redirectToApp(`error=${error}`)
  }
  
  if (!code) {
    return redirectToApp("error=no_code")
  }
  
  try {
    // 1. Exchange code for access token
    const tokenResponse = await fetch(
      "https://api.instagram.com/oauth/access_token",
      {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          client_id: instagramAppId,
          client_secret: instagramAppSecret,
          grant_type: "authorization_code",
          redirect_uri: redirectUri,
          code: code,
        }),
      }
    )
    
    const tokenData = await tokenResponse.json()
    
    if (tokenData.error_message) {
      throw new Error(tokenData.error_message)
    }
    
    const accessToken = tokenData.access_token
    const instagramUserId = tokenData.user_id
    
    // 2. Fetch user profile from Instagram Graph API
    const profileResponse = await fetch(
      `https://graph.instagram.com/v21.0/me?fields=user_id,username,account_type&access_token=${accessToken}`
    )
    
    const profile = await profileResponse.json()
    
    if (profile.error) {
      throw new Error(profile.error.message)
    }
    
    // 3. Create or update user in database
    const supabase = createClient(supabaseUrl, supabaseServiceKey)
    
    const { data: user, error: dbError } = await supabase
      .from("users")
      .upsert({
        instagram_user_id: profile.user_id.toString(),
        instagram_username: profile.username,
        account_type: profile.account_type,
        updated_at: new Date().toISOString(),
      }, {
        onConflict: "instagram_user_id",
      })
      .select()
      .single()
    
    if (dbError) throw dbError
    
    // 4. Generate JWT for the app
    const token = await new SignJWT({ 
      sub: user.id,
      instagram_username: profile.username,
    })
      .setProtectedHeader({ alg: "HS256" })
      .setExpirationTime("30d")
      .sign(new TextEncoder().encode(jwtSecret))
    
    // 5. Redirect back to app with token
    return redirectToApp(`token=${token}`)
    
  } catch (err) {
    console.error("Auth error:", err)
    return redirectToApp(`error=${encodeURIComponent(err.message)}`)
  }
})

function redirectToApp(params: string): Response {
  // Redirect to app's custom URL scheme
  const appUrl = `followpulse://auth/callback?${params}`
  
  return new Response(null, {
    status: 302,
    headers: { Location: appUrl },
  })
}
```

### Login View
```swift
// LoginView.swift
import SwiftUI

struct LoginView: View {
    @EnvironmentObject var authManager: AuthManager
    
    var body: some View {
        VStack(spacing: 32) {
            Spacer()
            
            // Logo
            Image(systemName: "person.2.circle.fill")
                .font(.system(size: 80))
                .foregroundColor(.fpPrimary)
            
            // Title
            VStack(spacing: 8) {
                Text("FollowPulse")
                    .font(.largeTitle)
                    .fontWeight(.bold)
                
                Text("Know when someone unfollows you")
                    .font(.body)
                    .foregroundColor(.fpNeutral)
            }
            
            Spacer()
            
            // Login Button
            Button {
                Task {
                    await authManager.signInWithInstagram()
                }
            } label: {
                HStack {
                    Image(systemName: "camera.circle.fill")
                    Text("Continue with Instagram")
                }
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.fpPrimary)
                .foregroundColor(.white)
                .cornerRadius(12)
            }
            .disabled(authManager.isLoading)
            
            // Error
            if let error = authManager.error {
                Text(error)
                    .font(.caption)
                    .foregroundColor(.red)
            }
            
            // Footer
            Text("We only access your public profile")
                .font(.caption)
                .foregroundColor(.fpNeutral)
            
            Button("Privacy Policy") {
                // Open privacy policy
            }
            .font(.caption)
            
            Spacer().frame(height: 32)
        }
        .padding(24)
    }
}
```

### Dashboard View
```swift
// DashboardView.swift
import SwiftUI

struct DashboardView: View {
    @StateObject private var viewModel = DashboardViewModel()
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 24) {
                    // Follower Count Card
                    FollowerCountCard(
                        count: viewModel.followerCount,
                        change: viewModel.todayChange
                    )
                    
                    // Chart (optional)
                    if !viewModel.chartData.isEmpty {
                        FollowerChartView(data: viewModel.chartData)
                            .frame(height: 200)
                    }
                    
                    // Unfollows List
                    UnfollowsSection(unfollows: viewModel.recentUnfollows)
                }
                .padding()
            }
            .refreshable {
                await viewModel.refresh()
            }
            .navigationTitle("FollowPulse")
            .toolbar {
                NavigationLink(destination: SettingsView()) {
                    Image(systemName: "gearshape")
                }
            }
        }
        .task {
            await viewModel.load()
        }
    }
}

// Follower Count Card
struct FollowerCountCard: View {
    let count: Int
    let change: Int
    
    var body: some View {
        VStack(spacing: 8) {
            Text("\(count.formatted())")
                .font(.system(size: 48, weight: .bold))
            
            Text("followers")
                .font(.body)
                .foregroundColor(.fpNeutral)
            
            if change != 0 {
                Text("\(change > 0 ? "+" : "")\(change) today")
                    .font(.subheadline)
                    .foregroundColor(change < 0 ? .fpUnfollow : .fpNeutral)
            }
        }
        .frame(maxWidth: .infinity)
        .padding(24)
        .background(Color.white)
        .cornerRadius(16)
        .shadow(color: .black.opacity(0.05), radius: 10)
    }
}

// Unfollows Section
struct UnfollowsSection: View {
    let unfollows: [Unfollow]
    
    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            Text("Recent Unfollows")
                .font(.headline)
            
            if unfollows.isEmpty {
                EmptyUnfollowsView()
            } else {
                ForEach(unfollows) { unfollow in
                    UnfollowRow(unfollow: unfollow)
                    Divider()
                }
            }
        }
    }
}

// Unfollow Row
struct UnfollowRow: View {
    let unfollow: Unfollow
    
    var body: some View {
        HStack {
            Circle()
                .fill(Color.fpNeutral.opacity(0.2))
                .frame(width: 40, height: 40)
                .overlay(
                    Image(systemName: "person.fill")
                        .foregroundColor(.fpNeutral)
                )
            
            VStack(alignment: .leading) {
                Text("@\(unfollow.username)")
                    .font(.body)
                    .fontWeight(.medium)
                
                Text(unfollow.detectedAt.timeAgoDisplay())
                    .font(.caption)
                    .foregroundColor(.fpNeutral)
            }
            
            Spacer()
            
            Image(systemName: "person.badge.minus")
                .foregroundColor(.fpUnfollow)
        }
    }
}

// Empty State
struct EmptyUnfollowsView: View {
    var body: some View {
        VStack(spacing: 12) {
            Text("✨")
                .font(.system(size: 40))
            
            Text("No unfollows yet")
                .font(.headline)
            
            Text("We'll notify you when someone unfollows")
                .font(.caption)
                .foregroundColor(.fpNeutral)
        }
        .frame(maxWidth: .infinity)
        .padding(32)
    }
}
```

### Models
```swift
// AppUser.swift
import Foundation

struct AppUser: Codable, Identifiable {
    let id: UUID
    let instagramUsername: String
    let instagramUserId: String?
    let accountType: String  // "BUSINESS" or "MEDIA_CREATOR"
    let email: String?
    let plan: String
    let notificationsEnabled: Bool
    let lastCheckedAt: Date?
    
    enum CodingKeys: String, CodingKey {
        case id
        case instagramUsername = "instagram_username"
        case instagramUserId = "instagram_user_id"
        case accountType = "account_type"
        case email
        case plan
        case notificationsEnabled = "notifications_enabled"
        case lastCheckedAt = "last_checked_at"
    }
    
    var isBusinessAccount: Bool {
        accountType == "BUSINESS"
    }
    
    var isCreatorAccount: Bool {
        accountType == "MEDIA_CREATOR"
    }
}

// Unfollow.swift
import Foundation

struct Unfollow: Codable, Identifiable {
    let id: UUID
    let username: String
    let detectedAt: Date
    
    enum CodingKeys: String, CodingKey {
        case id
        case username
        case detectedAt = "detected_at"
    }
}
```

### Color Extension
```swift
// Color+Extension.swift
import SwiftUI

extension Color {
    static let fpPrimary = Color(hex: "6366F1")
    static let fpUnfollow = Color(hex: "EF4444")
    static let fpNeutral = Color(hex: "6B7280")
    static let fpBackground = Color(hex: "F9FAFB")
    static let fpText = Color(hex: "111827")
    
    init(hex: String) {
        let hex = hex.trimmingCharacters(in: CharacterSet.alphanumerics.inverted)
        var int: UInt64 = 0
        Scanner(string: hex).scanHexInt64(&int)
        let a, r, g, b: UInt64
        switch hex.count {
        case 6:
            (a, r, g, b) = (255, int >> 16, int >> 8 & 0xFF, int & 0xFF)
        default:
            (a, r, g, b) = (255, 0, 0, 0)
        }
        self.init(
            .sRGB,
            red: Double(r) / 255,
            green: Double(g) / 255,
            blue: Double(b) / 255,
            opacity: Double(a) / 255
        )
    }
}
```

### Date Extension
```swift
// Date+Extension.swift
import Foundation

extension Date {
    func timeAgoDisplay() -> String {
        let calendar = Calendar.current
        let now = Date()
        let components = calendar.dateComponents(
            [.minute, .hour, .day],
            from: self,
            to: now
        )
        
        if let days = components.day, days > 0 {
            return days == 1 ? "Yesterday" : "\(days) days ago"
        }
        if let hours = components.hour, hours > 0 {
            return "\(hours) hour\(hours == 1 ? "" : "s") ago"
        }
        if let minutes = components.minute, minutes > 0 {
            return "\(minutes) min ago"
        }
        return "Just now"
    }
}
```

### Push Notification Setup
```swift
// NotificationService.swift
import UserNotifications
import UIKit

class NotificationService {
    static let shared = NotificationService()
    
    func requestPermission() async -> Bool {
        do {
            let granted = try await UNUserNotificationCenter.current()
                .requestAuthorization(options: [.alert, .badge, .sound])
            
            if granted {
                await MainActor.run {
                    UIApplication.shared.registerForRemoteNotifications()
                }
            }
            return granted
        } catch {
            return false
        }
    }
}

// In AppDelegate or App struct:
func application(
    _ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
) {
    let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    // Save token to Supabase
    Task {
        try? await SupabaseService.shared.client
            .from("users")
            .update(["device_token": token])
            .eq("auth_user_id", value: currentUserId)
            .execute()
    }
}
```

---

## Supabase Edge Functions (Deno/TypeScript)

### apify-webhook/index.ts
```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

const supabaseUrl = Deno.env.get("SUPABASE_URL")!
const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!

serve(async (req) => {
  try {
    const supabase = createClient(supabaseUrl, supabaseKey)
    
    // Parse Apify webhook payload
    const payload = await req.json()
    const { userId, followerUsernames } = parseApifyResult(payload)
    
    // Get previous snapshot
    const { data: previousSnapshot } = await supabase
      .from("follower_snapshots")
      .select("follower_usernames")
      .eq("user_id", userId)
      .order("captured_at", { ascending: false })
      .limit(1)
      .single()
    
    // Calculate unfollows
    const unfollows = findUnfollows(
      previousSnapshot?.follower_usernames || [],
      followerUsernames
    )
    
    // Store new snapshot
    await supabase.from("follower_snapshots").insert({
      user_id: userId,
      follower_usernames: followerUsernames,
      follower_count: followerUsernames.length
    })
    
    // Store and notify unfollows
    if (unfollows.length > 0) {
      // Insert unfollows
      await supabase.from("unfollows").insert(
        unfollows.map(username => ({
          user_id: userId,
          username,
          notification_sent: false
        }))
      )
      
      // Get user's device token
      const { data: user } = await supabase
        .from("users")
        .select("device_token, notifications_enabled, plan")
        .eq("id", userId)
        .single()
      
      if (user?.notifications_enabled && user?.device_token) {
        await sendPushNotification(user.device_token, unfollows, user.plan)
      }
    }
    
    // Update last_checked_at
    await supabase
      .from("users")
      .update({ last_checked_at: new Date().toISOString() })
      .eq("id", userId)
    
    return new Response(JSON.stringify({ success: true }), {
      headers: { "Content-Type": "application/json" }
    })
    
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" }
    })
  }
})

function parseApifyResult(payload: any) {
  // Extract from Apify webhook format
  // See /docs/Payloads.md for structure
  const items = payload.resource?.defaultDataset?.items || []
  const followerUsernames = items.map((item: any) => item.username)
  const userId = payload.customData?.userId
  
  return { userId, followerUsernames }
}

function findUnfollows(previous: string[], current: string[]): string[] {
  const currentSet = new Set(current)
  return previous.filter(username => !currentSet.has(username))
}

async function sendPushNotification(
  deviceToken: string, 
  unfollows: string[],
  plan: string
) {
  // Premium: individual notifications
  // Free: summary only
  const message = plan === "premium" && unfollows.length === 1
    ? `@${unfollows[0]} unfollowed you`
    : `${unfollows.length} people unfollowed you`
  
  // Call APNs (implementation depends on your push service)
  // See /docs/Payloads.md for APNs payload format
}
```

### trigger-check/index.ts
```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

const supabaseUrl = Deno.env.get("SUPABASE_URL")!
const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
const apifyToken = Deno.env.get("APIFY_TOKEN")!

serve(async (req) => {
  const supabase = createClient(supabaseUrl, supabaseKey)
  
  // Get users due for check
  const { data: users } = await supabase
    .from("users")
    .select("id, instagram_username, check_frequency_hours")
    .eq("notifications_enabled", true)
    .or(`last_checked_at.is.null,last_checked_at.lt.${getThresholdTime()}`)
  
  if (!users || users.length === 0) {
    return new Response(JSON.stringify({ message: "No users to check" }))
  }
  
  // Trigger Apify for each user
  const webhookUrl = `${supabaseUrl}/functions/v1/apify-webhook`
  
  for (const user of users) {
    await triggerApifyActor(user, webhookUrl)
  }
  
  return new Response(JSON.stringify({ 
    triggered: users.length 
  }))
})

function getThresholdTime(): string {
  // Returns ISO timestamp for "now - minimum check interval"
  const threshold = new Date()
  threshold.setHours(threshold.getHours() - 6) // Minimum 6 hours
  return threshold.toISOString()
}

async function triggerApifyActor(user: any, webhookUrl: string) {
  const response = await fetch(
    "https://api.apify.com/v2/acts/afanasenko~instagram-profile-scraper/runs",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${apifyToken}`
      },
      body: JSON.stringify({
        input: {
          operationMode: "analyzeFollowersFollowing",
          username: user.instagram_username,
          analyzeFollowers: true,
          analyzeFollowing: false,
          maxCount: 10000, // Adjust based on plan
          extractEmail: false,
          extractPhoneNumber: false,
          analyzeQuality: false
        },
        webhooks: [{
          eventTypes: ["ACTOR.RUN.SUCCEEDED"],
          requestUrl: webhookUrl,
          payloadTemplate: `{
            "resource": {{resource}},
            "customData": { "userId": "${user.id}" }
          }`
        }]
      })
    }
  )
  
  return response.json()
}
```

---

## SQL Migrations

### 001_initial.sql
```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
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
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  last_checked_at TIMESTAMPTZ
);

-- Note: account_type from Instagram API with Instagram Login
-- Only Professional accounts (BUSINESS, MEDIA_CREATOR) can authenticate
-- Personal accounts will fail OAuth

-- Follower snapshots
CREATE TABLE follower_snapshots (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  follower_usernames TEXT[] NOT NULL,
  follower_count INT NOT NULL,
  captured_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_snapshots_user_time 
  ON follower_snapshots(user_id, captured_at DESC);

-- Unfollows
CREATE TABLE unfollows (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  username TEXT NOT NULL,
  detected_at TIMESTAMPTZ DEFAULT NOW(),
  notification_sent BOOLEAN DEFAULT false
);

CREATE INDEX idx_unfollows_user_time 
  ON unfollows(user_id, detected_at DESC);

-- RLS Policies
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE follower_snapshots ENABLE ROW LEVEL SECURITY;
ALTER TABLE unfollows ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_select_own" ON users
  FOR SELECT USING (auth.uid() = auth_user_id);

CREATE POLICY "users_update_own" ON users
  FOR UPDATE USING (auth.uid() = auth_user_id);

CREATE POLICY "snapshots_select_own" ON follower_snapshots
  FOR SELECT USING (
    user_id IN (SELECT id FROM users WHERE auth_user_id = auth.uid())
  );

CREATE POLICY "unfollows_select_own" ON unfollows
  FOR SELECT USING (
    user_id IN (SELECT id FROM users WHERE auth_user_id = auth.uid())
  );
```

---

*Part of FollowPulse documentation*
