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
- Client Credentials flow
- Token refresh logic


### In Progress / Planned
- JWT parsing and validation
- Comprehensive documentation and examples

## Installation

1. Clone this repository
2. Ensure you have Dyalog APL installed
3. The library includes dependencies:
   - `HttpCommand.dyalog` - HTTP client library
   - `Jarvis.dyalog` - Web server framework

## Quick Start

### Authorization Code Flow with GitHub

```apl
⍝ Create OAuth client using CommonProviders
config←CommonProviders.GitHub
config.clientId←'YOUR_GITHUB_CLIENT_ID'
config.clientSecret←'YOUR_GITHUB_CLIENT_SECRET'
client ← SOAC.New config

⍝ Start temporary callback server
callbackUrl←client.StartCallbackServer 3000

⍝ Generate authorization URL (returns namespace with url and state)
params←(
    redirectUri: callbackUrl
    scope: 'read:user'
)
authResult ← client.GetAuthorizationURL params

⍝ Launch browser and wait for callback
client.LaunchURL authResult.url
result ← client.WaitForCallback

⍝ Exchange code for token (using namespace arguments)
:If result.success
    tokenResult ← client.GetToken (code: result.code ⋄ redirectUri: callbackUrl)

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
⍝ Create OAuth client for device flow using CommonProviders
config←CommonProviders.Google 'deviceCode'
config.clientId←'YOUR_CLIENT_ID'
config.clientSecret←'YOUR_CLIENT_SECRET'
client ← SOAC.New config

⍝ Start device flow
result ← client.StartDeviceFlow (scope: 'openid' 'profile')

:If result.success
    ⎕ ← 'Visit: ', result.verification_uri
    ⎕ ← 'Enter code: ', result.user_code

    ⍝ Wait for token
    tokenResult ← client.WaitForToken (
        deviceCode: result.device_code
        interval: result.interval
    )

    :If tokenResult.success
        ⎕ ← 'Access Token: ', tokenResult.access_token
    :Else
        ⎕ ← 'Token error: ', tokenResult.error_description
    :EndIf
:EndIf
```

## Design Principles

1. **Explicit over implicit** - Step-by-step methods that clearly show what's happening
2. **Simple token handling** - Return tokens to users, let them manage storage/refresh
3. **Flexible callback handling** - Support both automatic server and manual code entry
4. **Helpful errors** - Return structured error namespaces with clear information
5. **Basic validation** - Catch obvious mistakes but trust OAuth servers for detailed validation

## Configuration

SOAC clients are configured using flat namespaces with the following structure:

```apl
config←(
    clientId: 'your-client-id'        ⍝ Required
    clientSecret: 'your-client-secret' ⍝ Optional for PKCE
    usePKCE: 1                        ⍝ Enable PKCE (default: 1 for authCode, 0 otherwise)
    flow: 'authorizationCode'         ⍝ Options: 'authorizationCode', 'deviceCode', 'clientCredentials'
    authUrl: 'https://...'            ⍝ Authorization endpoint
    tokenUrl: 'https://...'           ⍝ Token endpoint
    deviceUrl: 'https://...'          ⍝ Device authorization endpoint (for device flow)
)
```

### Using CommonProviders

For common OAuth providers, use the pre-configured helpers:

```apl
⍝ GitHub
config←CommonProviders.GitHub
config.clientId←'your-id'
config.clientSecret←'your-secret'

⍝ Google (authorization code flow)
config←CommonProviders.Google 'authorizationCode'
config.clientId←'your-id'
config.clientSecret←'your-secret'

⍝ Google (device code flow)
config←CommonProviders.Google 'deviceCode'
config.clientId←'your-id'
config.clientSecret←'your-secret'

⍝ Microsoft
config←CommonProviders.Microsoft 'authorizationCode'
config.clientId←'your-id'
config.clientSecret←'your-secret'

⍝ Auth0 (requires domain)
config←CommonProviders.Auth0 'myapp.auth0.com'
config.clientId←'your-id'
config.clientSecret←'your-secret'
```

## API Reference

### Constructor
- `SOAC.New config` - Create a new OAuth client instance

### Methods

#### Authorization Code Flow
- `GetAuthorizationURL args` - Generate authorization URL with PKCE support
  - Returns: `{url: 'https://...' state: 'uuid'}`
- `StartCallbackServer port` - Start local server to receive OAuth callback
  - Returns: callback URL string
- `WaitForCallback` - Wait for OAuth callback (with 5-minute timeout)
  - Returns: `{success: 1 code: '...' state: '...'}` or error
- `LaunchURL url` - Open URL in system default browser
- `GetToken args` - Exchange authorization code for access token
  - Arguments: `(code: 'auth_code' ⋄ redirectUri: 'callback_url')`
  - Returns: `{success: 1 access_token: '...' ...}` or error

#### Device Code Flow
- `StartDeviceFlow args` - Start device authorization flow
  - Arguments: `(scope: 'openid' 'profile')`
  - Returns: `{success: 1 device_code: '...' user_code: '...' verification_uri: '...' interval: N}`
- `WaitForToken args` - Wait for token completion
  - Arguments: `(deviceCode: 'device_code' ⋄ interval: N)`
  - Returns: `{success: 1 access_token: '...' ...}` or error

#### Utilities
- `ParseUrl url` - Parse URL query parameters into namespace
- `GetState` - Retrieve the current state value (for CSRF validation)

### CommonProviders
- `CommonProviders.GitHub` - GitHub OAuth configuration
- `CommonProviders.Google flow` - Google OAuth configuration (authorizationCode or deviceCode)
- `CommonProviders.Microsoft flow` - Microsoft OAuth configuration (authorizationCode or deviceCode)
- `CommonProviders.Auth0 domain` - Auth0 OAuth configuration

## Security Features

- **PKCE Support** - Proof Key for Code Exchange for enhanced security
- **State Validation** - CSRF protection using state parameter
- **Automatic State Generation** - Cryptographically random UUIDs
- **Secure Code Verifier** - 43-128 character random strings for PKCE

## Error Handling

All methods that can fail return namespaces with a consistent structure:

**Success:**
```apl
(
  success: 1
  ⍝... response data ...⍝
)
```

**Error:**
```apl
(
  success: 0
  error: 'error_code'
  error_description: 'Human-readable description'
  ⍝... additional context ...⍝
)
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
- JWT parsing and signature validation
- Comprehensive test suite
- More usage examples

## License

This Project is licensed under the MIT license.

---

**Note:** This is a work-in-progress library. Use in production environments at your own risk. APIs are subject to change until version 1.0.
