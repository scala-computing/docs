---
title: "Authentication"
description: "The Network Simulation API uses bearer token authentication. All API requests require a valid JWT token in the Authorization header."
---

The Network Simulation API uses bearer token authentication. All API requests require a valid JWT token in the `Authorization` header.

## Obtaining a Token

### Using the CLI

The simplest way to get a token for development is using the CLI:

```bash
# Login (stores token locally)
central-cli login

# Get current token
central-cli token

# Check auth status
central-cli status
```

### Programmatic Token Generation (M2M)

For machine-to-machine authentication, use OAuth 2.0 client credentials flow with AWS Cognito:

```bash
curl -X POST "https://YOUR_COGNITO_DOMAIN/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

Response:
```json
{
  "access_token": "eyJraWQiOiJ...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Contact your administrator for Cognito client credentials.

## Token Refresh

Access tokens are valid for the number of seconds in the token response's `expires_in` field (3600, one hour, by default). A request made with an expired token returns `401 Unauthorized`. A client that runs longer than one token's lifetime, or reuses a cached token across runs, must obtain a new token before the old one expires.

The client credentials flow does not issue a refresh token. To refresh, repeat the [token request](#programmatic-token-generation-m2m) with the same client credentials.

### Refresh Rules

1. **Record when the token expires.** Compute the expiry time as the time of the token request plus `expires_in`. Do not hard-code one hour.
2. **Refresh 60 seconds early.** Before each request, request a new token if the current one expires within 60 seconds. The margin covers clock skew and requests still in flight when the token expires.
3. **Reuse the token.** Cache the token and send it on every request until it needs refreshing. Do not request a new token per API call. If you persist it across runs, treat it as a secret: store it in a file only your user can read.
4. **Retry once on 401, with a different token.** If a request returns `401` with a token your client still considers valid, request a new token and retry that request once. Do not request a new token if another caller has already replaced the rejected token; retry with the replacement instead. Do not request one if the rejected token was fetched less than 60 seconds ago either: a token that new was rejected for a reason other than expiry, and another token will not fix it. Retry only with a token different from the rejected one. If the request still returns `401`, stop and report the error; the usual cause is wrong or revoked client credentials. A permission problem returns [`403`](#403-forbidden), not `401`.
5. **Refresh once, not per thread.** In a concurrent client, let one caller request the new token while the others wait for it and reuse the result.

Expressed as pseudocode:

```text
get_token():
    if cached and now < cached.expires_at - 60s:
        return cached.token
    with refresh_lock:
        if cached and now < cached.expires_at - 60s:   # another caller refreshed it
            return cached.token
        fetch_token()
        return cached.token

fetch_token():                                         # caller holds refresh_lock
    token_response = POST token endpoint (grant_type=client_credentials)
    cached = { token: token_response.access_token,
               expires_at: now + token_response.expires_in,
               fetched_at: now }

send(request):
    token = get_token()
    response = call API with "Authorization: Bearer " + token
    if response.status == 401:
        with refresh_lock:
            # Refetch only if no other caller replaced the rejected token
            # and it was not fetched moments ago.
            if cached.token == token and now - cached.fetched_at >= 60s:
                fetch_token()
            retry_token = cached.token
        if retry_token != token:
            response = call API with "Authorization: Bearer " + retry_token   # once only
    return response
```

### Python Client

The Python client's `auth.py` module follows these rules. A client created with `ScalaAuth.client()` attaches a current token to every request, refreshes it 60 seconds before expiry, caches it on disk between runs in `~/.scala/m2m-token.json` (readable only by your user), and on a `401` fetches a new token and retries once, unless the rejected token was fetched within the last 60 seconds, in which case it returns the `401`. See the Python client README for setup. A client built from a fixed token with `AuthenticatedClient(token=...)` does not refresh; create a new client with a new token before the old token expires.

### CLI

`central-cli token` prints the token stored by `central-cli login` and never refreshes it. Within 5 minutes of the token's expiry it refuses to print it; run `central-cli login` again.

## Using Your Token

Include the token in the `Authorization` header:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/workspaces" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json"
```

## Authentication Errors

### 401 Unauthorized

Returned when:
- No token is provided
- The token is invalid or malformed
- The token has expired

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or missing authentication token"
  }
}
```

### 403 Forbidden

Returned when the token is valid but lacks permission for the requested resource:

```json
{
  "error": {
    "code": "FORBIDDEN", 
    "message": "You do not have access to this resource"
  }
}
```

## Token Best Practices

1. **Store securely** - Use environment variables or a secrets manager
2. **Refresh before expiry** - Tokens expire after `expires_in` seconds (1 hour by default); see [Token Refresh](#token-refresh)
3. **Never log tokens** - Ensure your logging doesn't capture authentication headers
4. **Use HTTPS in production** - Tokens should never be sent over unencrypted connections

## Environment Variable Example

Set your token as an environment variable:

```bash
export SCALA_API_TOKEN="your_token_here"
```

Then use it in your scripts:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/workspaces" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

## SDK Authentication

If using the Rust SDK, configure the client with your token:

```rust
use central_client::{Client, ClientConfig};

let config = ClientConfig::new("https://api.scalacomputing.com")?
    .with_bearer_token(std::env::var("SCALA_API_TOKEN")?);
let client = Client::new(config)?;
```

The Rust client sends the token it was built with and does not refresh it. Request a new token and build a new client before the token expires, following the [refresh rules](#refresh-rules).

See the [SDK Guide](https://github.com/scala-computing/scala/blob/main/code/scala-openapi/docs/CUSTOMER_SDK_GUIDE.md) for more details.
