# API Integration Guide

This guide shows how to integrate your application with Stackure for authentication and session validation using the REST API.

Use this approach if you need full control over how your application handles authentication instead of using the Stackure SDK.

The SDKs do everything below for you. See the [SDK overview](sdk.md) if you have not ruled them out.

---

## Step 1. Send a magic sign-in link

Trigger a magic-link email for your user.

```bash
curl -X POST https://stackure.com/api/public/auth/magic-link/send \
  -H "Content-Type: application/json" \
  -d '{
    "user_email": "user@example.com",
    "app_id": "YOUR_APP_ID"
  }'
```

**Response**
```json
{
  "message": "if the email exists, a sign in link will be sent."
}
```

---

## Step 2. Handle the magic-link redirect

After the user clicks the sign-in link, Stackure hands the browser back to your app's registered URL (configured during app registration) carrying a `session_token`.

It arrives one of two ways, and your endpoint must accept both:

**As a POST form field** — an auto-submitting form posts to your app URL with
`Content-Type: application/x-www-form-urlencoded`:

```
session_token=abc123...
```

**As a query parameter:**

```
https://myapp.com?session_token=abc123...
```

Extract the `session_token` and store it in an HTTP-only cookie on your own
domain:

- Name: `session`
- `HttpOnly: true`
- `Secure: true` (when served over HTTPS)
- `SameSite: Lax`
- `Path: /`

Then redirect to the same URL with the parameter stripped, so the token does
not linger in the address bar. The examples below use Bearer token
authentication, but that cookie is also accepted.

---

## Step 3. Validate the session on each protected request

Your backend must validate the `session_token` before serving protected data.

Stackure binds each session to the browser's user agent and IP address. Since
you are validating from your server, you must forward the browser's original
`User-Agent` and `X-Forwarded-For` — otherwise Stackure sees your server and
rejects the session.

**Request**
```bash
curl "https://stackure.com/api/public/auth/session/validate?app_id=YOUR_APP_ID" \
  -H "Authorization: Bearer SESSION_TOKEN" \
  -H "X-App-Secret: YOUR_APP_SECRET" \
  -H "User-Agent: ORIGINAL_BROWSER_USER_AGENT" \
  -H "X-Forwarded-For: ORIGINAL_CLIENT_IP"
```

**Response if the session is valid**
```json
{
  "authenticated": true,
  "user": {
    "user_id": "uuid",
    "account_id": "uuid",
    "user_email": "user@example.com",
    "user_first_name": "John",
    "user_last_name": "Doe",
    "user_is_app_admin": false,
    "user_teams": [{ "team_id": "uuid", "team_name": "Support" }]
  }
}
```

- `user_is_app_admin`: the user is an app admin or owner in their Stackure org, in charge of its apps
- `user_teams`: the Stackure teams the user belongs to in their org, empty when none

Both are present for every authenticated session, MCP included. Stackure
defines no in-app permissions; your app decides what they mean.

**Response if the session is invalid or expired**
```json
{
  "authenticated": false,
  "sign_in_url": "https://stackure.com/sign-in/magic-link?app_id=YOUR_APP_ID"
}
```

---

---

## Step 4. Sign the user out

Route every method on your sign-out path to one handler, and trigger it with a
form or button that POSTs from your own page. Make this call from your server
only for a POST with `Sec-Fetch-Site: same-origin` or, only when that header
is absent, with an `Origin` whose host and port match `Host` and whose scheme
is `https` if the request arrived over HTTPS. A request that repeats
`Sec-Fetch-Site`, `Origin` or `Host` does not qualify.

```bash
curl -X POST https://stackure.com/api/public/auth/sign-out \
  -H "Authorization: Bearer SESSION_TOKEN"
```

This signs the user out everywhere. Any 2xx status is success, whatever the
body. Clear the cookie you set in Step 2 as well. If the call fails, still
clear the cookie and send the browser to `https://stackure.com/signout`, where
the user can finish signing out.

For any other request (a link, a GET, a POST from another site), make no
call, leave the cookie in place, and send the browser to
`https://stackure.com/signout`, where the user confirms the sign-out. Your
cookie is `SameSite=Lax`, so a sign-out route that acts on such a request lets
a link on any other site sign the user out everywhere with no confirmation.

---

## Step 5. Protect your MCP endpoint

Skip this step if your app has no MCP endpoint.

AI clients such as Claude, Claude Code, VS Code and Cursor do not use your
cookie. The user adds your app's MCP address in their AI client, the client is
told to sign in at Stackure, and the user signs in and confirms. From then on
the client sends `Authorization: Bearer <credential>` with every MCP request.
Stackure issues that credential; your endpoint checks it on every request at
the Step 3 endpoint, with an added `mcp` parameter.

**Request**
```bash
curl "https://stackure.com/api/public/auth/session/validate?app_id=YOUR_APP_ID&mcp=https%3A%2F%2Fmyapp.com%2Fmcp" \
  -H "X-App-Secret: YOUR_APP_SECRET" \
  -H "Authorization: Bearer CREDENTIAL"
```

- `mcp` is the public URL of the MCP endpoint the request arrived at: the
  scheme, `://`, the `Host` header and the request path, with no query string
  and no fragment, percent-encoded as a query value
- `X-App-Secret` is your app secret, shown on the app's page in Stackure at
  registration and on each rotation
- Take the credential only from the incoming `Authorization: Bearer` header
  (the scheme is case-insensitive) and send it on as
  `Authorization: Bearer <credential>`. If the request has none, make the
  same call without that header
- Never send a cookie on this call, and never log the credential or the app
  secret

**Response if signed in**

The same body as in Step 3: `"authenticated": true` and the `user` object.
Serve the request.

**Response if not signed in**
```json
{
  "authenticated": false,
  "sign_in_url": "https://stackure.com/sign-in/magic-link?app_id=YOUR_APP_ID",
  "www_authenticate": "Bearer resource_metadata=\"...\""
}
```

Answer `401` with the `www_authenticate` value, unchanged, as the
`WWW-Authenticate` header (send `Bearer` if the field is missing),
`Content-Type: application/json`, and the body `{"error":"unauthorized"}`.
That header tells the AI client to sign in at Stackure. Never redirect, and
never set or clear a cookie.

**Any other status, or a network error**

Answer `503` with the body `{"error":"unavailable"}`. This covers 400, 401
for a wrong app secret, 429, 5xx and no response at all. Never treat the
caller as signed in.

The MCP endpoint must be on the same site as your app's registered URL (the
same host and port), unless an MCP URL is set for the app in Stackure.

Access ends when the user signs out, disconnects the client, loses access to
your app, or leaves the connection unused for 30 days. Users can see and
disconnect their AI clients in Stackure under **AI Clients**.

---

## Step 6. List who can open your app

Optional. For pickers and sharing, list the users and teams in the caller's
org who can open your app. Authenticate as in Step 3, with the session token
from your cookie. MCP credentials are not accepted.

**Request**
```bash
curl "https://stackure.com/api/public/directory?app_id=YOUR_APP_ID" \
  -H "Authorization: Bearer SESSION_TOKEN" \
  -H "X-App-Secret: YOUR_APP_SECRET" \
  -H "User-Agent: ORIGINAL_BROWSER_USER_AGENT" \
  -H "X-Forwarded-For: ORIGINAL_CLIENT_IP"
```

**Response**
```json
{
  "users": [
    { "user_id": "uuid", "user_email": "user@example.com", "user_first_name": "John", "user_last_name": "Doe" }
  ],
  "teams": [{ "team_id": "uuid", "team_name": "Support" }]
}
```

| Status | Body | When |
|---|---|---|
| 401 | `{"error":"invalid session"}` | No valid session |
| 401 | `{"error":"invalid app secret"}` | Wrong or missing app secret |
| 400 | `{"error":"app_id required"}` or `{"error":"invalid app_id format"}` | Missing or malformed `app_id` |
| 429 | `{"error":"too many requests, please try again later"}` | Rate limited; honour `Retry-After` |

---

**Prefer a simpler setup?** See the [SDK overview](sdk.md).
