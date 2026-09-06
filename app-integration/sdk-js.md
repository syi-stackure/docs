# JavaScript SDK

[`stackure`](https://www.npmjs.com/package/stackure) on npm — drop-in Express/Connect middleware, zero dependencies. Node.js 22+, ESM only.

## Install

```bash
npm install stackure
```

## Protect a route

```js
import { auth, userFromRequest } from 'stackure';

const appId = 'YOUR_APP_ID';

app.get('/admin', auth(appId, 'view_any_app'), (req, res) => {
  const user = userFromRequest(req);
  res.json({ email: user.user_email, permissions: user.user_permissions });
});
```

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

The middleware writes with `res.setHeader` / `res.writeHead`, so it works on
Express, Connect, and a bare `http.createServer`. On Fastify, pass
`request.raw` and `reply.raw`.

## Verify manually

```js
import { verify } from 'stackure';

const result = await verify(appId, req, 'view_any_app');

if (!result.authenticated) {
  return res.status(result.error.code).json(result.error);
}

// result.user
```

`verify` never throws — transport and API failures come back as a 500 result.

## Send a magic link

```js
import { sendMagicLink } from 'stackure';

const resp = await sendMagicLink('user@example.com', appId);
// resp.message
```

## Log out

```js
import { logout } from 'stackure';

app.get('/logout', (req, res) => logout(req, res));
```

Clears the app's cookie and redirects to Stackure's sign-out.

## Errors

Everything except `verify` throws `StackureError`. Switch on `.code`:

```js
import { StackureError } from 'stackure';

try {
  await sendMagicLink(email);
} catch (err) {
  if (err instanceof StackureError) {
    // err.code is "validation" | "auth" | "forbidden" | "timeout" | "network"
  }
}
```

See [SDK overview](sdk.md) for the handoff, session binding, and configuration rules that apply to every SDK.
