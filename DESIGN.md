# SimpleOAuthClient Design Plan

## Design Vision

A **general-purpose OAuth 2.0 client library** for Dyalog APL that provides step-by-step control over the OAuth flow while remaining simple and explicit.

### Design Principles
1. **Explicit over implicit** - Step-by-step methods that clearly show what's happening
2. **Simple token handling** - Return tokens to users, let them manage storage/refresh
3. **Flexible callback handling** - Support both automatic server and manual code entry
4. **Helpful errors** - Return structured error namespaces with clear information
5. **Basic validation** - Catch obvious mistakes but trust OAuth servers for detailed validation

### MVP Deliverable

The initial implementation will provide a **working OAuth 2.0 Authorization Code flow** that enables users to:

1. Authenticate users through any OAuth 2.0 provider (GitHub, Google, Auth0, etc.)
2. Automatically handle callbacks via a temporary local server
3. Exchange authorization codes for access tokens
4. Use those tokens to access protected APIs

**What you'll be able to build with the MVP:**
- APL applications that integrate with third-party OAuth APIs (GitHub, Google, Slack, etc.)
- Custom APL web applications with OAuth-based authentication
- Quick API token acquisition for testing and development

**What's deferred to future phases:**
- PKCE (can still use basic Authorization Code flow with client_secret)
- Client Credentials flow (for machine-to-machine auth)
- Token refresh logic (can be added by users manually)
- JWT parsing/validation (tokens work as opaque strings for now)
- Provider convenience helpers (users specify URLs manually)

---

## Target Flows

### 1. Authorization Code Flow (with server)
Most common 3-legged OAuth flow for web applications.

### 2. Authorization Code + PKCE
More secure variant, doesn't require client_secret. Good for public clients.

### 3. Client Credentials Flow
Machine-to-machine authentication for backend services.

---

## Usage Examples

### Example 1: Authorization Code Flow with Automatic Callback Server

```apl
⍝ Create OAuth client
config←(
    client: (
        id: 'YOUR_CLIENT_ID'
        secret: 'YOUR_CLIENT_SECRET'
    )
    auth: (
        authUrl: 'https://auth.example.com/oauth/authorize'
        tokenUrl: 'https://auth.example.com/oauth/token'
    )
)
client ← SOAC.New config

⍝ Start temp jarvis server
callbackUrl←client.LaunchCallbackServer 3000 ⍝ Starts Jarvis server on port 3000

⍝ Generate authorization URL
params←(
    redirectUri: callbackUrl
    scope: 'email' 'profile'
    state: '' ⍝ Optional state parameter
    custom: (
        ⍝ Custom params to be passed (Will not be parsed)
    )
)
authUrl ← client.AuthorizeURL params 

⍝ Show URL to user
⎕ ← 'Please visit: ', authUrl

⍝ wait for code
client.LaunchURL authUrl
result ← client.AwaitCallback ⍝ Waits until callback received. Autostops the Jarvis server



⍝ Check if successful
:If result.success
    code ← result.code

    ⍝ Exchange code for token
    tokenResult ← client.GetToken code callbackUrl

    :If tokenResult.success
        ⎕ ← 'Access Token: ', tokenResult.access_token
        ⎕ ← 'Refresh Token: ', tokenResult.refresh_token
        ⎕ ← 'Expires in: ', tokenResult.expires_in
    :Else
        ⎕ ← 'Token error: ', tokenResult.error_description
    :EndIf
:Else
    ⎕ ← 'Authorization error: ', result.error_description
:EndIf
```

### Example 2: Authorization Code Flow with Manual Code Entry

```apl
⍝ Create OAuth client
client ← SOAC.New 'my-client-id' 'https://auth.example.com/oauth/authorize' 'urn:ietf:wg:oauth:2.0:oob' 'read write'
client.clientSecret ← 'my-client-secret'

⍝ Generate authorization URL
authUrl ← client.GenerateAuthorizationUrl ''

⍝ User opens URL in browser and copies code
⎕ ← 'Visit this URL and copy the code:'
⎕ ← authUrl
code ← ⍞  ⍝ User pastes code

⍝ Exchange code for token
tokenResult ← client.GetToken code

:If tokenResult.success
    ⎕ ← 'Access Token: ', tokenResult.access_token
:Else
    ⎕ ← 'Error: ', tokenResult.error_description
:EndIf
```

### Example 3: Authorization Code + PKCE Flow

```apl
⍝ Create OAuth client
config←(
    client: (
        id: 'YOUR_CLIENT_ID'
        usePKCE: 1
    )
    auth: (
        authUrl: 'https://auth.example.com/oauth/authorize'
        tokenUrl: 'https://auth.example.com/oauth/token'
    )
)
client ← SOAC.New config

⍝ Generate authorization URL (PKCE parameters added automatically)
authUrl ← client.GenerateAuthorizationUrl ''

⍝ Wait for callback
result ← client.WaitForCallback 8080

⍝ Exchange code for token (PKCE verifier sent automatically)
:If result.success
    tokenResult ← client.GetToken result.code
    ⍝ ... handle token ...
:EndIf
```

### Example 4: Client Credentials Flow

```apl
⍝ Create OAuth client for client credentials
client ← SOAC.NewClientCredentials 'my-client-id' 'my-client-secret' 'https://auth.example.com/oauth/token' 'api:read api:write'

⍝ Get token directly (no user interaction)
tokenResult ← client.GetClientCredentialsToken ''

:If tokenResult.success
    ⎕ ← 'Access Token: ', tokenResult.access_token
    ⎕ ← 'Expires in: ', tokenResult.expires_in
:Else
    ⎕ ← 'Error: ', tokenResult.error_description
:EndIf
```

### Example 5: Refreshing Tokens (User-Managed)

```apl
⍝ User already has a refresh token from previous auth
refreshToken ← '...'  ⍝ User loads from their storage

⍝ Create client with token endpoint
client ← SOAC.New 'my-client-id' 'https://auth.example.com/oauth/authorize' 'http://localhost:8080/callback' 'read write'
client.clientSecret ← 'my-client-secret'
client.tokenUrl ← 'https://auth.example.com/oauth/token'  ⍝ May be different from auth URL

⍝ Refresh the token
tokenResult ← client.RefreshToken refreshToken

:If tokenResult.success
    ⎕ ← 'New Access Token: ', tokenResult.access_token
    ⎕ ← 'New Refresh Token: ', tokenResult.refresh_token  ⍝ Some servers rotate refresh tokens
:Else
    ⎕ ← 'Refresh failed: ', tokenResult.error_description
:EndIf
```

### Example 6: Using Provider Helpers with JWT Validation

```apl
⍝ Create Google OAuth client using helper
client ← SOAC.ForGoogle 'my-client-id.apps.googleusercontent.com' 'my-client-secret' 'http://localhost:8080/callback' 'openid email profile'

⍝ Get authorization URL and wait for callback
authUrl ← client.GenerateAuthorizationUrl ''
⎕ ← 'Visit: ', authUrl
result ← client.WaitForCallback 8080

⍝ Exchange code for token
:If result.success
    tokenResult ← client.GetToken result.code

    :If tokenResult.success
        ⍝ Google returns JWT as id_token
        idToken ← tokenResult.id_token

        ⍝ Parse JWT to see claims
        parsed ← SOAC.ParseJWT idToken
        ⎕ ← 'User email: ', parsed.claims.email
        ⎕ ← 'User name: ', parsed.claims.name

        ⍝ Validate JWT signature
        validationResult ← SOAC.ValidateJWT idToken 'https://www.googleapis.com/oauth2/v3/certs' (⎕NS '')

        :If validationResult.success
            ⎕ ← 'JWT signature is valid'
            ⎕ ← 'Verified user: ', validationResult.claims.sub
        :Else
            ⎕ ← 'JWT validation failed: ', validationResult.error_description
        :EndIf
    :EndIf
:EndIf
```

### Example 7: GitHub OAuth (Simple)

```apl
⍝ GitHub OAuth helper - simple and clean
client ← SOAC.ForGitHub 'my-github-client-id' 'my-github-client-secret' 'http://localhost:8080/callback' 'repo user'

authUrl ← client.GenerateAuthorizationUrl ''
⎕ ← 'Authorize at: ', authUrl

result ← client.WaitForCallback 8080
:If result.success
    tokenResult ← client.GetToken result.code
    :If tokenResult.success
        ⎕ ← 'GitHub Access Token: ', tokenResult.access_token
        ⍝ Use with GitHub API...
    :EndIf
:EndIf
```

---

## Required Features & Methods

### Core Class: SOAC (Simple OAuth Client)

#### Constructor Methods
- **`New clientId authUrl redirectUri scope`** - Create authorization code flow client
  - Already exists

- **`NewClientCredentials clientId clientSecret tokenUrl scope`** - Create client credentials flow client
  - New method needed

#### Configuration Properties
- **`clientId`** - OAuth client identifier (required)
- **`clientSecret`** - OAuth client secret (optional for PKCE, required otherwise)
- **`authorizationUrl`** - Authorization endpoint URL
- **`tokenUrl`** - Token endpoint URL (if different from authorizationUrl)
- **`redirectUrl`** - Redirect URI
- **`scope`** - Space-separated scope string
- **`usePKCE`** - Boolean flag to enable PKCE (default: 0)

#### Public Methods

##### Authorization Code Flow
- **`GenerateAuthorizationUrl state`** - Generate authorization URL
  - Already implemented
  - Enhancement needed: Add PKCE parameters when `usePKCE=1`
  - Enhancement needed: Generate and store `code_verifier` internally for PKCE

- **`WaitForCallback port timeout`** - Start temporary Jarvis server and wait for OAuth callback
  - `port` - Port number to listen on
  - `timeout` - Optional timeout in seconds (default: 300 = 5 minutes)
  - Returns namespace: `{success: bool, code: string, state: string, error: string, error_description: string}`
  - New method needed

- **`GetToken code`** - Exchange authorization code for access token
  - Currently incomplete
  - Needs full implementation including POST request to token endpoint
  - Should include PKCE `code_verifier` if enabled

##### Token Management
- **`RefreshToken refreshToken`** - Exchange refresh token for new access token
  - New method needed
  - Returns namespace: `{success: bool, access_token: string, refresh_token: string, expires_in: number, token_type: string, error: string, error_description: string}`

##### Client Credentials Flow
- **`GetClientCredentialsToken scope`** - Get token using client credentials
  - New method needed
  - Optional scope parameter (uses instance scope if empty)
  - Returns namespace: `{success: bool, access_token: string, expires_in: number, token_type: string, error: string, error_description: string}`

#### Utility Methods
- **`ParseUrl url`** - Parse URL query string into namespace
  - Already implemented
  - Keep as-is

#### Internal Methods
- **`_GenerateCodeVerifier`** - Generate PKCE code verifier (random string)
- **`_GenerateCodeChallenge verifier`** - Generate PKCE code challenge (SHA256 hash)
- **`_ValidateBasicParams`** - Basic validation of required parameters
- **`_BuildTokenRequest params`** - Build token exchange request body
- **`_ParseTokenResponse response`** - Parse token response into structured namespace

---

## Error Handling

All methods that can fail return namespaces with standard fields:

### Success Response
```apl
{
  success: 1
  [... specific response fields ...]
}
```

### Error Response
```apl
{
  success: 0
  error: 'error_code'              ⍝ OAuth error code (invalid_request, invalid_grant, etc.)
  error_description: 'Human readable description'
  http_status: 400                 ⍝ HTTP status code if applicable
  raw_response: '...'              ⍝ Raw response for debugging
}
```

---

## Implementation Priority: Minimal Viable Product

### MVP Scope (Initial Implementation)

Focus on getting a working Authorization Code flow with automatic callback server. This provides immediate value and can be extended later.

**MVP Features:**
1. Authorization Code flow (without PKCE initially)
2. Automatic callback server (`WaitForCallback`)
3. Token exchange (`GetToken`)
4. Manual code entry mode
5. Basic error handling with structured responses
6. Simple usage examples

**Future Enhancements (Post-MVP):**
- PKCE support
- Client Credentials flow
- Token refresh
- JWT parsing and validation
- Provider helpers
- Advanced error handling

---

## Implementation Phases

### Phase 1: MVP - Core Authorization Code Flow
1. Implement `GetToken` method fully
   - POST request to token endpoint with form-encoded body
   - Basic authentication with client_id/client_secret
   - Parse JSON response into structured namespace
   - Error handling for failed requests and OAuth errors

2. Implement `WaitForCallback` method
   - Start temporary Jarvis server on specified port
   - Create single-use endpoint for redirect URI path
   - Handle GET request and parse authorization code from query parameters
   - Validate state parameter if provided
   - Display success/error page to user's browser
   - Return structured result
   - Clean up and stop server

3. Add `tokenUrl` property
   - Allow explicit setting
   - Auto-derive from `authorizationUrl` if not set (common pattern: replace /authorize with /token)

4. Enhance `GenerateAuthorizationUrl` (already mostly done)
   - Ensure state parameter support
   - Test with real OAuth providers

5. Write comprehensive README
   - Show basic usage example
   - Document all public methods and properties
   - Include troubleshooting tips

### Phase 2: Future - Add PKCE Support
1. Implement `_GenerateCodeVerifier` and `_GenerateCodeChallenge`
2. Store code verifier internally when `usePKCE=1`
3. Modify `GenerateAuthorizationUrl` to include PKCE parameters
4. Modify `GetToken` to include `code_verifier` in token request

### Phase 3: Future - Add Client Credentials Flow
1. Implement `NewClientCredentials` constructor
2. Implement `GetClientCredentialsToken` method
3. Test with sample OAuth server

### Phase 4: Future - Add Token Refresh
1. Implement `RefreshToken` method
2. Test refresh token rotation scenarios

### Phase 5: Future - Add JWT Support
1. Implement `ParseJWT` - Base64URL decode and parse JWT structure
2. Implement `_DecodeJWTSegment` - Base64URL decoding utility
3. Implement `_FetchJWKS` - Fetch JSON Web Key Set from endpoint
4. Implement `_ValidateJWTSignature` - Signature validation (start with RS256, expand to others)
5. Implement `_ValidateJWTClaims` - Claims validation (exp, iat, nbf, aud, iss)
6. Implement `ValidateJWT` - Full JWT validation with options
7. Add support for multiple signature algorithms (HS256, RS256, ES256, etc.)

### Phase 6: Future - Add Provider Helpers
1. Implement `ForGitHub` helper method
   - Authorization URL: `https://github.com/login/oauth/authorize`
   - Token URL: `https://github.com/login/oauth/access_token`
2. Implement `ForGoogle` helper method
   - Authorization URL: `https://accounts.google.com/o/oauth2/v2/auth`
   - Token URL: `https://oauth2.googleapis.com/token`
   - JWKS URL: `https://www.googleapis.com/oauth2/v3/certs`
3. Implement `ForMicrosoft` helper method
   - Authorization URL: `https://login.microsoftonline.com/common/oauth2/v2.0/authorize`
   - Token URL: `https://login.microsoftonline.com/common/oauth2/v2.0/token`
4. Implement `ForAuth0` helper method (requires tenant parameter)
   - Authorization URL: `https://{tenant}/authorize`
   - Token URL: `https://{tenant}/oauth/token`

### Phase 7: Future - Polish & Advanced Features
1. Add strict parameter validation (optional enhancement)
2. Add configurable timeout for `WaitForCallback` (default: 5 minutes)
3. Expand README with all flow examples (PKCE, Client Credentials, JWT)
4. Add inline documentation/comments
5. Create test scripts for common OAuth providers
6. Add examples showing JWT validation with real providers

---

## Critical Files

- **[APLSource/SOAC.aplc](APLSource/SOAC.aplc)** - Main OAuth client class (needs significant extension)
- **[APLSource/HttpCommand.dyalog](APLSource/HttpCommand.dyalog)** - HTTP library (already complete)
- **[APLSource/Jarvis.dyalog](APLSource/Jarvis.dyalog)** - Web server framework (already complete)
- **[README.md](README.md)** - Needs comprehensive usage documentation
- **[plan.md](plan.md)** - Can be removed or converted to implementation notes

### New Files Needed

- **APLSource/JWT.aplc** - JWT parsing and validation class (optional: can be part of SOAC or separate)
- **Examples/** - Directory with working examples for each provider

---

## Technical Considerations

### JWT Signature Validation

JWT signature validation requires cryptographic operations:
- **RS256/RS384/RS512** - RSA signature verification (requires RSA public key operations)
- **ES256/ES384/ES512** - ECDSA signature verification (requires elliptic curve operations)
- **HS256/HS384/HS512** - HMAC signature verification (requires HMAC-SHA operations)

**Options for implementation:**
1. Use Dyalog's built-in cryptographic functions if available
2. Use external .NET cryptographic libraries via Dyalog's .NET interface
3. Use external command-line tools (openssl, etc.) via system calls
4. Implement basic crypto in APL (possible but complex for RSA/ECDSA)

**Recommended approach:** Use .NET System.Security.Cryptography namespace through Dyalog's .NET bridge for robust, secure implementation.

### PKCE Implementation

PKCE requires SHA256 hashing and Base64URL encoding:
- **Code Verifier**: Random 43-128 character string
- **Code Challenge**: Base64URL(SHA256(code_verifier))

Can use:
- .NET System.Security.Cryptography.SHA256 for hashing
- Custom Base64URL encoding (Base64 with URL-safe characters)

---

## MVP Testing Strategy

Focus on testing the core Authorization Code flow:

1. **Manual testing with a real OAuth provider** (e.g., GitHub or Auth0)
   - Create test OAuth application
   - Test authorization URL generation
   - Test callback server receives code
   - Test token exchange succeeds
   - Verify structured error responses

2. **Test both modes:**
   - Automatic callback server mode
   - Manual code entry mode

3. **Error scenarios:**
   - Invalid client credentials
   - Expired authorization codes
   - Network failures
   - Malformed responses

## MVP Verification Checklist

Once implementation is complete, verify:

- [ ] `GenerateAuthorizationUrl` creates valid OAuth URLs with all required parameters
- [ ] `WaitForCallback` starts Jarvis server and listens on specified port
- [ ] Callback endpoint correctly parses authorization code from query string
- [ ] Browser receives success page after callback
- [ ] `GetToken` successfully exchanges code for access token
- [ ] Token response is parsed into namespace with `success`, `access_token`, `token_type`, etc.
- [ ] OAuth errors return structured error namespaces with `success=0`
- [ ] HTTP errors are handled gracefully
- [ ] Manual code entry mode works (without callback server)
- [ ] State parameter is preserved throughout flow
- [ ] README contains clear, working example
- [ ] Code has basic inline documentation

---

## Design Decisions (Finalized)

1. **OAuth 2.0 only** - No OAuth 1.0a support (legacy, too complex)
2. **Provider helpers included** - Pre-configured methods for GitHub, Google, Microsoft, Auth0, etc.
3. **Full JWT validation** - Parse and validate JWT signatures and claims
4. **5 minute default timeout** for `WaitForCallback` (user-configurable)
5. **No built-in logging** - Users can add their own if needed

---

## Additional Features

### OAuth Provider Helpers

Convenience methods for common OAuth providers with pre-configured endpoints:

```apl
⍝ GitHub OAuth
client ← SOAC.ForGitHub 'my-client-id' 'my-client-secret' 'http://localhost:8080/callback' 'repo user'

⍝ Google OAuth
client ← SOAC.ForGoogle 'my-client-id' 'my-client-secret' 'http://localhost:8080/callback' 'https://www.googleapis.com/auth/userinfo.email'

⍝ Microsoft OAuth
client ← SOAC.ForMicrosoft 'my-client-id' 'my-client-secret' 'http://localhost:8080/callback' 'User.Read'

⍝ Auth0 OAuth (requires tenant domain)
client ← SOAC.ForAuth0 'my-tenant.auth0.com' 'my-client-id' 'my-client-secret' 'http://localhost:8080/callback' 'openid profile'
```

Each helper method sets the correct `authorizationUrl` and `tokenUrl` automatically.

### JWT (JSON Web Token) Support

Many OAuth providers return access tokens as JWTs. Support for parsing and validating them:

```apl
⍝ Parse JWT without validation
claims ← SOAC.ParseJWT accessToken
⎕ ← claims.sub  ⍝ Subject (user ID)
⎕ ← claims.exp  ⍝ Expiration timestamp
⎕ ← claims.aud  ⍝ Audience

⍝ Validate JWT signature using JWKS endpoint
validationResult ← SOAC.ValidateJWT accessToken 'https://auth.example.com/.well-known/jwks.json'

:If validationResult.success
    ⎕ ← 'Token is valid'
    claims ← validationResult.claims
:Else
    ⎕ ← 'Token validation failed: ', validationResult.error_description
:EndIf

⍝ Validate JWT with additional claims checking
options ← ⎕NS ''
options.audience ← 'my-client-id'  ⍝ Verify 'aud' claim
options.issuer ← 'https://auth.example.com'  ⍝ Verify 'iss' claim
options.clockSkew ← 60  ⍝ Allow 60 seconds clock skew for exp/iat

validationResult ← SOAC.ValidateJWT accessToken jwksUrl options
```

#### JWT Methods Required

- **`ParseJWT token`** - Parse JWT header and claims (no validation)
  - Returns namespace: `{success: bool, header: namespace, claims: namespace, signature: string, error: string}`

- **`ValidateJWT token jwksUrl options`** - Validate JWT signature and claims
  - Fetches public keys from JWKS endpoint
  - Validates signature (RS256, RS384, RS512, ES256, ES384, ES512, HS256, etc.)
  - Validates standard claims (exp, iat, nbf, aud, iss)
  - Returns namespace: `{success: bool, claims: namespace, error: string, error_description: string}`

- **`_FetchJWKS url`** - Fetch JSON Web Key Set from URL
- **`_ValidateJWTSignature token jwks`** - Validate JWT signature against JWKS
- **`_ValidateJWTClaims claims options`** - Validate JWT claims (exp, aud, iss, etc.)
- **`_DecodeJWTSegment segment`** - Base64URL decode JWT segment
