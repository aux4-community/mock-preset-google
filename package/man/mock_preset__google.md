#### Description

The `google` command registers Google's OAuth auth dance on a running `aux4/mock` server in one shot, so a test can stand up the boilerplate and then stub only the business endpoints it cares about. It is contributed to `aux4/mock`'s `mock:preset` profile by installing this package — the mock core ships no presets and knows nothing about Google.

It registers five stubs by calling `aux4 mock stub` once per endpoint:

- **`POST /token`** → `200` standard OAuth token response (`access_token`, `refresh_token`, `scope`, `token_type`).
- **`GET /oauth2/v2/userinfo`** → `200` identity JSON, gated on `Authorization: Bearer *`; plus a bare `401` fallback with Google's `UNAUTHENTICATED` envelope.
- **`GET /oauth2/v3/userinfo`** → the same identity + `401` fallback (Google exposes userinfo at both v2 and v3).

Each userinfo endpoint is two stubs on the same path: a happy path gated on any bearer token and a fallback returning the real `401`. So a request with a bearer token gets the identity; one without gets Google's error shape.

The preset is **additive** — it only calls `mock stub`, so it never clears existing stubs and layers cleanly with your own business stubs.

#### Usage

```bash
aux4 mock preset google [--port 7070] [--name <handle>] [--stateDir <dir>] \
  [--accessToken <token>] [--email <email>] [--user <name>]
```

--port          Port of the running mock server, derives stateDir (default: 7070)
--name          Address the server by handle instead of port (precedence over --port)
--stateDir      Explicit state directory (highest precedence)
--accessToken   Token returned by POST /token (default: mock-access-token)
--email         Email in the userinfo response (default: sally@example.com)
--user          Display/given name in the userinfo response (default: Sally)

#### Example

```bash
aux4 mock start --port 8080
aux4 mock preset google --port 8080
```

```text
stub POST /token -> 200
stub GET /oauth2/v2/userinfo -> 200
stub GET /oauth2/v2/userinfo -> 401
stub GET /oauth2/v3/userinfo -> 200
stub GET /oauth2/v3/userinfo -> 401
```

```bash
curl -s http://localhost:8080/api/oauth2/v2/userinfo -H 'Authorization: Bearer x'
# {"email":"sally@example.com","email_verified":true,...,"sub":"1234567890"}

curl -s http://localhost:8080/api/oauth2/v2/userinfo
# {"error":{"code":401,"message":"Request is missing required authentication credential...","status":"UNAUTHENTICATED"}}
```
