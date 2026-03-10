# Authorization Code Flow + PKCE

A step-by-step walkthrough of the OAuth 2.0 Authorization Code Flow with PKCE, designed for **public clients** that cannot safely store a `client_secret` (SPAs, mobile apps, CLI tools).

---

## What is PKCE and why does it exist?

**PKCE** (Proof Key for Code Exchange, pronounced *"pixie"*) is an extension to the Authorization Code Flow that solves a specific problem: if an attacker intercepts the authorization code before the client exchanges it for a token, they can use it themselves — because there is no `client_secret` to authenticate the client.

PKCE mitigates this by **binding the authorization request to the token request** through a cryptographic challenge. The client generates a secret locally, sends a hash of it upfront, and proves ownership of the original secret at exchange time.

---

## PKCE Mechanics — The Code Verifier & Challenge

Before sending the authorization request, the client generates two linked values:

| Value | Description |
|---|---|
| `code_verifier` | A cryptographically random string, 43–128 characters. Generated locally. **Never sent to the authorization server directly.** |
| `code_challenge` | The result of hashing the verifier with SHA-256, then Base64URL-encoding it. This is what gets sent in Step 1. |

```
code_verifier  = base64url( random_bytes(32) )
code_challenge = base64url( SHA256( code_verifier ) )
```


![Code verifier construction](https://github.com/user-attachments/assets/1ed725a7-1e26-4839-9228-afb1850beca6)


> **Key insight:** the authorization server stores the `code_challenge`. When the client later exchanges the code, it sends the raw `code_verifier`. The server re-hashes it and compares — if they match, it proves the requester is the same entity that started the flow.


---

## Step 1 — Generate PKCE Values & Build the Authorization URL

Generate the `code_verifier` and `code_challenge` locally, then redirect the browser to the authorization endpoint with the following query parameters:

| Parameter | Required | Description |
|---|---|---|
| `response_type` | ✅ | Must be `code` |
| `client_id` | ✅ | Your application's client ID |
| `redirect_uri` | ✅ | Where the server redirects after authorization |
| `scope` | ✅ | The permissions being requested |
| `code_challenge` | ✅ | `base64url(SHA256(code_verifier))` |
| `code_challenge_method` | ✅ | Always `S256` — never `plain` in production |
| `state` | ⚠️ | Strongly recommended — random value for CSRF protection |

> **Note:** This is not a regular API call — it is a browser redirect. The `client_secret` is absent by design; PKCE replaces it.

```
GET https://accounts.spotify.com/authorize
  ?response_type=code
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=http://localhost:3000/callback
  &scope=user-read-private%20playlist-read-private
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
  &state=xyzABC123
```

![Code verifier construction](https://github.com/user-attachments/assets/53d336c2-7168-4c8b-95d8-3a15fef1d634)






---

## Step 2 — User Authenticates and Grants Permission

Paste the complete authorization URL in the browser. The user authenticates and grants permission to the client application.

After approval, the authorization server redirects back to the `redirect_uri` with the authorization code and the `state` value:

```
https://localhost:3000/callback
  ?code=AQDfZPixxxxxxxxxxxxxxxxxxxxxx
  &state=xyzABC123
```

![User login screen](https://github.com/user-attachments/assets/ead8081e-6a35-45ab-aa6b-fb7fe99ea9e9)

> ⚠️ **Validate the `state` value before proceeding.** If it doesn't match what you sent in Step 1, abort — this could be a CSRF attack.

---

## Step 3 — Exchange the Authorization Code for an Access Token

Extract the authorization code from the redirect URL and send a `POST` request to the token endpoint. This is where PKCE proves its value — you send the raw `code_verifier` instead of a `client_secret`.

**Headers**

| Header | Value |
|---|---|
| `Content-Type` | `application/x-www-form-urlencoded` |

> No `Authorization` header here. Public clients do not authenticate with a `client_secret`. The `code_verifier` is the proof of identity.

**Body Parameters**

| Parameter | Value |
|---|---|
| `grant_type` | `authorization_code` |
| `code` | The authorization code received in Step 2 |
| `redirect_uri` | Must exactly match the URI used in Step 1 |
| `client_id` | Your application's client ID |
| `code_verifier` | The original random string generated in Step 1 (**not** the hash) |

```
POST https://accounts.spotify.com/api/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AQDfZPixxxxxxxxxxxxxxxxxxxxxx
&redirect_uri=http://localhost:3000/callback
&client_id=YOUR_CLIENT_ID
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```
![Request token](https://github.com/user-attachments/assets/70dea2b7-2dfb-44a8-9bdd-9c17311bd503)


If the request is valid, the authorization server responds with an access token and, typically, a refresh token:

```json
{
  "access_token":  "BQDxxxxxxx...",
  "token_type":    "Bearer",
  "expires_in":    3600,
  "refresh_token": "AQDxxxxxxx...",
  "scope":         "user-read-private playlist-read-private"
}
```

---

## Step 4 — Store the Refresh Token

Store the refresh token securely. It allows the client to obtain a new access token later without prompting the user to authenticate again.

| Client Type | Recommended Storage |
|---|---|
| SPA (browser) | In-memory only — never `localStorage` (XSS risk) |
| Mobile app | Secure OS keychain (Keychain on iOS, Keystore on Android) |
| CLI tool | Encrypted file or OS credential manager |

> ⚠️ If the server uses **refresh token rotation**, the old token is invalidated on use. Store the new one immediately — discarding it means the user must re-authenticate.

![Request token](https://github.com/user-attachments/assets/fc1329a0-16f9-4696-b82f-c7affec18a02)


---

## Step 5 — Refresh the Access Token

When the access token expires, send a new `POST` request to the token endpoint:

**Headers**

| Header | Value |
|---|---|
| `Content-Type` | `application/x-www-form-urlencoded` |

**Body Parameters**

| Parameter | Value |
|---|---|
| `grant_type` | `refresh_token` |
| `refresh_token` | The stored refresh token |
| `client_id` | Your application's client ID |

Public clients do not include a `client_secret` here either. The `client_id` alone identifies the application.

```
POST https://accounts.spotify.com/api/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=AQDxxxxxxx...
&client_id=YOUR_CLIENT_ID
```

If the refresh token is still valid, the authorization server returns a new access token — and in some implementations, a new refresh token as well.

![Refresh token](https://github.com/user-attachments/assets/d2a39aa8-075f-453a-8cd9-4c24b8a4ebdf)

---

## Step 6 — Access the Protected Resource

Use the access token to call the resource server. Include it in the `Authorization` header using the Bearer scheme:

```http
GET /v1/me
Authorization: Bearer BQDxxxxxxx...
Host: api.spotify.com
```

The resource server validates the token and, if it is valid and the required scopes are present, returns the requested resource.

![Protected Resource](https://github.com/user-attachments/assets/a0d4dbdf-764b-41e4-9041-bad7b1175a42)


---

## PKCE vs Standard Authorization Code — Key Differences

| Aspect | Standard Auth Code | Auth Code + PKCE |
|---|---|---|
| Client type | Confidential | Public (SPA, mobile, CLI) |
| Client authenticates with | `client_secret` (Base64 in header) | `code_verifier` (body param, no secret) |
| Extra params in auth URL | None | `code_challenge` + `code_challenge_method` |
| Extra params in token exchange | None | `code_verifier` |
| Protects against | Stolen code (with secret as second factor) | Stolen code (via hash binding) |

---

## Security Notes

- **Always use `S256`** — the `plain` method offers no real protection and should never be used in production.
- **`code_verifier` is single-use** — generate a fresh pair for every authorization flow. Never reuse the same verifier.
- **Validate `state` on return** — a mismatch means the response is forged or replayed.
- **Refresh tokens in SPAs** — if you cannot store the refresh token safely, consider short-lived access tokens and silent re-auth instead.
- **Authorization codes are single-use** — they expire quickly (typically 10 minutes). If you get a `400` on exchange, the code may have already been consumed.
