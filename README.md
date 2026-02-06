# SimpleOAuthClient (SOAC)

> **Work In Progress** - This library is under active development. APIs may change.

A general-purpose OAuth 2.0 client library for Dyalog APL that provides step-by-step control over OAuth flows while remaining simple and explicit.

## Status

This project is currently in development. Core functionality is implemented but not all features are complete.

### Currently Working
- Authorization Code flow with automatic callback handling
- Device Code flow
- PKCE (Proof Key for Code Exchange) support
- Automatic local callback server using Jarvis
- Token exchange
- State validation for CSRF protection
- GitHub OAuth integration (example provided)

### In Progress / Planned
- Client Credentials flow
- Token refresh logic
- JWT parsing and validation
- Provider convenience helpers (Google, Microsoft, Auth0, etc.)
- Comprehensive documentation and examples
- Error handling improvements

## Installation

1. Clone this repository
2. Ensure you have Dyalog APL installed
3. The library includes dependencies:
   - `HttpCommand.dyalog` - HTTP client library
   - `Jarvis.dyalog` - Web server framework

## Quick Start

### Authorization Code Flow with GitHub

```apl
⍝ Create OAuth client
config←(
    client: (
        id: 'YOUR_GITHUB_CLIENT_ID'
        secret: 'YOUR_GITHUB_CLIENT_SECRET'
    )
    auth: (
        authUrl: 'https://github.com/login/oauth/authorize'
        tokenUrl: 'https://github.com/login/oauth/access_token'
    )
)
client ← SOAC.New config

⍝ Start temporary callback server
callbackUrl←client.LaunchCallbackServer 3000

⍝ Generate authorization URL
params←(
    redirectUri: callbackUrl
    scope: 'read:user'
    state: ''
)
authUrl ← client.AuthorizeURL params

⍝ Launch browser and wait for callback
client.LaunchURL authUrl
result ← client.AwaitCallback

⍝ Exchange code for token
:If result.success
    code ← result.code
    tokenResult ← client.GetToken code callbackUrl

    :If tokenResult.success
        ⎕ ← 'Access Token: ', tokenResult.access_token
    :Else
        ⎕ ← 'Token error: ', tokenResult.error_description
    :EndIf
:Else
    ⎕ ← 'Authorization error: ', result.error_description
:EndIf
```

### Device Code Flow

```apl
⍝ Create OAuth client for device flow
config←(
    client: (
        id: 'YOUR_CLIENT_ID'
        secret: 'YOUR_CLIENT_SECRET'
        flow: 'deviceCode'
    )
    auth: (
        deviceUrl: 'https://provider.com/device/code'
        tokenUrl: 'https://provider.com/oauth/token'
    )
)
client ← SOAC.New config

⍝ Initiate device flow
result ← client.InitiateDeviceFlow (scope: 'openid' 'profile')

:If result.success
    ⎕ ← 'Visit: ', result.verification_uri
    ⎕ ← 'Enter code: ', result.user_code

    ⍝ Poll for token
    tokenResult ← client.PollForToken ()
:EndIf
```

## Design Principles

1. **Explicit over implicit** - Step-by-step methods that clearly show what's happening
2. **Simple token handling** - Return tokens to users, let them manage storage/refresh
3. **Flexible callback handling** - Support both automatic server and manual code entry
4. **Helpful errors** - Return structured error namespaces with clear information
5. **Basic validation** - Catch obvious mistakes but trust OAuth servers for detailed validation

## Configuration

SOAC clients are configured using namespaces with the following structure:

```apl
config←(
    client: (
        id: 'your-client-id'          ⍝ Required
        secret: 'your-client-secret'  ⍝ Optional for PKCE
        usePKCE: 1                    ⍝ Enable PKCE (default: 1 for authCode, 0 otherwise)
        flow: 'authorizationCode'     ⍝ Options: 'authorizationCode', 'deviceCode', 'clientCredentials'
    )
    auth: (
        authUrl: 'https://...'        ⍝ Authorization endpoint
        tokenUrl: 'https://...'       ⍝ Token endpoint
        deviceUrl: 'https://...'      ⍝ Device authorization endpoint (for device flow)
    )
)
```

## API Reference

### Constructor
- `SOAC.New config` - Create a new OAuth client instance

### Methods

#### Authorization Code Flow
- `AuthorizeURL args` - Generate authorization URL with PKCE support
- `LaunchCallbackServer port` - Start local server to receive OAuth callback
- `AwaitCallback` - Wait for OAuth callback (with 5-minute timeout)
- `LaunchURL url` - Open URL in system default browser
- `GetToken args` - Exchange authorization code for access token

#### Device Code Flow
- `InitiateDeviceFlow args` - Start device authorization flow
- `PollForToken args` - Poll for token completion (WIP)

#### Utilities
- `ParseUrl url` - Parse URL query parameters into namespace
- `GetState` - Retrieve the current state value (for CSRF validation)

### Shared Methods
- `sha256 data` - SHA-256 hash (used for PKCE)
- `hmacsha256 data` - HMAC-SHA256 (utility)

## Security Features

- **PKCE Support** - Proof Key for Code Exchange for enhanced security
- **State Validation** - CSRF protection using state parameter
- **Automatic State Generation** - Cryptographically random UUIDs
- **Secure Code Verifier** - 43-128 character random strings for PKCE

## Error Handling

All methods that can fail return namespaces with a consistent structure:

**Success:**
```apl
{
  success: 1
  [... response data ...]
}
```

**Error:**
```apl
{
  success: 0
  error: 'error_code'
  error_description: 'Human-readable description'
  [... additional context ...]
}
```

## Dependencies

- **Dyalog APL** - Version 20.0 or later
- **HttpCommand** - Included in APLSource
- **Jarvis** - Web server framework, included in APLSource
- **Conga SSL** - Required for cryptographic operations (typically bundled with Dyalog)

## Contributing

This project is in active development. If you encounter issues or have suggestions, please open an issue.

## Roadmap

**Upcoming Features:**
- Token refresh functionality
- JWT parsing and signature validation
- Provider helpers (Google, Microsoft, Auth0)
- Client Credentials flow completion
- Comprehensive test suite
- More usage examples

## License

This Project is licensed under the MIT license.

---

**Note:** This is a work-in-progress library. Use in production environments at your own risk. APIs are subject to change until version 1.0.
