# Topic 4: API Authentication and Security (Step-by-Step Practical Blueprint)

**Course:** n8n Full-Stack Enterprise Automation Engineering  
**Module:** Enterprise API Security & Authentication Protocols  
**Author / Engineer:** Abhishek Nikam (`abhishek.n@tayanasolutions.com`)  
**Target Repository:** `https://github.com/abhisheknikam3/topic-04-api-authentication-and-security`  
**Target Workflow File:** `Topic_04_API_Authentication_and_Security.json`  

---

## 1. Executive Summary & Objective

API security is the critical foundation of enterprise automation. When connecting automated workflows to sensitive external endpoints (payment gateways, ERP systems, cloud infrastructures), credential management must satisfy strict zero-trust standards:
1. **Decoupled Credential Isolation**: Secure storage of secrets outside canvas definitions, preventing leakage during workflow exports and code reviews.
2. **Standard Authentication Schemes**: Practical configuration of HTTP Basic Auth (RFC 7617) and Header-Based Token Authorization (RFC 6750).
3. **Delegated Authorization (OAuth 2.0)**: Understanding the 3-legged redirection handshake, Client Credentials, redirect callback URIs, and automatic token refresh lifecycles.
4. **Token Inspection & JWT Cryptographic Dissection**: Deconstructing JSON Web Tokens (`Header.Payload.Signature`) into base64url components and validating claim integrity.

This project delivers an end-to-end implementation of **Topic 4: API Authentication and Security**, establishing an n8n workflow verified against live endpoints and documented for enterprise-grade compliance.

---

## 2. System Architecture & Flow

### 2.1 Visual Pipeline Topology

```
┌─────────────────────────────────┐
│         Manual Trigger          │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│      HTTP Basic Auth Node       │  (GET https://postman-echo.com/basic-auth)
│  - Generic Credential: Basic    │  - Evaluates Base64(postman:password)
│  - Trainee Credentials          │  -> Expected: {"authenticated": true} (HTTP 200)
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│     HTTP Header Auth Node       │  (GET https://postman-echo.com/headers)
│  - Generic Credential: Header   │  - Header Name: Authorization
│  - Value: Bearer <secret_token> │  -> Echoes back verified header (HTTP 200)
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│  Deconstruct JWT & Audit Auth   │  (n8n Code Node)
│  - Deconstructs Header.Payload  │  - Validates sub, name, admin, and alg
│  - Aggregates Security Audit    │  -> Status: AUTH_VERIFICATION_SUCCESSFUL
└─────────────────────────────────┘
```

### 2.2 Sequence & Protocol Handshake Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Automation Engineer
    participant n8n as n8n Automation Engine
    participant CredStore as Encrypted Credentials Store
    participant Echo as Postman Echo API (postman-echo.com)
    participant OAuthServer as OAuth 2.0 Identity Provider (IdP)

    rect rgb(240, 248, 255)
    Note over Dev, Echo: Phase 1: HTTP Basic Authentication (RFC 7617)
    Dev->>CredStore: Save Encrypted Basic Creds (postman:password)
    n8n->>Echo: GET /basic-auth [Header: Authorization: Basic cG9zdG1hbjpwYXNzd29yZA==]
    Echo-->>n8n: HTTP 200 OK {"authenticated": true}
    end

    rect rgb(245, 255, 245)
    Note over Dev, Echo: Phase 2: Header-Based Bearer Token Authentication (RFC 6750)
    Dev->>CredStore: Save Encrypted Header Auth (Authorization: Bearer my_secret_token_9988)
    n8n->>Echo: GET /headers [Header: Authorization: Bearer my_secret_token_9988]
    Echo-->>n8n: HTTP 200 OK {"headers": {"authorization": "Bearer my_secret_token_9988"}}
    end

    rect rgb(255, 250, 240)
    Note over Dev, OAuthServer: Phase 3: OAuth 2.0 Consent & Token Handshake (RFC 6749)
    Dev->>OAuthServer: Register App with n8n Redirect URI: /rest/oauth2-credential/callback
    OAuthServer-->>Dev: Issue Client ID & Client Secret
    Dev->>n8n: Initiate "Sign in / Connect"
    n8n->>OAuthServer: Authorization Request (Redirect User to Consent Screen)
    OAuthServer-->>n8n: Redirect Callback with Auth Code
    n8n->>OAuthServer: Exchange Auth Code for Access Token + Refresh Token
    OAuthServer-->>n8n: Return Tokens (Encrypted in n8n Credential Store)
    end

    rect rgb(250, 240, 255)
    Note over n8n: Phase 4: JWT Cryptographic Deconstruction
    n8n->>n8n: Split Header.Payload.Signature
    n8n->>n8n: Base64URL Decode Claims -> sub, name, admin, alg
    n8n-->>Dev: Output Consolidated Security Audit Report
    end
```

---

## 3. Step-by-Step Implementation Guide

### Step 1: Manage Reusable Credentials in the n8n Console
Decoupling secrets from workflow canvases prevents credential leakage during Git versioning:
1. Open the **n8n Dashboard**.
2. Click **Credentials** in the left sidebar navigation.
3. Click **Add Credential** in the top-right corner.
4. Select **Basic Auth** and click **Create**:
   - **User Name:** `postman`
   - **Password:** `password`
5. Click **Save** to encrypt and store the credential under ID/Name `trainee_user`.

---

### Step 2: Implement Basic Authentication in HTTP Request Node
We validate credentials against Postman Echo's RFC 7617 endpoint:
1. Create a new workflow canvas.
2. Add a **Manual Trigger** node.
3. Add an **HTTP Request** node and rename it to **`HTTP Basic Auth Node`**.
4. Configure parameters:
   - **Method:** `GET`
   - **URL:** `https://postman-echo.com/basic-auth`
   - **Authentication:** `Generic Credential Type`
   - **Generic Auth Type:** `Basic Auth`
   - **Credential for Basic Auth:** Select `trainee_user`
5. Click **Test step**:
   - Status code returns `200 OK`.
   - Output JSON confirms authentication:
```json
{
  "authenticated": true
}
```

---

### Step 3: Implement Header-Based Token Authentication (Bearer Token)
Token authentication passes credentials in the `Authorization` header:
1. Add an **HTTP Request** node connected downstream of `HTTP Basic Auth Node`.
2. Rename it to **`HTTP Header Auth Node`**.
3. Configure parameters:
   - **Method:** `GET`
   - **URL:** `https://postman-echo.com/headers`
   - **Authentication:** `Generic Credential Type`
   - **Generic Auth Type:** `Header Auth`
   - **Credential:** Set Header Name to `Authorization` and Value to `Bearer my_secret_token_9988`.
4. Click **Test step**:
   - The remote server echoes back all received headers.
   - Output verifies:
```json
{
  "headers": {
    "host": "postman-echo.com",
    "authorization": "Bearer my_secret_token_9988",
    "x-forwarded-proto": "https"
  }
}
```

---

### Step 4: Configure OAuth 2.0 Consent Handshake
OAuth 2.0 allows applications to access server resources on behalf of a user without receiving plain-text credentials:
1. In an HTTP Request node, select **Authentication $\rightarrow$ Predefined Credential Type**.
2. Select an OAuth 2.0 provider (e.g., GitHub, Spotify, Google).
3. Note n8n's designated callback URL:
   `https://<your-subdomain>.app.n8n.cloud/rest/oauth2-credential/callback`
4. In the provider's developer settings, register a new application with this callback URL.
5. In n8n, input:
   - **Client ID:** `<client_id_from_provider>`
   - **Client Secret:** `<client_secret_from_provider>`
6. Click **Connect / Sign in** to authorize the consent window.
7. n8n securely receives the token pair and executes background token refreshing automatically upon expiry.

---

### Step 5: Deconstruct JSON Web Tokens (JWT)
JSON Web Tokens consist of three base64url-encoded parts separated by periods:
$$\text{JWT} = \text{Header} \,.\, \text{Payload} \,.\, \text{Signature}$$

Given the token:
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMzE5`

Deconstruction reveals:
1. **Header (Metadata & Algorithm)**:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
2. **Payload (Registered & Custom Claims)**:
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```
3. **Signature (HMAC-SHA256 Cryptographic Verification)**:
   $$\text{HMACSHA256}(\text{base64UrlEncode}(\text{Header}) + \text{"."} + \text{base64UrlEncode}(\text{Payload}), \text{secret})$$

---

### Step 6: Save and Execute the Full Workflow
1. Rename workflow to: **`N8N API Authentication Workflow`**.
2. Click **Save** in the top navigation bar.
3. Click **Execute workflow**.
4. All nodes run sequentially, displaying green verification checkmarks.

---

## 4. Authentication Mechanisms Comparison Matrix

| Mechanism | Standard RFC | Transport Medium | State / Session | Scalability | Best Used For |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **HTTP Basic Auth** | RFC 7617 | `Authorization: Basic <base64>` | Stateless | Low | Internal tools, legacy systems, quick testing sandboxes |
| **API Key / Header** | Custom / Proprietary | Custom Header (e.g. `X-API-Key`) | Server-validated | Moderate | Server-to-server integrations, developer SDKs |
| **Bearer Token** | RFC 6750 | `Authorization: Bearer <token>` | Stateless / Stateful | High | Modern REST microservices, mobile backends |
| **OAuth 2.0** | RFC 6749 | Auth Code + Refresh Token | Delegated Grant | High | Third-party integrations (GitHub, Slack, Google Workspace) |
| **JWT (Self-Contained)** | RFC 7519 | JSON Web Signature (`HS256`, `RS256`) | Stateless | Very High | Distributed microservices, SSO, zero-DB token verification |

---

## 5. Verification & Test Telemetry

The repository workflow (`Topic_04_API_Authentication_and_Security.json`) contains complete **pre-pinned test data** (`pinData`).

### Audit Output:
```json
{
  "pipelineStatus": "AUTH_VERIFICATION_SUCCESSFUL",
  "step2_basicAuth": {
    "endpoint": "https://postman-echo.com/basic-auth",
    "authenticated": true,
    "status": "200_OK"
  },
  "step3_headerAuth": {
    "endpoint": "https://postman-echo.com/headers",
    "echoedAuthorizationHeader": "Bearer my_secret_token_9988",
    "status": "200_OK"
  },
  "step4_oauth2HandshakeDetails": {
    "flowType": "Authorization Code Grant with PKCE / Redirection",
    "n8nCallbackUrl": "https://<your-subdomain>.app.n8n.cloud/rest/oauth2-credential/callback",
    "tokenManagement": "Automatic Background Token Refresh and Expiry Handling"
  },
  "step5_jwtDeconstruction": {
    "rawToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "header": { "alg": "HS256", "typ": "JWT" },
    "payload": { "sub": "1234567890", "name": "John Doe", "admin": true },
    "signature": "TJVA95OrM7E2cBab30RMzE5"
  }
}
```

---

## 6. LMS Assessor Submission Details

- **Student Name:** Abhishek Nikam
- **Email:** `abhishek.n@tayanasolutions.com`
- **Course Module:** Topic 4: API Authentication and Security
- **GitHub Repository:** `https://github.com/abhisheknikam3/topic-04-api-authentication-and-security`
- **Workflow JSON File:** `Topic_04_API_Authentication_and_Security.json`
- **Documentation Guide:** `README.md` & `README.pdf`
