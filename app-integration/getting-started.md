# App Integration – Getting Started

## Step 1. Register your app in Stackure

Register your application so Stackure can authenticate users into it.

### Using the Web UI

1.  Sign in to Stackure.
2.  Go to **Apps** → **Add Application**.
3.  Enter the app name, URL, and logo.
4.  Save and copy the generated `app_id`.

### Using the API

``` bash
curl -X POST https://stackure.com/api/internal/app \
  -H "Cookie: stackure_session=YOUR_SESSION_COOKIE" \
  -H "Content-Type: application/json" \
  -d '{
    "app_name": "My App",
    "app_url": "https://myapp.com",
    "app_logo": "https://myapp.com/logo.png",
    "app_logo_type": "url"
  }'
```

**Example response**

``` json
{
  "app_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "message": "app registered successfully"
}
```

---

## Step 2. Grant access to your app

Choose which users or groups should see and launch this app in Stackure.

### Share with your entire account

**Web UI:** Apps → select your app → Share → Entire Account

**API:**

``` bash
curl -X POST https://stackure.com/api/internal/app/account \
  -H "Cookie: stackure_session=YOUR_SESSION_COOKIE" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "YOUR_APP_ID",
    "account_id": "YOUR_ACCOUNT_ID"
  }'
```

### Share with a team

**Web UI:** Apps → select your app → Share → Specific Teams

**API:**

``` bash
curl -X POST https://stackure.com/api/internal/app/team \
  -H "Cookie: stackure_session=YOUR_SESSION_COOKIE" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "YOUR_APP_ID",
    "team_id": "TEAM_ID"
  }'
```

### Share with a partner

**Web UI:** Apps → select your app → Share → Partner

**API:**

``` bash
curl -X POST "https://stackure.com/api/internal/app/partner?app_id=YOUR_APP_ID" \
  -H "Cookie: stackure_session=YOUR_SESSION_COOKIE" \
  -H "Content-Type: application/json" \
  -d '{
    "partner_id": "PARTNER_ID"
  }'
```

---

## Step 3. Add authentication to your app

Stackure authenticates the user. Your app receives a verified session and the user's access context.

Choose your integration path:

### SDK (recommended)

One line to protect a route. The SDK handles the sign-in handoff, session
validation, and permission checks for you.

Set `STACKURE_APP_ID` to the `app_id` from Step 1 and `STACKURE_APP_SECRET` to
the app secret shown once when you registered the app (rotate it from the app's
page if you no longer have it), then follow the guide for your language.

| Language | Guide |
|---|---|
| Go | [Go SDK](sdk-go.md) |
| JavaScript / TypeScript | [JavaScript SDK](sdk-js.md) |
| Python | [Python SDK](sdk-py.md) |
| Rust | [Rust SDK](sdk-rust.md) |

All four behave identically — see the [SDK overview](sdk.md) for the shared
rules.

### Custom API integration

Use the REST endpoints directly for full control.

See the [API Integration Guide](api-integration.md).
