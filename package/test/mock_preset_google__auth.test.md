# community/mock-preset-google-auth

Proves that installing this package makes `aux4 mock preset google auth` available purely through
aux4's `global.aux4` profile merge — the `aux4/mock` core has zero knowledge of Google. The preset
does the two things every Google API test needs: it writes a token file to pass to `--tokenFile`,
and it stands up Google's auth dance (token endpoint + bearer-gated userinfo at v2 and v3 + the
real `401` envelope).

The server is started inside the first test's `execute` block (not `beforeAll`, which the harness
would kill) and torn down in `afterAll` on a unique port. It is a detached process, so it survives
across the sibling `##` scenarios; each override scenario clears the stub store first. All checks
use plain `curl` with no external network.

```afterAll
aux4 mock stop --port 18781 2>/dev/null
pkill -f "18781" 2>/dev/null
rm -f google-token.json custom-token.json
true
```

## preset google auth via install alone

### should start the server and apply the google auth preset

```execute
aux4 mock start --port 18781
sleep 1
aux4 mock preset google auth --port 18781
```

```expect:partial
stub POST /token -> 200
```

### should write a token file

```execute
cat google-token.json
```

```expect
{"clientId":"mock-client-id","clientSecret":"mock-client-secret","authUrl":"https://accounts.google.com/o/oauth2/v2/auth","tokenUrl":"http://127.0.0.1:18781/api/token","scopes":"https://www.googleapis.com/auth/userinfo.email","accessToken":"mock-access-token","refreshToken":"mock-refresh-token","expiresAt":"2099-12-31T23:59:59Z"}
```

### should point the token file at this mock so a refresh stays offline

```execute
cat google-token.json
```

```expect:partial
"tokenUrl":"http://127.0.0.1:18781/api/token"
```

### should return an OAuth token from the token endpoint

```execute
curl -s -X POST http://localhost:18781/api/token
```

```expect
{"access_token":"mock-access-token","expires_in":3599,"refresh_token":"mock-refresh-token","scope":"https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile","token_type":"Bearer"}
```

### should return the identity when a bearer token is present

```execute
curl -s -w "\n%{http_code}" http://localhost:18781/api/oauth2/v2/userinfo -H 'Authorization: Bearer sally-token'
```

```expect
{"email":"sally@example.com","email_verified":true,"family_name":"Example","given_name":"Sally","name":"Sally","picture":"https://example.com/photo.jpg","sub":"1234567890"}
200
```

### should return a 401 Google error envelope without a bearer token

```execute
curl -s -w "\n%{http_code}" http://localhost:18781/api/oauth2/v2/userinfo
```

```expect
{"error":{"code":401,"message":"Request is missing required authentication credential. Expected OAuth 2 access token, login cookie or other valid authentication credential.","status":"UNAUTHENTICATED"}}
401
```

### should gate the v3 userinfo endpoint the same way

```execute
curl -s -w "\n%{http_code}" http://localhost:18781/api/oauth2/v3/userinfo
```

```expect
{"error":{"code":401,"message":"Request is missing required authentication credential. Expected OAuth 2 access token, login cookie or other valid authentication credential.","status":"UNAUTHENTICATED"}}
401
```

## overrides: pin a known token, identity, and token file

### should re-apply the preset with overrides

```execute
aux4 mock reset --port 18781 --stubs true
aux4 mock preset google auth --port 18781 --accessToken TOK123 --email devon@corp.io --user "Devon" --tokenFile custom-token.json --scopes https://www.googleapis.com/auth/calendar
```

```expect:partial
stub POST /token -> 200
```

### should write the token file where asked

```execute
cat custom-token.json
```

```expect:partial
"accessToken":"TOK123"
```

### should write the requested scopes

```execute
cat custom-token.json
```

```expect:partial
"scopes":"https://www.googleapis.com/auth/calendar"
```

### should return the overridden token

```execute
curl -s -X POST http://localhost:18781/api/token
```

```expect:partial
"access_token":"TOK123"
```

### should return the overridden identity with a bearer token

```execute
curl -s http://localhost:18781/api/oauth2/v2/userinfo -H 'Authorization: Bearer devon-token'
```

```expect
{"email":"devon@corp.io","email_verified":true,"family_name":"Example","given_name":"Devon","name":"Devon","picture":"https://example.com/photo.jpg","sub":"1234567890"}
```

## an expired token, for exercising the refresh path

### should write an expiry in the past when asked

```execute
aux4 mock reset --port 18781 --stubs true
aux4 mock preset google auth --port 18781 --expiresAt 2020-01-01T00:00:00Z
cat google-token.json
```

```expect:partial
"expiresAt":"2020-01-01T00:00:00Z"
```

### should still serve the token endpoint the refresh would call

```execute
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:18781/api/token
```

```expect
200
```

## layering: a preset coexists with a later business stub

### should apply the preset then a gmail business stub

```execute
aux4 mock reset --port 18781 --stubs true
aux4 mock preset google auth --port 18781
aux4 mock stub --port 18781 --method POST --path /gmail/v1/users/me/messages/send --status 200 --body '{"id":"msg_1","labelIds":["SENT"]}'
```

```expect:partial
stub POST /gmail/v1/users/me/messages/send -> 200
```

### should still serve the preset token endpoint

```execute
curl -s -X POST http://localhost:18781/api/token
```

```expect
{"access_token":"mock-access-token","expires_in":3599,"refresh_token":"mock-refresh-token","scope":"https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile","token_type":"Bearer"}
```

### should serve the layered gmail business endpoint

```execute
curl -s -w "\n%{http_code}" -X POST http://localhost:18781/api/gmail/v1/users/me/messages/send -H 'Authorization: Bearer sally-token'
```

```expect
{"id":"msg_1","labelIds":["SENT"]}
200
```
