# Python SDK

[`stackure`](https://pypi.org/project/stackure/) on PyPI — drop-in ASGI and WSGI middleware, zero dependencies. Python 3.14+.

## Install

```bash
pip install stackure
```

## Protect an app

```python
import stackure

app_id = "YOUR_APP_ID"

# ASGI — FastAPI, Starlette, Quart
app = stackure.auth(app_id, "can_approve_invoice")(app)

# WSGI — Flask, Django
flask_app.wsgi_app = stackure.auth(app_id, "can_approve_invoice")(flask_app.wsgi_app)
```

The same wrapper handles both; it detects the protocol from the app you wrap.

Access the authenticated user in your view:

```python
user = stackure.user_from_request(request)
print(user.user_email, user.user_permissions)
```

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

## Verify manually

```python
result = stackure.verify(app_id, request, "can_approve_invoice")

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
resp = stackure.send_magic_link("user@example.com", app_id)
# resp.message
```

## Log out

```python
r = stackure.logout(request)
```

Returns the status and headers that clear the app's cookie and redirect to
Stackure's sign-out. Your framework builds the response:

```python
# Flask
return "", r.status, r.headers

# Starlette / FastAPI
return Response(status_code=r.status, headers=dict(r.headers))
```

## Errors

Everything except `verify` raises `StackureError`. Switch on `.code`:

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
