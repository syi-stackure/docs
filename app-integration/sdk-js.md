# JavaScript SDK

[`stackure`](https://www.npmjs.com/package/stackure) on npm — drop-in Express/Connect middleware, zero dependencies. Node.js 22+, ESM only.

## Install

```bash
npm install stackure
```

## Configure

```bash
export STACKURE_APP_ID=...       # the app's UUID, shown on the app's page in Stackure
export STACKURE_APP_SECRET=...   # the app secret, shown at registration and on each rotation
```

The SDK reads both from the environment, so no function takes an app ID. See
[Configuration](sdk.md#configuration).

## Protect a route

```js
import { auth, userFromRequest } from 'stackure';

app.get('/admin', auth(), (req, res) => {
  const user = userFromRequest(req);
  res.json({ email: user.user_email, account: user.account_id });
});
```

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

The middleware writes with `res.setHeader` / `res.writeHead`, so it works on
Express, Connect, and a bare `http.createServer`. On Fastify, pass
`request.raw` and `reply.raw`.

## MCP

```js
import { mcp, userFromRequest } from 'stackure';

app.all('/mcp', mcp(), (req, res) => {
  const user = userFromRequest(req);
  // serve the MCP request
});
```

AI clients such as Claude, Claude Code, VS Code and Cursor sign users in
through Stackure. This one line checks every MCP request in real time with the
same app secret. There is no extra setup.

`mcp` attaches the user the same way as `auth`.

- A request that is not signed in gets a `401` with a `WWW-Authenticate` header that tells the AI client where to sign in, and the JSON body `{"error":"unauthorized"}`
- A failed check gets a `503`
- `mcp` reads only `Authorization: Bearer`, never a cookie, and never redirects

The MCP endpoint must be served from the same site as the app's registered
URL unless an MCP URL is set for the app in Stackure. See [MCP](sdk.md#mcp).

## Identity facts

Every authenticated user, from `auth` or `mcp`, also carries
`user_is_app_admin` (an app admin or owner in their Stackure org) and
`user_teams` (their Stackure teams, `{ team_id, team_name }[]`, empty when
none).

List the users and teams in the caller's org who can open the app:

```js
import { directory } from 'stackure';

const { users, teams } = await directory(req);
// users: { user_id, user_email, user_first_name, user_last_name }[]
// teams: { team_id, team_name }[]
```

`directory` uses the session cookie, so call it behind `auth`, not `mcp`. No
valid session throws `StackureError` with code `auth`.

Stackure defines no in-app permissions; your app decides what these facts
mean. See [Identity facts](sdk.md#identity-facts).

## Verify manually

```js
import { verify } from 'stackure';

const result = await verify(req);

if (!result.authenticated) {
  return res.status(result.error.code).json(result.error);
}

// result.user
```

`verify` never throws — transport and API failures come back as a 500 result.

## Send a magic link

```js
import { sendMagicLink } from 'stackure';

const resp = await sendMagicLink('user@example.com');
// resp.message
```

## Log out

```js
import { logout } from 'stackure';

app.all('/logout', (req, res) => logout(req, res));
```

Mount it for every method on the logout path, as `app.all` does here. Trigger
it with a form or button that POSTs from the app's own page; a link or any
other request is sent to Stackure's sign-out page, where the user confirms.

`logout` returns a promise (`Promise<void>`). That POST signs the user out
everywhere through Stackure's API, clears the app's cookie, and redirects to
Stackure. If the API call fails, the redirect goes to Stackure's sign-out
page, where the user can finish signing out. See
[Sign out](sdk.md#sign-out).

## Errors

Everything except `verify` and `logout` throws `StackureError`. Switch on `.code`:

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
