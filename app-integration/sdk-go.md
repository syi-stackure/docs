# Go SDK

[`stackure.com/sdk-go`](https://pkg.go.dev/stackure.com/sdk-go) — drop-in `net/http` middleware, zero dependencies.

## Install

```bash
go get stackure.com/sdk-go
```

## Configure

```bash
export STACKURE_APP_ID=...       # the app's UUID, shown on the app's page in Stackure
export STACKURE_APP_SECRET=...   # the app secret, shown at registration and on each rotation
```

The SDK reads both from the environment, so no function takes an app ID. See
[Configuration](sdk.md#configuration).

## Protect a route

```go
import "stackure.com/sdk-go"

http.Handle("/admin", stackure.Auth()(handler))
```

Access the authenticated user in your handler:

```go
user := stackure.UserFromContext(r.Context())
fmt.Println(user.UserEmail, user.AccountID)
```

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

## MCP

```go
http.Handle("/mcp", stackure.MCP()(mcpHandler)) // mcpHandler is your MCP server's http.Handler
```

AI clients such as Claude, Claude Code, VS Code and Cursor sign users in
through Stackure. This one line checks every MCP request against Stackure in
real time with the same app secret. There is no extra setup.

`MCP` attaches the user the same way as `Auth`, so `UserFromContext` works in
`mcpHandler`.

- A request that is not signed in gets a `401` with a `WWW-Authenticate` header that tells the AI client where to sign in, and the JSON body `{"error":"unauthorized"}`
- A check that cannot be completed gets a `503`
- `MCP` never redirects and never reads or sets a cookie

The MCP endpoint must be served from the same site as the app's registered
URL unless an MCP URL is set for the app in Stackure. See [MCP](sdk.md#mcp).

## Identity facts

Every authenticated `User`, from `Auth` or `MCP`, also carries
`UserIsAppAdmin` (an app admin or owner in their Stackure org) and `UserTeams`
(their Stackure teams, `[]Team{TeamID, TeamName}`, empty when none).

List the users and teams in the caller's org who can open the app:

```go
dir, err := stackure.Directory(r)
// dir.Users: UserID, UserEmail, UserFirstName, UserLastName
// dir.Teams: TeamID, TeamName
```

`Directory` uses the session cookie, so call it behind `Auth`, not `MCP`. No
valid session is an `auth` error.

Stackure defines no in-app permissions; your app decides what these facts
mean. See [Identity facts](sdk.md#identity-facts).

## Verify manually

```go
result := stackure.Verify(r)

if !result.Authenticated {
    // result.Error.Code, result.Error.Message, result.Error.SignInURL
    return
}

// result.User
```

`Verify` never returns an error — transport and API failures come back as a 500 result.

## Send a magic link

```go
resp, err := stackure.SendMagicLink("user@example.com")
// resp.Message
```

## Log out

```go
http.HandleFunc("/logout", stackure.Logout)
```

Mount `Logout` on the path alone, with no method in the pattern (`"/logout"`,
not `"POST /logout"`), so every request reaches it. Trigger it with a form or
button that POSTs from the app's own page; a link or any other request is sent
to Stackure's sign-out page, where the user confirms.

`Logout` has the `http.HandlerFunc` signature and returns nothing. That POST
signs the user out everywhere with a server-side call, clears the app's
cookie, and redirects to Stackure. If the call fails, the redirect goes to
Stackure's sign-out page, where the user can finish signing out. See
[Sign out](sdk.md#sign-out).

## Errors

All errors are `*stackure.StackureError`. Switch on `.Code`:

```go
var se *stackure.StackureError
if errors.As(err, &se) {
    switch se.Code {
    case "validation", "auth", "forbidden", "timeout", "network":
        // ...
    }
}
```

See [SDK overview](sdk.md) for the handoff, session binding, and configuration rules that apply to every SDK.
