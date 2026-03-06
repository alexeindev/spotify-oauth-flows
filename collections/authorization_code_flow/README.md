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

![Authorization URL construction](https://github.com/user-attachments/assets/b2e9f0bd-14f1-4673-9fcd-5b285e84290b)

---

## Step 2 — User Authenticates and Grants Permission

Paste the complete authorization URL in the browser. The user authenticates and grants permission to the client application.

![User login screen](https://github.com/user-attachments/assets/ead8081e-6a35-45ab-aa6b-fb7fe99ea9e9)



After approval, the authorization server redirects the browser back to the `redirect_uri`. The redirect URL contains the authorization code as a query parameter, and the `state` value if one was sent:

```
https://example.com/callback?code=AUTH_CODE&state=xyz
```
![Redirect URL Screen](https://github.com/user-attachments/assets/96fd83d3-b221-41cc-acc7-3bd4a8fb396f)



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

![Token exchange request](https://github.com/user-attachments/assets/da863b48-4e5a-4ca3-a6a6-472b159a8253)


![Token exchange request body](https://github.com/user-attachments/assets/5227118c-c426-4be3-a1d0-d3427c521fd3)

If the request is valid, the authorization server responds with an access token and, typically, a refresh token.

![Token response](https://github.com/user-attachments/assets/dc7e1db9-e3da-4fce-b8e6-cacea26d73f5)


---

## Step 4 — Store the Refresh Token

Store the refresh token securely. It allows the client to obtain a new access token later without prompting the user to authenticate again.

![Storing the refresh token](https://github.com/user-attachments/assets/91af3d33-c3b2-4dd1-8b4b-229da9a321bc)


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

![Refresh token request](https://github.com/user-attachments/assets/a3b7df86-c067-400a-9ce5-6fc152acc0e0)

![Refresh token request body](https://github.com/user-attachments/assets/dea95e5a-7746-4166-aae5-71f6992dfc75)

If the refresh token is still valid, the authorization server returns a new access token — and in some implementations, a new refresh token as well.


![Refresh token response](https://github.com/user-attachments/assets/8dc87fa3-a2d2-4e8b-9ca5-c7bf8198dd9f)

---

## Step 6 — Access the Protected Resource

Use the access token to call the resource server. Include it in the `Authorization` header using the **Bearer** scheme:

```http
GET /v1/me/playlists
Authorization: Bearer ACCESS_TOKEN
```

The resource server validates the token and, if it is valid and the required scopes are present, returns the requested resource.

![Accessing protected resource](https://github.com/user-attachments/assets/0c311fa0-64c7-4bdc-a464-75ae2b762122)

