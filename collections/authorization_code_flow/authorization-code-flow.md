# Authorization Code Flow

A step-by-step walkthrough of the OAuth 2.0 Authorization Code Flow, from building the authorization URL to using the access token against a protected resource.

---

## Step 1 — Build the Authorization URL

Redirect the browser to the authorization endpoint with the following query parameters:

| Parameter | Required | Description |
|---|---|---|
| `response_type` | ✅ | Must be `code` |
| `client_id` | ✅ | Your application's client ID |
| `redirect_uri` | ✅ | Where the server redirects after authorization |
| `scope` | ✅ | The permissions being requested |
| `state` | ⚠️ Recommended | Random value for CSRF protection |

> **Note:** This is not a regular API call — it is a browser redirect that sends the user to the authorization server to authenticate and approve the requested permissions.

![Authorization URL construction](Pasted%20image%2020260306172012.png)

---

## Step 2 — User Authenticates and Grants Permission

Paste the complete authorization URL in the browser. The user authenticates and grants permission to the client application.

![User login screen](Pasted%20image%2020260306172355.png)

After approval, the authorization server redirects the browser back to the `redirect_uri`. The redirect URL contains the authorization code as a query parameter, and the `state` value if one was sent:

```
https://example.com/callback?code=AUTH_CODE&state=xyz
```

---

## Step 3 — Exchange the Authorization Code for an Access Token

Extract the authorization code from the redirect URL and send a `POST` request to the token endpoint:

**Headers**

| Header | Value |
|---|---|
| `Authorization` | `Basic base64(client_id:client_secret)` |
| `Content-Type` | `application/x-www-form-urlencoded` |

**Body Parameters**

| Parameter | Value |
|---|---|
| `grant_type` | `authorization_code` |
| `code` | The authorization code received in Step 2 |
| `redirect_uri` | Must exactly match the URI used in Step 1 |

![Token exchange request](Pasted%20image%2020260306173024.png)
![Token exchange request body](Pasted%20image%2020260306173059.png)

If the request is valid, the authorization server responds with an access token and, typically, a refresh token.

![Token response](Pasted%20image%2020260306173352.png)

---

## Step 4 — Store the Refresh Token

Store the refresh token securely. It allows the client to obtain a new access token later without prompting the user to authenticate again.

![Storing the refresh token](Pasted%20image%2020260306174052.png)

---

## Step 5 — Refresh the Access Token

When the access token expires, send a new `POST` request to the token endpoint:

**Headers**

| Header | Value |
|---|---|
| `Authorization` | `Basic base64(client_id:client_secret)` |
| `Content-Type` | `application/x-www-form-urlencoded` |

**Body Parameters**

| Parameter | Value |
|---|---|
| `grant_type` | `refresh_token` |
| `refresh_token` | The stored refresh token |

> Since this is a **confidential client**, it authenticates itself using its `client_id` and `client_secret` encoded in Base64 in the `Authorization` header.

If the refresh token is still valid, the authorization server returns a new access token — and in some implementations, a new refresh token as well.

![Refresh token request](Pasted%20image%2020260306174726.png)

---

## Step 6 — Access the Protected Resource

Use the access token to call the resource server. Include it in the `Authorization` header using the **Bearer** scheme:

```http
GET /v1/me/playlists
Authorization: Bearer ACCESS_TOKEN
```

The resource server validates the token and, if it is valid and the required scopes are present, returns the requested resource.

![Accessing protected resource](Pasted%20image%2020260306174850.png)
