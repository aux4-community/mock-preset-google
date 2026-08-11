#### Description

The `auth` command sets up everything a Google API call needs before it can reach the API itself.
It writes a token file to pass to a Google package's `--tokenFile`, and registers Google's auth
surface on a running `aux4/mock` server: the OAuth token endpoint, and the userinfo endpoint at
both v2 and v3, gated on a bearer token and returning Google's real `UNAUTHENTICATED` envelope
when one is missing.

It is the base the other Google presets build on. Apply it first, then apply the service preset —
`aux4 mock preset google calendar`, for instance — which registers only that API's endpoints.

The token file is written to the working directory as `google-token.json` unless `--tokenFile`
says otherwise, and carries the fields a Google package reads: `clientId`, `clientSecret`,
`authUrl`, `tokenUrl`, `scopes`, `accessToken`, `refreshToken`, and `expiresAt`. Its `tokenUrl`
points back at this mock, so a client that decides to refresh is served locally rather than
reaching Google. `expiresAt` defaults far into the future, so nothing refreshes; set it in the past
to exercise the refresh path on purpose.

When a flag does not reach the field you need, replace a whole body with `--tokenFileBody`,
`--tokenBody`, `--identityBody`, or `--unauthBody`.

#### Usage

```bash
aux4 mock preset google auth [--port <port>] [--tokenFile <path>] [--accessToken <token>]
```

--port          Port of the running mock server, which derives the state directory (default: `7070`)
--name          Stable handle of the server, used instead of `--port`
--stateDir      Explicit state directory, taking precedence over `--port` and `--name`
--tokenFile     Path of the token file to write (default: `google-token.json`)
--accessToken   Token returned by the token endpoint and written to the file (default: `mock-access-token`)
--refreshToken  Refresh token written to the file (default: `mock-refresh-token`)
--expiresAt     Token expiry written to the file (default: `2099-12-31T23:59:59Z`)
--scopes        Scopes written to the file (default: `https://www.googleapis.com/auth/userinfo.email`)
--clientId      OAuth client id written to the file (default: `mock-client-id`)
--clientSecret  OAuth client secret written to the file (default: `mock-client-secret`)
--email         Email returned by the userinfo endpoint (default: `sally@example.com`)
--user          Display name returned by the userinfo endpoint (default: `Sally`)
--authUrl       Authorization URL written to the file. Never called during a mock run
--tokenUrl      Token URL written to the file (default: this mock's `/token`)

#### Example

```bash
aux4 mock start --port 18901
aux4 mock preset google auth --port 18901
```

```text
stub POST /token -> 200
stub GET /oauth2/v2/userinfo -> 200
stub GET /oauth2/v2/userinfo -> 401
stub GET /oauth2/v3/userinfo -> 200
stub GET /oauth2/v3/userinfo -> 401
token file google-token.json
```

Pin the credentials and the identity when a test asserts on them:

```bash
aux4 mock preset google auth --port 18901 --accessToken TOK123 --email devon@corp.io --user Devon
```

```bash
curl -s http://localhost:18901/api/oauth2/v3/userinfo -H 'Authorization: Bearer TOK123'
```

```text
{"email":"devon@corp.io","email_verified":true,"family_name":"Example","given_name":"Devon","name":"Devon","picture":"https://example.com/photo.jpg","sub":"1234567890"}
```
