# SOAC Tests

This directory contains tests for the SimpleOAuthClient (SOAC) library.

## Setting Up Test Credentials

### 1. Create Your Credentials File

Copy the example credentials file:

```bash
cp exampleCredentials.apla credentials.apla
```

### 2. Fill In Your OAuth Credentials

Edit `credentials.apla` and add your actual OAuth credentials:

```apl
(
    github: (
        clientId: 'your-actual-github-client-id'
        clientSecret: 'your-actual-github-client-secret'
    )
    google: (
        clientId: 'your-actual-google-client-id.apps.googleusercontent.com'
        clientSecret: 'your-actual-google-client-secret'
    )
    microsoft: (
        clientId: 'your-actual-microsoft-client-id'
        clientSecret: 'your-actual-microsoft-client-secret'
    )
)
```

### 3. Obtain OAuth Credentials

#### GitHub

1. Go to https://github.com/settings/developers
2. Click "New OAuth App"
3. Fill in:
   - Application name: "SOAC Test"
   - Homepage URL: http://localhost
   - Authorization callback URL: http://localhost:3000/auth
4. Copy the Client ID and Client Secret

#### Google

1. Go to https://console.cloud.google.com/
2. Create a new project or select existing
3. Enable OAuth consent screen
4. Go to Credentials > Create Credentials > OAuth 2.0 Client ID
5. Application type: Web application or Desktop app
6. Add authorized redirect URI: http://localhost:3000/auth
7. Copy the Client ID and Client Secret

#### Microsoft

1. Go to https://portal.azure.com/
2. Navigate to Azure Active Directory > App registrations
3. Click "New registration"
4. Add redirect URI: http://localhost:3000/auth
5. Go to Certificates & secrets > New client secret
6. Copy the Application (client) ID and secret value

## Running Tests

### Using the Test Runner

The `TestRunner.aplf` namespace provides a simple test framework:

```apl
⍝ Create a test namespace
testNs←⎕NS''

⍝ Define test functions (must start with 'test')
testNs.testExample←{
    ⍝ Test code here
    result←1+1
    TestRunner.AssertEqual result 2
    1  ⍝ Return 1 for success
}

⍝ Run all tests
results←TestRunner.RunTests testNs
```

### Available Test Files

- **`test.aplf`** - General OAuth flow tests
- **`ghtest.aplf`** - GitHub-specific OAuth tests
- **`deviceFlowTest.aplf`** - Device code flow tests

### Running Existing Tests

```apl
]link.create # ./APLSource
```

## Test Framework API

### TestRunner.RunTests

Runs all test functions in a namespace:

```apl
results←TestRunner.RunTests testNamespace
```

Returns a namespace with:
- `passed` - Number of passed tests
- `failed` - Number of failed tests
- `total` - Total number of tests
- `success` - Boolean indicating all tests passed
- `tests` - Array of individual test results

### Assertion Functions

#### Assert

Basic assertion:
```apl
TestRunner.Assert condition
TestRunner.Assert (result>0) 'Value must be positive'
```

#### AssertEqual

Assert values are equal:
```apl
TestRunner.AssertEqual actual expected
```

#### AssertNotEqual

Assert values are different:
```apl
TestRunner.AssertNotEqual actual unexpected
```

#### AssertSuccess

Assert namespace has `success=1`:
```apl
result←client.GetToken args
TestRunner.AssertSuccess result
```

#### AssertFail

Assert namespace has `success=0`:
```apl
result←client.GetToken invalidArgs
TestRunner.AssertFail result
```

#### AssertContains

Assert string contains substring:
```apl
TestRunner.AssertContains url 'github.com'
```

#### AssertType

Assert value has expected type:
```apl
TestRunner.AssertType value 9  ⍝ Namespace
TestRunner.AssertType value 2  ⍝ Character
```

## Writing New Tests

### Test Function Convention

Test functions must:
1. Start with `test` prefix
2. Take one argument (typically ignored)
3. Return 1 for success, 0 for failure
4. Use assertions to validate behavior

### Example Test

```apl
testGetToken←{
    ⍝ Test token retrieval
    config←(
        clientId: credentials.github.clientId
        clientSecret: credentials.github.clientSecret
        authUrl: 'https://github.com/login/oauth/authorize'
        tokenUrl: 'https://github.com/login/oauth/access_token'
    )
    client←SOAC.New config

    ⍝ Test requires manual OAuth flow completion
    ⍝ This is a placeholder for demonstration

    1  ⍝ Success
}
```

## Best Practices

### Credential Management

1. **Never hardcode credentials** - Always load from `credentials.apla`
2. **Use environment-specific configs** - Create different credential files for dev/staging/prod
3. **Rotate credentials regularly** - Especially after any suspected exposure
4. **Use least privilege** - Request only the OAuth scopes you need
5. **Verify .gitignore** - Always check `git status` before committing

### Test Organization

1. **Group related tests** - Keep authorization code, device flow, etc. separate
2. **Use descriptive names** - `testGitHubAuthFlow` not `test1`
3. **Test error cases** - Don't just test happy paths
4. **Keep tests isolated** - Each test should be independent
5. **Clean up resources** - Stop servers, deallocate threads, etc.

### Manual vs Automated Tests

Some OAuth tests require manual intervention:
- Opening browser for authorization
- Entering device codes
- Approving permissions

For automated CI/CD:
- Use mock OAuth servers
- Test with expired/invalid tokens
- Validate URL generation and parsing
- Test error handling

## Troubleshooting

### Credential File Not Found

Error: `FILE ERROR` when loading credentials

**Solution**: Create `credentials.apla` from `exampleCredentials.apla`

### Invalid Credentials

Error: `HTTP Error: 401` or `invalid_client`

**Solution**:
- Verify credentials are correct
- Check OAuth app configuration (redirect URIs, etc.)
- Ensure OAuth app is not suspended

### Redirect URI Mismatch

Error: `redirect_uri_mismatch`

**Solution**:
- Add `http://localhost:3000/auth` to authorized redirect URIs
- Use exact port number specified in your OAuth app config

### Port Already in Use

Error: `Jarvis failed to init: Address already in use`

**Solution**:
- Change port number in test
- Stop other applications using the port
- Kill existing Jarvis instances

## Security Checklist

Before committing code:

- [ ] `credentials.apla` is not staged (`git status`)
- [ ] No credentials in test files
- [ ] No credentials in comments or debug statements
- [ ] `.gitignore` patterns are working
- [ ] No actual tokens in commit history

## Additional Resources

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [OAuth 2.0 for Native Apps (RFC 8252)](https://tools.ietf.org/html/rfc8252)
- [PKCE (RFC 7636)](https://tools.ietf.org/html/rfc7636)
- [Device Authorization Grant (RFC 8628)](https://tools.ietf.org/html/rfc8628)
