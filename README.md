# 🎵 Spotify OAuth 2.0 Flows — Postman Demo

A Postman collection demonstrating the three OAuth 2.0 grant types supported by the Spotify Web API, with pre-request scripts for automated token management and environment variable handling.

---

## 📌 What This Project Covers

| Flow                      | Client Type  | Token Type       | Requires User |
| ------------------------- | ------------ | ---------------- | ------------- |
| Client Credentials        | Confidential | Access only      | No            |
| Authorization Code        | Confidential | Access + Refresh | Yes           |
| Authorization Code + PKCE | Public       | Access + Refresh | Yes           |

---

## 🗂️ Collection Structure

Each flow is organized into **Positive Scenarios** and **Negative Scenarios**.

```
Flow 01 - Client Credentials
├── Positive Scenarios
│   ├── POST  01 - Get authorization token
│   └── GET   02 - Get Artist Info
└── Negative Scenarios
    ├── POST  01 - Get authorization token - Invalid Credentials
    ├── GET   02 - Get artist info - Invalid Token
    └── GET   03 - Get artist info - Invalid Artist ID

Flow 02 - Authorization Code
├── Positive Scenarios
│   ├── GET   01 - Build Authorization URL (Open URL in Browser)
│   ├── POST  02 - Exchange Authorization Code for Tokens
│   ├── POST  03 - Refresh Token
│   └── GET   04 - Get User Playlists
└── Negative Scenarios
    └── GET   01 - Get User Playlists No token provided

Flow 03 - Authorization Code + PKCE
├── Positive Scenarios
│   ├── GET   01 - Build Authorization URL (Open URL in Browser)
│   ├── POST  02 - Exchange Authorization Code for Tokens
│   ├── POST  03 - Refresh Token
│   └── GET   04 - Get user information
└── Negative Scenarios
    ├── GET   Get users playlists - No token provided
    └── POST  Refresh Token revoked
```

---

## ⚙️ Setup

### Prerequisites

- [Postman](https://www.postman.com/downloads/) v10+
- A registered app in the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard)

### 1. Register Your App on Spotify

1. Go to [Spotify Developer Dashboard](https://developer.spotify.com/dashboard) → **Create App**
2. Set **Redirect URI** to: `https://oauth.pstmn.io/v1/callback`
3. Copy your `Client ID` and `Client Secret`

### 2. Configure the Environment

Import `SpotifyEnv.postman_environment.json` into Postman and fill in your credentials:

| Variable                  | Type    | Description                                                              |
| ------------------------- | ------- | ------------------------------------------------------------------------ |
| `client_id`               | secret  | Your Spotify App Client ID                                               |
| `client_secret`           | secret  | Your Spotify App Client Secret _(Auth Code flow only)_                   |
| `access_token`            | secret  | Populated automatically after token exchange (Auth Code flow)            |
| `refresh_token`           | secret  | Populated automatically after token exchange (Auth Code flow)            |
| `access_token_pkce`       | secret  | Populated automatically after token exchange (PKCE flow)                 |
| `refresh_token_pkce`      | secret  | Populated automatically after token exchange (PKCE flow)                 |
| `authorization_code`      | default | The `code` returned by Spotify after user authorization (Auth Code flow) |
| `authorization_code_pkce` | default | The `code` returned by Spotify after user authorization (PKCE flow)      |

> ⚠️ **Never commit credentials to version control.** Secret-type variables are stored locally in Postman and are not exported. Keep `SpotifyEnv.postman_environment.json` in `.gitignore`.

### 3. Import the Collection

Import the collection file into Postman. All three flows live in a single collection.

---

## 🚀 Running the Flows

### Flow 01 — Client Credentials

No user interaction required.

1. Fill in `client_id` and `client_secret` in the environment
2. Run **01 - Get authorization token** — `access_token` is stored automatically
3. Run **02 - Get Artist Info** to verify

### Flow 02 — Authorization Code

Requires manual browser step to obtain the authorization code.

1. Run **01 - Build Authorization URL** — the pre-request script generates the URL and logs it to the Postman console
2. Open that URL in a browser, log in, and authorize the app
3. Copy the `code` parameter from the redirect URL and paste it into `authorization_code` in the environment
4. Run **02 - Exchange Authorization Code for Tokens** — `access_token` and `refresh_token` are stored automatically
5. Run **03 - Refresh Token** to test token renewal
6. Run **04 - Get User Playlists** to verify the token works

### Flow 03 — Authorization Code + PKCE

Same browser step as Flow 02, but no `client_secret` is used. The PKCE `code_verifier` and `code_challenge` are generated in the pre-request script.

1. Run **01 - Build Authorization URL** — PKCE parameters are generated and the URL is logged to the console
2. Open that URL in a browser, log in, and authorize the app
3. Copy the `code` from the redirect URL and paste it into `authorization_code_pkce` in the environment
4. Run **02 - Exchange Authorization Code for Tokens** — `access_token_pkce` and `refresh_token_pkce` are stored automatically
5. Run **03 - Refresh Token** to test rotation
6. Run **04 - Get user information** to verify

---

## ⚠️ Why These Flows Can't Be Automated with Newman

Newman (Postman's CLI runner) is designed for headless, non-interactive execution. Two of the three flows in this project — **Authorization Code** and **Authorization Code + PKCE** — require a real user to log in to Spotify and grant permissions through a browser. That interactive step happens outside of Postman entirely and cannot be automated headlessly.

The flow breaks down like this:

1. Your app redirects the user to `accounts.spotify.com/authorize`
2. Spotify renders a login/consent page in a browser
3. The user authenticates and approves scopes
4. Spotify redirects back with a one-time `authorization_code` in the URL

Newman has no browser, no way to interact with a login page, and no way to intercept that redirect. You would need to manually grab the `authorization_code` from the redirect URL and paste it into the environment before the token exchange step can run.

**The only flow that is fully automatable with Newman is Client Credentials**, since it is a pure machine-to-machine exchange — no user, no browser, no redirect.

```bash
# Only the Client Credentials collection can run headlessly
newman run collections/client-credentials.postman_collection.json \
  --environment environments/spotify-oauth.postman_environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export reports/results.html
```

For the Authorization Code and PKCE flows, use Postman's collection runner after completing the manual auth step described above.

## 🔄 Token Lifecycle & Security Notes

- **Access tokens** expire after 3600 seconds (1 hour). The collections include pre-request scripts that check expiry and refresh automatically.
- **Refresh token rotation**: Spotify may issue a new refresh token on each use. The collections store the latest token, invalidating the previous one.
- **PKCE protects against authorization code interception** — even if the code is stolen, it's useless without the `code_verifier` that never leaves the client.
- `client_secret` is only used in confidential flows. It must never be exposed in client-side code or committed to a public repository.

---

## 📊 Scopes Used

| Scope               | Purpose                          |
| ------------------- | -------------------------------- |
| `user-read-private` | Read user's subscription details |
| `user-read-email`   | Access user's email address      |

Adjust scopes in the environment variable based on what your use case requires. Full scope list: [Spotify Scopes Reference](https://developer.spotify.com/documentation/web-api/concepts/scopes).

---

## 📚 References

- [Spotify Authorization Guide](https://developer.spotify.com/documentation/web-api/concepts/authorization)
- [RFC 6749 – The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7636 – PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
- [Postman OAuth 2.0 Docs](https://learning.postman.com/docs/sending-requests/authorization/oauth-20/)
