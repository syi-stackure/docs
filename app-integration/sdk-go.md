# Go SDK

[`stackure.com/sdk-go`](https://pkg.go.dev/stackure.com/sdk-go) — drop-in `net/http` middleware, zero dependencies.

## Install

```bash
go get stackure.com/sdk-go
```

## Protect a route

```go
import "stackure.com/sdk-go"

const appID = "YOUR_APP_ID"

http.Handle("/admin", stackure.Auth(appID, "can_approve_invoice")(handler))
```

Access the authenticated user in your handler:

```go
user := stackure.UserFromContext(r.Context())
fmt.Println(user.UserEmail, user.UserPermissions)
```

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

## Verify manually

```go
result := stackure.Verify(appID, r, "can_approve_invoice")

if !result.Authenticated {
    // result.Error.Code, result.Error.Message, result.Error.SignInURL
    return
}

// result.User
```

`Verify` never returns an error — transport and API failures come back as a 500 result.

## Send a magic link

```go
resp, err := stackure.SendMagicLink("user@example.com", appID)
// resp.Message
```

## Log out

```go
stackure.Logout(w, r)
```

Clears the app's cookie and redirects to Stackure's sign-out.

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
