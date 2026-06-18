## Authentication Naming / Roles

---

### The core roles

| What you called it | Standard name | Also called |
|---|---|---|
| **Okta / Google** | **Identity Provider (IdP)** | Authorization Server, OP (OpenID Provider in OIDC), Issuer |
| **Your backend** | **Resource Server** | API Server, Protected Resource |
| **Your frontend** | **Client** | Relying Party (in OIDC), Service Provider / SP (in SAML) |
| **The user** | **Resource Owner** | End User, Subject (`sub` in JWT) |

---

### How they relate in a typical flow

```
User (Resource Owner)
  │
  │  "I want to log in"
  ▼
Frontend (Client / Relying Party)
  │
  │  "Go authenticate with Okta"  →  redirect
  ▼
Okta/Google (Identity Provider / Authorization Server)
  │
  │  issues tokens (ID token + Access token)
  ▼
Frontend receives tokens, sends Access token to...
  ▼
Backend (Resource Server)
  │
  │  validates token against IdP's public keys (JWKS)
  ▼
  Returns protected data
```

---

### The token vocabulary

| Token | Issued by | Used by | Purpose |
|---|---|---|---|
| **ID Token** (JWT) | IdP | Frontend | Proves *who* the user is (authentication) |
| **Access Token** | IdP | Backend | Proves the user is *authorized* to call the API |
| **Refresh Token** | IdP | Frontend/Backend | Get new Access Tokens without re-login |

---

### The concept layer

| Term | Meaning |
|---|---|
| **SSO** | User logs in once, accesses many apps — a *goal*, not a protocol |
| **OAuth 2.0** | The *authorization* framework (access tokens, scopes) |
| **OIDC** | The *authentication* layer on top of OAuth 2.0 (adds ID token, `/userinfo`) |
| **SAML** | Alternative all-in-one auth + SSO protocol (enterprise) |
| **JWKS** | Public keys the IdP publishes so anyone can verify its tokens |
| **Tenant** | Your isolated org/workspace inside a multi-tenant IdP like Okta |

---

### What Okta vs Google actually are

Both are IdPs, but with different flavors:

- **Google** — consumer-facing IdP ("Sign in with Google"), also enterprise via Google Workspace
- **Okta** — enterprise IdP, acts as a *hub* that can itself federate to Google, Active Directory, etc. — often called an **Identity Broker**

So the full picture can be:

```
User → Your App → Okta (broker) → Google / AD / LDAP (upstream IdP)
```

Okta abstracts away *where* the identity actually lives.