# SDK Integration Guide

Stackure ships four SDKs. They are behaviourally identical — same session
handling, same errors, same middleware semantics — so pick your language and
follow that page.

| Language | Package | Middleware for |
|---|---|---|
| [Go](sdk-go.md) | `stackure.com/sdk-go` | `net/http` |
| [JavaScript](sdk-js.md) | `stackure` (npm) | Express, Connect, bare `http` |
| [Python](sdk-py.md) | `stackure` (PyPI) | ASGI and WSGI |
| [Rust](sdk-rust.md) | `stackure` (crates.io) | tower — axum, tonic, hyper |

Go, JavaScript, and Python have zero dependencies. Rust cannot: its standard
library has no HTTP client and no TLS.

---

## Sign-in handoff

You do not implement this. The middleware does it for you.

After the user clicks the magic link, Stackure hands the browser back to your
app's registered URL carrying a `session_token` — as a POST form field, or as
a `?session_token=` query parameter.

The middleware consumes either one automatically:

1. Reads the token off the incoming request
2. Stores it in a `session` cookie on **your** domain — `HttpOnly`, `SameSite=Lax`, `Secure` when the request is HTTPS
3. Redirects to the same URL with the parameter stripped, so the token does not linger in the address bar

Stackure's own session cookie is scoped to the Stackure host and is never
visible to your app.

---

## Session binding

Stackure binds each session to the browser's user agent and IP address.
Because the SDK validates from your server rather than from the browser, it
forwards the original `User-Agent` and `X-Forwarded-For` on every validation
call.

**Your app must see the real client IP.** If it runs behind a proxy or CDN,
make sure that layer sets `X-Forwarded-For`. Without it, Stackure sees your
server's address instead of the browser's and rejects the session.

Every request is validated against Stackure, so revoking a session takes
effect immediately.

---

## Content negotiation

The middleware inspects the `Accept` header on a 401:

- `text/html` → redirects to the sign-in URL
- `application/json` → returns a JSON error body

```json
{
  "error": "Unauthorized",
  "message": "Valid authentication required",
  "sign_in_url": "https://stackure.com/sign-in/magic-link?app_id=YOUR_APP_ID"
}
```

A 403 always returns JSON:

```json
{
  "error": "Forbidden",
  "message": "Requires one of: can_approve_invoice",
  "sign_in_url": ""
}
```

---

## Permissions

Pass the permissions a route requires. The user must hold at least one of
them; passing none means "any authenticated user".

The user's permissions arrive as `user_permissions` on the user object.

---

## Configuration

There is no configuration API. Point an SDK at a non-production environment by
setting `STACKURE_BASE_URL` before the first call:

```bash
STACKURE_BASE_URL=https://stage.stackure.com
```

Retry-on-5xx (one retry after 500 ms) and the 2-second request timeout are
hard-coded in every SDK. Timeouts are never retried.

---

## Errors

Every SDK exposes one error type with the same five categories:

| Code | Meaning |
|---|---|
| `validation` | Bad input, caught before any request |
| `auth` | 401 from the API |
| `forbidden` | 403 from the API |
| `timeout` | Request exceeded the 2-second timeout |
| `network` | Everything else |

`verify` is the exception: it never fails. Transport and API problems come
back as a 500 result so you can decide how to respond.

---

## API surface

| | Go | JavaScript | Python | Rust |
|---|---|---|---|---|
| Middleware | `Auth` | `auth` | `auth` | `auth` |
| Manual check | `Verify` | `verify` | `verify` | `verify` |
| Read user | `UserFromContext` | `userFromRequest` | `user_from_request` | `user_from_request` |
| Magic link | `SendMagicLink` | `sendMagicLink` | `send_magic_link` | `send_magic_link` |
| Raw validation | `ValidateSession` | `validateSession` | `validate_session` | `validate_session` |
| Sign out | `Logout` | `logout` | `logout` | `logout` |

---

**Prefer full control?** See the [API Integration Guide](api-integration.md).
