# Authentication and session

Auth runs against a separate origin IAM service, from every AI
feature, but shares one token: the IAM service issues the JWT, the AI
service validates it with the same public key.

```mermaid
flowchart TD
    Visit["Visit any protected route"] --> Guard{RequireAuth:<br/>session status?}
    Guard -->|"loading<br/>(refresh token in storage)"| Restoring["'Restoring your session…'"]
    Restoring --> Refresh["POST /iam/auth/refresh-token<br/>{refresh_token}"]
    Refresh -->|200| Authed[Authenticated — render the route]
    Refresh -->|401| Anon
    Guard -->|anonymous, no tokens| Anon[Redirect to /login<br/>state.from = attempted path]
    Guard -->|authenticated| Authed

    Anon --> LoginPage["Login screen<br/>(username, password)"]
    LoginPage -->|Sign in| LoginCall["POST /iam/auth/login<br/>{username, password}"]
    LoginCall -->|200| Store["tokenStore.setTokens()<br/>access token in memory,<br/>refresh token in localStorage"]
    Store --> Redirect["Navigate to state.from, or /jobs"]
    LoginCall -->|401| BadCreds["'Username or password is wrong.'"]
    LoginCall -->|validation errors| FieldErr["Field-level errors under<br/>Username / Password"]
    LoginCall -->|network/other| Unreachable["'Could not reach the sign-in<br/>service. Check that it is running.'"]

    Authed --> AnyCall["Any authenticated API call"]
    AnyCall -->|200| Done[Rendered normally]
    AnyCall -->|401| SingleFlight["Single-flight refresh<br/>(reuses an in-flight one)"]
    SingleFlight -->|refreshed| Retry["Retry the original request once"]
    Retry -->|200| Done
    Retry -->|401 again| Propagate[ApiError propagates to the caller]
    SingleFlight -->|refresh 401| Clear["tokenStore.clear() → anonymous"]
    Clear --> Anon

    Authed --> Logout["Sign out"]
    Logout --> LogoutCall["POST /iam/auth/logout<br/>{refresh_token}"]
    LogoutCall --> ClearAlways["tokenStore.clear() regardless of<br/>the call's outcome"]
    ClearAlways --> Anon
```

## Details

- **Session bootstrap.** On load, `SessionProvider` starts `authenticated` if an access
  token is already in memory, `loading` if only a refresh token survives in
  `localStorage` (a page reload), or `anonymous` otherwise.
- **Single-flight refresh.** `refreshAccessToken()` keeps a module-level in-flight
  promise; concurrent 401s from different requests all await the same refresh rather than
  each starting their own. On a genuine 401 from the refresh call itself, the refresh
  token is cleared; on a network error it's left alone so a later retry can still succeed.
- **Retry-once.** `createClient()`'s `onUnauthorized` hook (wired to `refreshAccessToken`)
  fires on any 401, awaits the refresh, and retries the original request exactly once
  (`isRetry` prevents a second loop). The refresh call itself is made on a client with no
  `onUnauthorized`, so it can never recursively trigger another refresh.
- **`RequireAuth`** is the only route guard: `loading` → a plain restoring message,
  `anonymous` → redirect to `/login` with `state.from` so the login screen can send the
  user back to what they were trying to reach, otherwise render the protected route.
- **Logout always clears local state** in a `finally`, even if the `POST
  /iam/auth/logout` call fails — a stuck server-side session shouldn't leave the client
  looking logged in.
