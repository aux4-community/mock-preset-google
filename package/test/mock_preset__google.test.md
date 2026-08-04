# community/mock-preset-google

Proves that installing this package makes `aux4 mock preset google` available purely
through aux4's `global.aux4` profile merge — the `aux4/mock` core has zero knowledge of
Google. The preset stands up Google's auth dance (token endpoint + bearer-gated userinfo at
v2 and v3 + the real `401` envelope) with one command, and the stub set it applies is
byte-identical to what the old built-in `mock preset google` produced.

The server is started inside the first test's `execute` block (not `beforeAll`, which the
harness would kill) and torn down in `afterAll` on a unique port. It is a detached process,
so it survives across the sibling `##` scenarios; each override scenario clears the stub
store first. All checks use plain `curl` with no external network.

```afterAll
aux4 mock stop --port 18781 2>/dev/null
pkill -f "18781" 2>/dev/null
true
```

## preset google via install alone

### should list google under mock presets

```execute
aux4 mock presets
```

```expect:partial
google
```

### should start the server and apply the google preset

```execute
aux4 mock start --port 18781
sleep 1
aux4 mock preset google --port 18781
```

```expect:partial
stub POST /token -> 200
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

## overrides: pin a known token and identity

### should re-apply the google preset with overrides

```execute
aux4 mock reset --port 18781 --stubs true
aux4 mock preset google --port 18781 --accessToken TOK123 --email devon@corp.io --user "Devon"
```

```expect:partial
stub POST /token -> 200
```

### should return the overridden token

```execute
curl -s -X POST http://localhost:18781/api/token
```

```expect
{"access_token":"TOK123","expires_in":3599,"refresh_token":"mock-refresh-token","scope":"https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile","token_type":"Bearer"}
```

### should return the overridden identity with a bearer token

```execute
curl -s http://localhost:18781/api/oauth2/v2/userinfo -H 'Authorization: Bearer devon-token'
```

```expect
{"email":"devon@corp.io","email_verified":true,"family_name":"Example","given_name":"Devon","name":"Devon","picture":"https://example.com/photo.jpg","sub":"1234567890"}
```

## layering: a preset coexists with a later business stub

### should apply the preset then a gmail business stub

```execute
aux4 mock reset --port 18781 --stubs true
aux4 mock preset google --port 18781
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
