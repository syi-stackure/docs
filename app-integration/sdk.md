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

---

## Sign out

Mount `logout` so every method on the logout path reaches it, not POST only,
and trigger it with a form or button that POSTs from your app's own page:

```html
<form method="post" action="/logout"><button>Sign out</button></form>
```

A link or any other request is sent to Stackure's sign-out page, where the
user confirms, with no call made and the cookie left in place.

A request from your app's own page is a POST with
`Sec-Fetch-Site: same-origin` or, only when that header is absent, with an
`Origin` whose host and port match `Host` and whose scheme is `https` if the
request arrived over HTTPS. A request that repeats `Sec-Fetch-Site`, `Origin`
or `Host` never counts. On that POST, `logout`:

1. Signs the user out everywhere with a server-side call to Stackure
2. Clears your app's cookie
3. Redirects to Stackure, or, if the call failed, to Stackure's sign-out page,
   where the user can finish signing out

The cookie is `SameSite=Lax`, so without this check a link on any other site
could sign the user out with no confirmation.

---

## MCP

AI clients such as Claude, Claude Code, VS Code and Cursor reach your app
through its MCP endpoint. Protecting it takes one line: mount the MCP
middleware on that route, `MCP` in Go and `mcp` in the other SDKs. It uses
the same `STACKURE_APP_ID` and `STACKURE_APP_SECRET` as the auth middleware,
so there is no new secret and no extra setup.

1. The user adds your app's MCP address in their AI client
2. The client is told to sign in at Stackure
3. The user signs in and confirms
4. From then on, every MCP request is checked in real time by that one line

Access ends when the user signs out, disconnects the client, loses access to
your app, or leaves the connection unused for 30 days. Users can see and
disconnect their AI clients in Stackure under **AI Clients**.

**The MCP endpoint must be on the same site as your app's registered URL**
(the same host and port), unless an MCP URL is set for the app in Stackure.
The SDK works out the endpoint's address from the request's scheme, `Host`
header and path, so behind a proxy or CDN make sure the original `Host` and
`X-Forwarded-Proto` reach your app.

The MCP middleware reads only `Authorization: Bearer`. It ignores cookies and
never redirects, so mount it on the MCP route alone, not behind the auth
middleware as well. When it turns a request away, the answer is JSON:

| Status | Body | When |
|---|---|---|
| 401 | `{"error":"unauthorized"}` | Not signed in. The `WWW-Authenticate` header tells the AI client where to sign in |
| 503 | `{"error":"unavailable"}` | The check against Stackure could not be completed |

---

## Identity facts

Every authenticated user, from the auth or the MCP middleware, carries two
facts from their Stackure org (`UserIsAppAdmin` and `UserTeams` in Go):

| Field | Meaning |
|---|---|
| `user_is_app_admin` | The user is an app admin or owner in their Stackure org, in charge of its apps |
| `user_teams` | The Stackure teams the user belongs to, each a `team_id` and `team_name`; empty when none |

The directory call lists the users and teams in the caller's org who can open
your app, for pickers and sharing. It uses the request's session cookie, so
call it from a route behind the auth middleware; MCP bearer tokens are not
accepted. No valid session is an `auth` error.

Stackure defines no in-app permissions. Your app decides what these facts
mean.

---

## Configuration

There is no configuration API. Every SDK reads its settings from the
environment:

```bash
export STACKURE_APP_ID=...       # the app's UUID, shown on the app's page in Stackure
export STACKURE_APP_SECRET=...   # the app secret, shown at registration and on each rotation
```

`STACKURE_APP_ID` is read at call time by every call except `logout`, so no
function takes an app ID. If it is unset, the call fails with a `validation` error, `STACKURE_APP_ID is not
set`; if it is not a UUID, with `invalid STACKURE_APP_ID format (must be a
valid UUID)`. In the middleware and `verify` this surfaces like any other
failed check: the auth middleware answers 500, the MCP middleware 503, and
`verify` returns a 500 result.

Point an SDK at a non-production environment by setting `STACKURE_BASE_URL`
before the first call:

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
| `validation` | Bad input or a missing or malformed setting, caught before any request |
| `auth` | 401 from the API |
| `forbidden` | 403 from the API |
| `timeout` | Request exceeded the 2-second timeout |
| `network` | Everything else |

`verify` is the exception: it never fails. Transport and API problems come
back as a 500 result so you can decide how to respond. `logout` never fails
either: it always answers with a redirect.

---

## API surface

| | Go | JavaScript | Python | Rust |
|---|---|---|---|---|
| Middleware | `Auth` | `auth` | `auth` | `auth` |
| MCP middleware | `MCP` | `mcp` | `mcp` | `mcp` |
| Manual check | `Verify` | `verify` | `verify` | `verify` |
| Read user | `UserFromContext` | `userFromRequest` | `user_from_request` | `user_from_request` |
| Magic link | `SendMagicLink` | `sendMagicLink` | `send_magic_link` | `send_magic_link` |
| Raw validation | `ValidateSession` | `validateSession` | `validate_session` | `validate_session` |
| Directory | `Directory` | `directory` | `directory` | `directory` |
| Sign out | `Logout` | `logout` | `logout` | `logout` |

---

**Prefer full control?** See the [API Integration Guide](api-integration.md).
