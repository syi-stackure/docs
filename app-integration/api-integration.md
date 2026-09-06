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
  -H "User-Agent: ORIGINAL_BROWSER_USER_AGENT" \
  -H "X-Forwarded-For: ORIGINAL_CLIENT_IP"
```

**Response if the session is valid**
```json
{
  "authenticated": true,
  "user": {
    "user_id": "uuid",
    "user_email": "user@example.com",
    "user_first_name": "John",
    "user_last_name": "Doe",
    "user_permissions": ["view_any_app", "edit_any_app"]
  }
}
```

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

```bash
curl -X POST https://stackure.com/api/public/auth/sign-out \
  -H "Authorization: Bearer SESSION_TOKEN"
```

Clear the cookie you set in Step 2 as well.

---

**Prefer a simpler setup?** See the [SDK overview](sdk.md).
