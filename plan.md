// Client-side pseudo-code to initiate the flow
GET https://authorization-server.com?
  client_id=YOUR_CLIENT_ID&
  response_type=code&
  redirect_uri=YOUR_REDIRECT_URI&
  scope=REQUESTED_SCOPE&
  state=SECURE_RANDOM_STATE // Used for CSRF protection

// User interacts

// Server-side (your application's redirect handler) receives the code
https://example-app.com/cb?
  code=AUTH_CODE_FROM_SERVER&
  state=RECEIVED_STATE

// Server-side pseudo-code to exchange the code for a token
POST /oauth/token HTTP/1.1
Host: authorization-server.com
Content-Type: application/x-www-form-urlencoded

code=AUTH_CODE_FROM_SERVER&
grant_type=authorization_code&
redirect_uri=YOUR_REDIRECT_URI&
client_id=YOUR_CLIENT_ID&
client_secret=YOUR_CLIENT_SECRET


// Server-side pseudo-code to access a resource
GET https://resource-server.com HTTP/1.1
Authorization: Bearer ACCESS_TOKEN_FROM_SERVER