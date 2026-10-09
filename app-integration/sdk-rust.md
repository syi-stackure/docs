# Rust SDK

[`stackure`](https://crates.io/crates/stackure) on crates.io — drop-in tower middleware for axum, tonic, and hyper. Rust 2024 edition.

## Install

```toml
[dependencies]
stackure = "1"
```

## Configure

```bash
export STACKURE_APP_ID=...       # the app's UUID, shown on the app's page in Stackure
export STACKURE_APP_SECRET=...   # the app secret, shown at registration and on each rotation
```

The SDK reads both from the environment, so no function takes an app ID. See
[Configuration](sdk.md#configuration).

## Protect an app

```rust
use stackure::{auth, user_from_request};

let app = Router::new()
    .route("/admin", get(handler))
    .layer(auth());
```

Access the authenticated user in your handler:

```rust
let user = user_from_request(&parts).unwrap();
println!("{} {}", user.user_email, user.account_id);
```

In axum you can also take an `Extension<User>` directly.

- API requests get JSON errors
- Browser requests get redirected to sign-in
- The sign-in handoff is automatic — see [Sign-in handoff](sdk.md#sign-in-handoff)

## MCP

```rust
use stackure::mcp;

let app = Router::new()
    .route("/mcp", any(handler))
    .layer(mcp());
```

AI clients such as Claude, Claude Code, VS Code and Cursor sign users in
through Stackure. This one line checks every MCP request in real time with the
same app secret; there is no extra setup.

- `mcp` attaches the user exactly as `auth` does
- Only `Authorization: Bearer` is read; cookies are ignored
- A request that is not signed in gets a `401` with a `WWW-Authenticate` header that tells the AI client where to sign in, and the JSON body `{"error":"unauthorized"}`
- A failed check gets a `503`, all JSON and never a redirect

The MCP endpoint must be served from the same site as the app's registered
URL unless an MCP URL is set for the app in Stackure. See [MCP](sdk.md#mcp).

## Verify manually

```rust
let result = stackure::verify(&parts).await;

if !result.authenticated {
    let error = result.error.unwrap();
    // error.code, error.message, error.sign_in_url
}

// result.user
```

`verify` never returns an error — transport and API failures come back as a
500 result.

## Send a magic link

```rust
let resp = stackure::send_magic_link("user@example.com").await?;
// resp.message
```

## Log out

```rust
async fn logout(parts: Parts) -> Response<Body> {
    stackure::logout(&parts).await
}

let app = Router::new().route("/logout", any(logout));
```

Mount it for every method on the logout path, as `any` does here. Trigger it
with a form or button that POSTs from the app's own page; a link or any other
request is sent to Stackure's sign-out page, where the user confirms.

That POST signs the user out everywhere with a server-side call, then returns
a 303 that clears the app's cookie and redirects to Stackure. If that call
fails, the 303 goes to Stackure's sign-out page, where the user can finish
signing out. See [Sign out](sdk.md#sign-out).

`logout` is asynchronous: awaiting it yields the `Response<B>`, never an
error.

## Errors

Everything except `verify` and `logout` returns `StackureError`. Match on the
variant, or call `.code()` for the same category string the other SDKs expose
as `.code`:

```rust
use stackure::StackureError;

match stackure::send_magic_link(email).await {
    Err(StackureError::Validation(m)) => {}
    Err(StackureError::Auth(m)) => {}
    Err(StackureError::Forbidden(m)) => {}
    Err(StackureError::Timeout(m)) => {}
    Err(StackureError::Network(m)) => {}
    Ok(resp) => {}
}
```

## Dependencies

Rust's standard library has no HTTP client and no TLS, so unlike the other
three SDKs this one cannot be dependency-free. It builds on `hyper` and
`rustls` — the stack axum and tonic already run on — rather than a
higher-level client, so in a typical axum app it adds around twenty crates.

See [SDK overview](sdk.md) for the handoff, session binding, and configuration rules that apply to every SDK.
