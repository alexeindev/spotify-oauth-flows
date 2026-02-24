# Spotify API — OAuth 2.0 Flows Postman Collection

> Postman collection covering the three main OAuth 2.0 grant types — Client Credentials, Authorization Code, and Authorization Code with PKCE — using the Spotify Web API.

---

## 📋 Project Overview

This collection demonstrates the three OAuth 2.0 grant types supported by the Spotify Web API, implemented progressively from least to most complex. Each flow is organized as its own folder with dedicated environment variables, token management scripts, and test assertions.

---

## 🔑 The Three Flows — At a Glance

| Flow | Use Case | Requires User Login | client_secret Exposed |
|---|---|---|---|
| Client Credentials | App-to-app, public data | No | Yes (server-side only) |
| Authorization Code | User private data, server-side apps | Yes | Yes (server-side only) |
| Authorization Code + PKCE | User private data, frontend/mobile apps | Yes | No |

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| Postman | Collection design and manual execution |
| Newman | Command-line execution |
| newman-reporter-htmlextra | HTML report generation |
| Node.js v18+ | Required by Newman |
| Spotify Web API | API under test |

---

## 📁 Collection Structure

```
Spotify OAuth 2.0 Flows
│
├── 01 - Client Credentials Flow
│   ├── POST Get Token
│   ├── GET Search Artist
│   └── GET Get Album
│
├── 02 - Authorization Code Flow
│   ├── Step 1 - Get Authorization Code
│   ├── Step 2 - Exchange Code for Tokens
│   ├── Step 3 - Refresh Token
│   ├── GET /me
│   ├── POST Create Playlist
│   ├── GET Playlist by ID
│   ├── PUT Update Playlist
│   └── DELETE Playlist
│
└── 03 - Authorization Code + PKCE
    ├── Step 1 - Generate Code Verifier & Challenge
    ├── Step 2 - Get Authorization Code (PKCE)
    ├── Step 3 - Exchange Code for Tokens (no client_secret)
    └── GET /me (PKCE-authenticated)
```

---

## ⚙️ Environment Variables

### Shared (all flows)
| Variable | Description | Set by... |
|---|---|---|
| `client_id` | Your app's Client ID | Manual (initial setup) |
| `redirect_uri` | `https://oauth.pstmn.io/v1/callback` | Manual (initial setup) |

### Flow 01 — Client Credentials
| Variable | Description | Set by... |
|---|---|---|
| `client_secret` | Your app's Client Secret | Manual (initial setup) |
| `cc_access_token` | App-level access token | Automatically via Test script |
| `cc_token_expiry` | Expiry timestamp | Automatically via Test script |

### Flow 02 — Authorization Code
| Variable | Description | Set by... |
|---|---|---|
| `client_secret` | Your app's Client Secret | Manual (initial setup) |
| `ac_access_token` | User access token | Automatically via Test script |
| `ac_refresh_token` | Refresh token | Automatically via Test script |
| `ac_token_expiry` | Expiry timestamp | Automatically via Test script |
| `user_id` | Authenticated user's ID | Automatically via Test script |
| `created_playlist_id` | ID of the playlist created in tests | Automatically via Test script |

### Flow 03 — Authorization Code + PKCE
| Variable | Description | Set by... |
|---|---|---|
| `pkce_code_verifier` | Random string (43–128 chars) | Automatically via Pre-request Script |
| `pkce_code_challenge` | SHA-256 hash of verifier, Base64url encoded | Automatically via Pre-request Script |
| `pkce_access_token` | User access token (no client_secret used) | Automatically via Test script |
| `pkce_refresh_token` | Refresh token | Automatically via Test script |

> ⚠️ Never commit an environment file with real credentials. The `spotify_environment.json` in the repo must have all secret values empty.

---

## 🚀 Setup

### 1. Prerequisites
- A Spotify account (free tier works)
- Node.js v18+
- Postman installed

### 2. Register a Spotify App
1. Go to [developer.spotify.com/dashboard](https://developer.spotify.com/dashboard)
2. Create a new app
3. Under *Redirect URIs*, add: `https://oauth.pstmn.io/v1/callback`
4. Copy your `Client ID` and `Client Secret`

### 3. Install Newman
```bash
npm install -g newman newman-reporter-htmlextra
```

### 4. Import into Postman
1. Import `spotify_collection.json`
2. Import `spotify_environment.json`
3. Fill in `client_id`, `client_secret`, and `redirect_uri`

### 5. First Authorization per Flow

**Flow 01 — Client Credentials:** No browser interaction needed. Run `POST Get Token` and you're ready.

**Flow 02 — Authorization Code (manual, one time only):**
1. Run *Step 1 - Get Authorization Code* — the Spotify login page opens in the browser
2. Authorize the app — Postman captures the callback automatically
3. Run *Step 2 - Exchange Code for Tokens*
4. The `ac_refresh_token` is saved — from this point the token refresh is fully automatic

**Flow 03 — Authorization Code + PKCE (manual, one time only):**
1. Run *Step 1 - Generate Code Verifier & Challenge* — values are generated and stored automatically
2. Run *Step 2 - Get Authorization Code (PKCE)* — authorize in the browser
3. Run *Step 3 - Exchange Code for Tokens* — no `client_secret` is sent in this request

---

## ▶️ Run with Newman

Run all flows:
```bash
newman run spotify_collection.json \
  -e spotify_environment.json \
  -r htmlextra \
  --reporter-htmlextra-export ./reports/report.html
```

Run a single folder:
```bash
newman run spotify_collection.json \
  -e spotify_environment.json \
  --folder "01 - Client Credentials Flow" \
  -r htmlextra \
  --reporter-htmlextra-export ./reports/report-cc.html
```

Reports are saved to the `/reports` folder. Open the `.html` file in any browser to view results.

---

## 📚 References

- [Spotify — Which OAuth Flow to Use](https://developer.spotify.com/documentation/web-api/concepts/authorization)
- [Spotify — Client Credentials Flow](https://developer.spotify.com/documentation/web-api/tutorials/client-credentials-flow)
- [Spotify — Authorization Code Flow](https://developer.spotify.com/documentation/web-api/tutorials/code-flow)
- [Spotify — Authorization Code + PKCE Flow](https://developer.spotify.com/documentation/web-api/tutorials/code-pkce-flow)
- [Spotify — Refreshing Tokens](https://developer.spotify.com/documentation/web-api/tutorials/refreshing-tokens)
- [newman-reporter-htmlextra](https://github.com/DannyDainton/newman-reporter-htmlextra)
