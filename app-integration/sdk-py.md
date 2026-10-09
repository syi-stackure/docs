# Python SDK

[`stackure`](https://pypi.org/project/stackure/) on PyPI — drop-in ASGI and WSGI middleware, zero dependencies. Python 3.14+.

## Install

```bash
pip install stackure
```

## Configure

```bash
export STACKURE_APP_ID=...       # the app's UUID, shown on the app's page in Stackure
export STACKURE_APP_SECRET=...   # the app secret, shown at registration and on each rotation
```

The SDK reads both from the environment, so no function takes an app ID. See
[Configuration](sdk.md#configuration).

## Protect an app

```python
import stackure

# ASGI — FastAPI, Starlette, Quart
app = stackure.auth()(app)

# WSGI — Flask, Django
flask_app.wsgi_app = stackure.auth()(flask_app.wsgi_app)
```

The same wrapper handles both; it detects the protocol from the app you wrap.

Access the authenticated user in your view:

```python
user = stackure.user_from_request(request)
print(user.user_email, user.account_id)
```

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

## MCP

```python
# ASGI — FastAPI, Starlette
app.mount("/mcp", stackure.mcp()(mcp_app))

# WSGI
mcp_wsgi_app = stackure.mcp()(mcp_wsgi_app)
```

AI clients (Claude, Claude Code, VS Code, Cursor) sign users in through
Stackure. This one line checks every MCP request in real time with the same
app secret; there is no extra setup.

- `mcp` wraps ASGI and WSGI apps like `auth`, with the same `user_from_request`
- It reads `Authorization: Bearer` and ignores cookies, so keep the MCP route outside `stackure.auth`
- A request that is not signed in gets a `401` with a `WWW-Authenticate` header that tells the AI client where to sign in, and the JSON body `{"error":"unauthorized"}`
- A failed check gets a `503`; never a redirect

The MCP endpoint must be served from the same site as the app's registered
URL unless an MCP URL is set for the app in Stackure. See [MCP](sdk.md#mcp).

## Verify manually

```python
result = stackure.verify(request)

if not result.authenticated:
    # result.error.code, result.error.message, result.error.sign_in_url
    ...

# result.user
```

`verify` never raises — transport and API failures come back as a 500 result.
It accepts a WSGI `environ`, an ASGI `scope`, or a framework request object
(Starlette, FastAPI, Flask, Django).

## Send a magic link

```python
resp = stackure.send_magic_link("user@example.com")
# resp.message
```

## Log out

```python
METHODS = ["GET", "HEAD", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]

# Flask
@app.route("/logout", methods=METHODS)
def logout():
    r = stackure.logout(request)
    return "", r.status, r.headers

# FastAPI
@app.api_route("/logout", methods=METHODS, include_in_schema=False)
async def logout(request: Request):
    r = await asyncio.to_thread(stackure.logout, request)
    return Response(status_code=r.status, headers=dict(r.headers))

# Starlette: the same function, routed with
Route("/logout", logout, methods=METHODS)
```

Mount it for every method on the logout path. Trigger it with a form or
button that POSTs from the app's own page; a link or any other request is sent
to Stackure's sign-out page, where the user confirms.

That POST signs the user out everywhere with a server-side call, then returns
the status and headers that clear the app's cookie and redirect to Stackure.
If that call fails, the redirect goes to Stackure's sign-out page, where the
user can finish signing out. See [Sign out](sdk.md#sign-out).

`logout` is synchronous and returns a `Redirect` (`status`, `headers`) for
your framework to send; it never raises. It blocks for up to 2 seconds, hence
`asyncio.to_thread` in the `async def` view.

## Errors

Everything except `verify` and `logout` raises `StackureError`; those two
never raise. Switch on `.code`:

```python
from stackure import StackureError

try:
    stackure.send_magic_link(email)
except StackureError as err:
    match err.code:
        case "validation": ...
        case "auth": ...
        case "forbidden": ...
        case "timeout": ...
        case "network": ...
```

See [SDK overview](sdk.md) for the handoff, session binding, and configuration rules that apply to every SDK.
