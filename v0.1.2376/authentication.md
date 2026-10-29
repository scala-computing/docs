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
central-cli auth login

# Get current token
central-cli auth token

# Check auth status
central-cli auth status
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
2. **Rotate regularly** - Tokens expire after 1 hour by default; refresh before expiry
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

See the [SDK Guide](https://github.com/scala-computing/scala/blob/main/code/scala-openapi/docs/CUSTOMER_SDK_GUIDE.md) for more details.
