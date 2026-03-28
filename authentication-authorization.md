# Authentication, Authorization & JWT

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [Authentication Fundamentals](#authentication-fundamentals)
2. [JWT (JSON Web Tokens)](#jwt-json-web-tokens)
3. [OAuth 2.0 & OpenID Connect](#oauth-20--openid-connect)
4. [Session-Based vs Token-Based Auth](#session-based-vs-token-based-auth)
5. [Token Storage & Security](#token-storage--security)
6. [Authorization & RBAC](#authorization--rbac)
7. [Frontend Auth Implementation](#frontend-auth-implementation)
8. [Security Best Practices](#security-best-practices)
9. [SSO & Multi-Factor Authentication](#sso--multi-factor-authentication)

---

## Authentication Fundamentals

### What is the difference between authentication and authorization?

| Concept | Authentication (AuthN) | Authorization (AuthZ) |
|---|---|---|
| **Question** | "Who are you?" | "What can you do?" |
| **Purpose** | Verify identity | Verify permissions |
| **When** | At login | On every request |
| **Method** | Credentials, tokens, biometrics | Roles, policies, claims |
| **HTTP code** | 401 Unauthorized | 403 Forbidden |

```
User Login (Authentication) → Token Issued → API Request + Token → Check Permissions (Authorization) → Allow/Deny
```

---

### What are common authentication methods?

| Method | How It Works | Best For |
|---|---|---|
| **Username/Password** | Credentials sent to server | Traditional apps |
| **OAuth 2.0** | Delegated auth via third party | "Login with Google" |
| **SAML** | XML-based SSO | Enterprise SSO |
| **Passwordless** | Magic links, OTP | Modern consumer apps |
| **Biometric** | Fingerprint, face | Mobile, WebAuthn |
| **API Keys** | Static key in header | Server-to-server |
| **Certificate-based** | mTLS client certs | High-security systems |

---

### What is the difference between stateful and stateless authentication?

| Feature | Stateful (Sessions) | Stateless (Tokens) |
|---|---|---|
| Server storage | Session store (DB/Redis) | None |
| Scalability | Harder (sticky sessions or shared store) | Easy (any server can validate) |
| Revocation | Easy (delete session) | Hard (need blocklist) |
| Size | Small session ID | Larger (JWT payload) |
| Mobile-friendly | Less (cookies required) | More (token in header) |

---

## JWT (JSON Web Tokens)

### What is a JWT and how is it structured?

JWT is an open standard (RFC 7519) for securely transmitting claims between parties as a JSON object.

**Structure:** `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.     ← Header (Base64)
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTYyMzkwMjJ9.  ← Payload (Base64)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature
```

```js
// Header
{
  "alg": "HS256",   // signing algorithm
  "typ": "JWT"
}

// Payload (claims)
{
  "sub": "user123",      // subject (user ID)
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin",
  "iat": 1516239022,     // issued at
  "exp": 1516242622,     // expiration
  "iss": "myapp.com"     // issuer
}

// Signature
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

**Important:** JWT payload is **Base64-encoded, NOT encrypted**. Anyone can read it. Never put sensitive data in the payload.

---

### What are JWT claim types?

| Claim Type | Description | Examples |
|---|---|---|
| **Registered** | Predefined by spec | `iss`, `sub`, `aud`, `exp`, `iat`, `nbf`, `jti` |
| **Public** | Defined in IANA registry | `name`, `email`, `role` |
| **Private** | Custom claims | `org_id`, `permissions` |

```js
// Common registered claims
{
  "iss": "auth.myapp.com",    // issuer
  "sub": "user123",           // subject
  "aud": "api.myapp.com",     // audience
  "exp": 1700000000,          // expiration (Unix timestamp)
  "iat": 1699996400,          // issued at
  "nbf": 1699996400,          // not before
  "jti": "unique-token-id"    // JWT ID (for revocation)
}
```

---

### What is the difference between HS256 and RS256?

| Algorithm | Type | Key | Use Case |
|---|---|---|---|
| **HS256** | Symmetric | Shared secret | Single service |
| **RS256** | Asymmetric | Private/public key pair | Distributed services, public verification |
| **ES256** | Asymmetric (ECDSA) | Smaller keys, faster | Mobile, IoT |

```
HS256: Server signs + verifies with SAME secret
RS256: Server signs with PRIVATE key → Anyone verifies with PUBLIC key

Microservices:
Auth Service (private key) → signs JWT
API Service (public key) → verifies JWT (no shared secret needed)
```

---

### How does the JWT authentication flow work?

```
1. User sends credentials
   POST /auth/login { email, password }

2. Server validates → creates JWT
   JWT = sign({ sub: userId, role: "admin" }, secret, { expiresIn: "1h" })

3. Server returns tokens
   { accessToken: "eyJ...", refreshToken: "abc123" }

4. Client stores tokens and sends on every request
   Authorization: Bearer eyJ...

5. Server validates JWT on each request
   verify(token, secret) → { sub: userId, role: "admin" }
```

```js
// Server-side (Express)
const jwt = require("jsonwebtoken");

// Login
app.post("/auth/login", async (req, res) => {
  const user = await validateCredentials(req.body);
  if (!user) return res.status(401).json({ error: "Invalid credentials" });

  const accessToken = jwt.sign(
    { sub: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: "15m" }
  );
  const refreshToken = generateRefreshToken(user.id);

  res.json({ accessToken, refreshToken });
});

// Middleware
function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ error: "No token" });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (err) {
    return res.status(401).json({ error: "Invalid token" });
  }
}
```

---

### What is the refresh token pattern?

```
Access Token: short-lived (15 min) — used for API requests
Refresh Token: long-lived (7 days) — used to get new access tokens
```

```js
// Client-side refresh logic
async function refreshAccessToken() {
  const response = await fetch("/auth/refresh", {
    method: "POST",
    credentials: "include", // send refresh token cookie
  });

  if (!response.ok) {
    logout(); // refresh token expired
    return null;
  }

  const { accessToken } = await response.json();
  setAccessToken(accessToken);
  return accessToken;
}

// Axios interceptor for automatic refresh
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const token = await refreshAccessToken();
      if (token) {
        originalRequest.headers.Authorization = `Bearer ${token}`;
        return api(originalRequest);
      }
    }
    return Promise.reject(error);
  }
);
```

---

### What are JWT best practices and common pitfalls?

**Best practices:**
1. Keep access tokens **short-lived** (5–15 minutes)
2. Use **refresh tokens** for session continuity
3. Store access tokens in **memory**, refresh tokens in **httpOnly cookies**
4. **Validate all claims** on the server (exp, iss, aud)
5. Use **RS256** for distributed systems
6. Include minimal data in payload
7. Implement **token rotation** for refresh tokens

**Common pitfalls:**
- Storing JWTs in localStorage (XSS vulnerable)
- Not validating expiration
- Putting sensitive data in payload (it's not encrypted)
- Using weak secrets for HS256
- No revocation strategy

---

## OAuth 2.0 & OpenID Connect

### What is OAuth 2.0?

OAuth 2.0 is an **authorization framework** that allows third-party apps to access resources on behalf of a user.

**Roles:**
| Role | Description | Example |
|---|---|---|
| **Resource Owner** | The user | End user |
| **Client** | The app requesting access | Your web app |
| **Authorization Server** | Issues tokens | Google Auth |
| **Resource Server** | Hosts protected resources | Google API |

---

### What is the Authorization Code Flow?

The most secure OAuth flow for web apps with a backend.

```
1. User clicks "Login with Google"
2. App redirects to Google's auth page
   → GET /authorize?response_type=code&client_id=XXX&redirect_uri=YYY&scope=openid+email

3. User grants permission
4. Google redirects back with authorization code
   → GET /callback?code=AUTH_CODE

5. Server exchanges code for tokens (server-to-server)
   → POST /token { code, client_id, client_secret, redirect_uri }

6. Google returns access_token + id_token + refresh_token
```

---

### What is the Authorization Code Flow with PKCE?

PKCE (Proof Key for Code Exchange) secures the flow for SPAs and mobile apps (no client secret).

```js
// 1. Generate code verifier + challenge
const codeVerifier = generateRandomString(128);
const codeChallenge = base64url(sha256(codeVerifier));

// 2. Authorization request
const authUrl = new URL("https://auth.example.com/authorize");
authUrl.searchParams.set("response_type", "code");
authUrl.searchParams.set("client_id", CLIENT_ID);
authUrl.searchParams.set("redirect_uri", REDIRECT_URI);
authUrl.searchParams.set("code_challenge", codeChallenge);
authUrl.searchParams.set("code_challenge_method", "S256");
authUrl.searchParams.set("scope", "openid profile email");
window.location.href = authUrl;

// 3. Exchange code for tokens (include code_verifier)
const tokenResponse = await fetch("https://auth.example.com/token", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({
    grant_type: "authorization_code",
    code: authCode,
    redirect_uri: REDIRECT_URI,
    client_id: CLIENT_ID,
    code_verifier: codeVerifier, // proves we initiated the flow
  }),
});
```

---

### What is OpenID Connect (OIDC)?

OIDC is an **identity layer** on top of OAuth 2.0. While OAuth handles authorization, OIDC adds authentication.

| Feature | OAuth 2.0 | OIDC |
|---|---|---|
| Purpose | Authorization | Authentication + Authorization |
| Token | Access token | Access token + **ID token** |
| User info | Via API call | In ID token (JWT) |
| Scope | Custom | `openid`, `profile`, `email` |

```js
// ID token payload (OIDC)
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",
  "aud": "your-client-id.apps.googleusercontent.com",
  "email": "user@example.com",
  "name": "John Doe",
  "picture": "https://lh3.googleusercontent.com/...",
  "iat": 1699996400,
  "exp": 1700000000
}
```

---

## Session-Based vs Token-Based Auth

### How does session-based authentication work?

```
1. User logs in → Server creates session → stores in DB/Redis
2. Server sends session ID in cookie
3. Every request includes cookie automatically
4. Server looks up session by ID → finds user
```

```js
// Express session
const session = require("express-session");
const RedisStore = require("connect-redis").default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  },
}));
```

---

### When should you use sessions vs JWTs?

| Criteria | Sessions | JWTs |
|---|---|---|
| **Server scaling** | Need shared session store | Stateless — any server works |
| **Revocation** | Easy (delete from store) | Hard (need blocklist) |
| **Mobile apps** | Harder (cookie-based) | Easier (header-based) |
| **Microservices** | Complex (shared store) | Natural fit |
| **Bandwidth** | Small cookie | Larger token on every request |
| **Security** | Server-side control | Client-side risk (storage) |

---

## Token Storage & Security

### Where should you store tokens in a browser?

| Storage | XSS Safe | CSRF Safe | Recommendation |
|---|---|---|---|
| **localStorage** | ❌ | ✅ | Avoid for tokens |
| **sessionStorage** | ❌ | ✅ | Slightly better (tab-scoped) |
| **httpOnly cookie** | ✅ | ❌ (needs CSRF protection) | **Best for refresh tokens** |
| **In-memory (variable)** | ✅ | ✅ | **Best for access tokens** |

```jsx
// Recommended pattern
// Access token: in-memory variable
let accessToken = null;

function setAccessToken(token) {
  accessToken = token;
}

function getAccessToken() {
  return accessToken;
}

// Refresh token: httpOnly, secure, sameSite cookie
// Set by server:
// Set-Cookie: refreshToken=abc123; HttpOnly; Secure; SameSite=Strict; Path=/auth/refresh
```

---

### How do you protect against XSS in auth?

```jsx
// 1. Never store tokens in localStorage
// ❌ localStorage.setItem("token", jwt)

// 2. Sanitize all user input
import DOMPurify from "dompurify";
const clean = DOMPurify.sanitize(userInput);

// 3. Use Content Security Policy
// <meta http-equiv="Content-Security-Policy"
//   content="default-src 'self'; script-src 'self'">

// 4. Avoid dangerouslySetInnerHTML
// ❌ <div dangerouslySetInnerHTML={{ __html: userContent }} />

// 5. Use httpOnly cookies for refresh tokens
```

---

### How do you protect against CSRF?

```js
// 1. SameSite cookies
// Set-Cookie: session=abc; SameSite=Strict; Secure; HttpOnly

// 2. CSRF token pattern
// Server generates token, client sends it in header
app.use((req, res, next) => {
  if (req.method !== "GET") {
    const csrfToken = req.headers["x-csrf-token"];
    if (csrfToken !== req.session.csrfToken) {
      return res.status(403).json({ error: "Invalid CSRF token" });
    }
  }
  next();
});

// 3. Double-submit cookie pattern
// Set token in both cookie and header — server compares them
```

---

## Authorization & RBAC

### What is Role-Based Access Control (RBAC)?

```jsx
// Define roles and permissions
const PERMISSIONS = {
  admin: ["read", "write", "delete", "manage_users"],
  editor: ["read", "write"],
  viewer: ["read"],
};

// Permission check hook
function usePermission(permission) {
  const { user } = useAuth();
  return PERMISSIONS[user.role]?.includes(permission) ?? false;
}

// Usage in component
function ArticleActions({ article }) {
  const canEdit = usePermission("write");
  const canDelete = usePermission("delete");

  return (
    <div>
      {canEdit && <button>Edit</button>}
      {canDelete && <button>Delete</button>}
    </div>
  );
}

// Route guard
function ProtectedRoute({ children, requiredPermission }) {
  const hasPermission = usePermission(requiredPermission);
  if (!hasPermission) return <Navigate to="/unauthorized" />;
  return children;
}
```

---

### What is Attribute-Based Access Control (ABAC)?

ABAC uses attributes (user, resource, environment) for fine-grained control.

```js
// ABAC policy engine
function checkAccess({ user, resource, action, environment }) {
  const policies = [
    // Only document owner can delete
    {
      effect: "allow",
      condition: () =>
        action === "delete" && resource.ownerId === user.id,
    },
    // Admins can do anything
    {
      effect: "allow",
      condition: () => user.role === "admin",
    },
    // Only allow writes during business hours
    {
      effect: "deny",
      condition: () => {
        const hour = new Date().getHours();
        return action === "write" && (hour < 9 || hour > 17);
      },
    },
  ];

  return policies.some(
    (p) => p.effect === "allow" && p.condition()
  );
}
```

---

## Frontend Auth Implementation

### How do you implement auth context in React?

```jsx
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check for existing session on mount
    checkAuth()
      .then(setUser)
      .catch(() => setUser(null))
      .finally(() => setLoading(false));
  }, []);

  const login = async (credentials) => {
    const response = await fetch("/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(credentials),
      credentials: "include",
    });
    if (!response.ok) throw new Error("Login failed");
    const data = await response.json();
    setAccessToken(data.accessToken);
    setUser(data.user);
  };

  const logout = async () => {
    await fetch("/auth/logout", { method: "POST", credentials: "include" });
    setAccessToken(null);
    setUser(null);
  };

  if (loading) return <FullPageSpinner />;

  return (
    <AuthContext.Provider value={{ user, login, logout, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error("useAuth must be used within AuthProvider");
  return context;
}
```

---

### How do you implement an authenticated API client?

```js
// api.js — Axios with interceptors
import axios from "axios";

const api = axios.create({
  baseURL: "/api",
  withCredentials: true,
});

// Request interceptor — attach access token
api.interceptors.request.use((config) => {
  const token = getAccessToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor — handle token refresh
let isRefreshing = false;
let failedQueue = [];

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then((token) => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return api(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const { data } = await axios.post("/auth/refresh", null, {
          withCredentials: true,
        });
        setAccessToken(data.accessToken);
        failedQueue.forEach(({ resolve }) => resolve(data.accessToken));
        failedQueue = [];
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return api(originalRequest);
      } catch (err) {
        failedQueue.forEach(({ reject }) => reject(err));
        failedQueue = [];
        logout();
        return Promise.reject(err);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

export default api;
```

---

## Security Best Practices

### What is the OWASP Top 10 for frontend?

| Vulnerability | Frontend Impact | Prevention |
|---|---|---|
| **XSS** | Script injection | Sanitize input, CSP, avoid `innerHTML` |
| **CSRF** | Forged requests | SameSite cookies, CSRF tokens |
| **Broken Auth** | Token theft | httpOnly cookies, short-lived tokens |
| **Injection** | SQL/NoSQL injection | Parameterized queries, input validation |
| **Sensitive Data Exposure** | Tokens in URL/logs | No tokens in URLs, secure storage |
| **Security Misconfiguration** | Open CORS, debug mode | Strict CORS, remove debug |
| **Insecure Deserialization** | Tampered tokens | Verify JWT signatures |

---

### What security headers should every app set?

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## SSO & Multi-Factor Authentication

### What is Single Sign-On (SSO)?

SSO allows users to authenticate once and access multiple applications.

```
User → App A (not logged in) → Redirects to IdP
  → User authenticates with IdP
  → IdP redirects back to App A with token
  → User accesses App A

User → App B → Redirects to IdP → Already authenticated!
  → IdP redirects back with token immediately
  → User accesses App B (no login required)
```

**Protocols:** SAML 2.0 (enterprise), OIDC (modern), CAS.

---

### How does Multi-Factor Authentication work?

| Factor | Type | Example |
|---|---|---|
| **Something you know** | Knowledge | Password, PIN |
| **Something you have** | Possession | Phone (OTP), hardware key |
| **Something you are** | Inherence | Fingerprint, face |

```jsx
// TOTP (Time-based One-Time Password) flow
function MfaForm({ onVerify }) {
  const [code, setCode] = useState("");

  const handleSubmit = async (e) => {
    e.preventDefault();
    const response = await fetch("/auth/verify-mfa", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ code }),
    });
    if (response.ok) {
      const { accessToken } = await response.json();
      onVerify(accessToken);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>Enter 6-digit code from your authenticator app</label>
      <input
        value={code}
        onChange={(e) => setCode(e.target.value)}
        maxLength={6}
        pattern="[0-9]{6}"
        inputMode="numeric"
        autoComplete="one-time-code"
      />
      <button type="submit">Verify</button>
    </form>
  );
}
```

---

### What is WebAuthn / Passkeys?

WebAuthn enables **passwordless** authentication using biometrics or hardware keys.

```js
// Registration (create credential)
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: new Uint8Array(serverChallenge),
    rp: { name: "My App", id: "myapp.com" },
    user: {
      id: new Uint8Array(userId),
      name: "user@example.com",
      displayName: "John Doe",
    },
    pubKeyCredParams: [
      { alg: -7, type: "public-key" },  // ES256
      { alg: -257, type: "public-key" }, // RS256
    ],
    authenticatorSelection: {
      authenticatorAttachment: "platform", // built-in (Touch ID, Windows Hello)
      userVerification: "required",
    },
  },
});

// Authentication (get credential)
const assertion = await navigator.credentials.get({
  publicKey: {
    challenge: new Uint8Array(serverChallenge),
    rpId: "myapp.com",
    allowCredentials: [{
      id: new Uint8Array(credentialId),
      type: "public-key",
    }],
    userVerification: "required",
  },
});
```

---

[← Back to README](./README.md)
