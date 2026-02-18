# ViziPass API Documentation

## Overview

ViziPass provides passwordless authentication using biometric image matching.
Integrations use standard **OIDC (OpenID Connect)** for web applications.

## Authentication Flows

### 1. OIDC Authorization Code Flow (Recommended for Websites)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Website    │     │   ViziPass   │     │  Phone App   │
│ (Waldo's)    │     │   Server     │     │              │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       │ 1. /oauth2/authorize                    │
       │ ──────────────────>│                    │
       │                    │                    │
       │ 2. Show challenge  │                    │
       │ <──────────────────│                    │
       │                    │                    │
       │                    │ 3. Capture image   │
       │                    │ <──────────────────│
       │                    │                    │
       │                    │ 4. SSIM verify     │
       │                    │ ──────────────────>│
       │                    │                    │
       │ 5. Redirect + code │                    │
       │ <──────────────────│                    │
       │                    │                    │
       │ 6. Exchange code   │                    │
       │ ──────────────────>│                    │
       │                    │                    │
       │ 7. Tokens (JWT)    │                    │
       │ <──────────────────│                    │
```

## OIDC Endpoints

### Discovery Document
```
GET /.well-known/openid-configuration
```

Returns:
```json
{
  "issuer": "http://localhost:4000",
  "authorization_endpoint": "http://localhost:4000/oauth2/authorize",
  "token_endpoint": "http://localhost:4000/oauth2/token",
  "userinfo_endpoint": "http://localhost:4000/oauth2/userinfo",
  "jwks_uri": "http://localhost:4000/.well-known/jwks.json",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

### Authorization Endpoint
```
GET /oauth2/authorize
```

Parameters:
| Parameter | Required | Description |
|-----------|----------|-------------|
| response_type | Yes | Must be "code" |
| client_id | Yes | Your registered client ID |
| redirect_uri | Yes | Callback URL (must match registered) |
| scope | Yes | "openid profile email" |
| state | Yes | CSRF protection token |
| nonce | Yes | Replay protection token |
| code_challenge | Recommended | PKCE challenge (SHA256) |
| code_challenge_method | Recommended | "S256" |

### Token Endpoint
```
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded
```

Parameters:
| Parameter | Required | Description |
|-----------|----------|-------------|
| grant_type | Yes | "authorization_code" or "refresh_token" |
| code | Yes* | Authorization code (*for auth_code grant) |
| redirect_uri | Yes* | Must match authorize request |
| client_id | Yes | Your client ID |
| client_secret | Yes | Your client secret |
| code_verifier | If PKCE | Original PKCE verifier |

Response:
```json
{
  "access_token": "eyJ...",
  "id_token": "eyJ...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### UserInfo Endpoint
```
GET /oauth2/userinfo
Authorization: Bearer <access_token>
```

Response:
```json
{
  "sub": "user_123",
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe"
}
```

## ID Token Claims

The ID token is a signed JWT containing:

| Claim | Description |
|-------|-------------|
| sub | Unique user identifier |
| email | User's email address |
| email_verified | Email verification status |
| name | User's display name |
| amr | Authentication methods ["vizipass", "face"] |
| nonce | Echo of the nonce parameter |
| iat | Issued at timestamp |
| exp | Expiration timestamp |
| aud | Client ID (audience) |
| iss | Issuer URL |

## Direct API (Mobile/Device Flow)

For mobile apps or devices that handle their own image capture:

### Create Challenge
```
POST /api/v1/auth/challenge
Content-Type: application/json

{
  "email": "user@example.com"
}
```

Response:
```json
{
  "challenge_id": "chal_abc123",
  "image_data": "<base64 PNG>",
  "expires_at": "2024-01-01T12:05:00Z"
}
```

### Verify Challenge
```
POST /api/v1/auth/verify
Content-Type: application/json

{
  "challenge_id": "chal_abc123",
  "captured_image": "<base64 PNG>"
}
```

Response (success):
```json
{
  "authenticated": true,
  "session_id": "sess_xyz",
  "user_id": "user_123",
  "email": "user@example.com",
  "expires_at": "2024-01-01T20:00:00Z"
}
```

### Dev Testing Endpoint
```
POST /api/v1/auth/admin
Content-Type: application/json

{
  "email": "user@example.com",
  "anchor_image": "<base64>",
  "captured_challenge": "<base64>"
}
```

Response:
```json
{
  "status": "verified",
  "message": "Challenge image matched",
  "challenge_score": 0.85,
  "match_type": "challenge"
}
```

## Security Considerations

1. **Always use HTTPS** in production
2. **Validate state parameter** to prevent CSRF
3. **Validate nonce** in ID token to prevent replay
4. **Use PKCE** for public clients (mobile apps)
5. **Verify JWT signature** using JWKS endpoint
6. **Check token expiration** before trusting claims
7. **Store client_secret securely** (never in frontend code)

## Client Registration

Contact ViziPass to register your application:

1. Provide your redirect URIs
2. Receive client_id and client_secret
3. Configure your application with the credentials

Test credentials (development only):
- client_id: `test_client`
- client_secret: `test_secret_123`
- redirect_uris: `http://localhost:3000/callback`, `http://localhost:8080/callback`
