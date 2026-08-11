# community/mock-preset-google-auth

Google auth preset for [aux4/mock](https://hub.aux4.io/r/public/packages/aux4/mock).

Every Google API test needs the same two things before it can call anything: a token file to pass
to `--tokenFile`, and an OAuth surface to answer the auth dance. This preset supplies both in one
command, so a test can get to the API call it actually cares about.

It is the base every other Google preset builds on — `community/mock-preset-google-calendar`,
`-gmail`, `-drive` and friends all depend on it.

## Installation

```bash
aux4 aux4 pkger install community/mock-preset-google-auth
```

## Usage

Start a mock, then apply the preset:

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

That writes `google-token.json` in the working directory and registers the endpoints. A Google
client now runs entirely offline:

```bash
aux4 google calendar events list --tokenFile google-token.json --apiUrl http://127.0.0.1:18901/api
```

### What it registers

| Method | Path | Behaviour |
|---|---|---|
| `POST` | `/token` | returns an OAuth token response |
| `GET` | `/oauth2/v2/userinfo` | the identity with a bearer, `401` without |
| `GET` | `/oauth2/v3/userinfo` | the same, at v3 |

The `401` is Google's real `UNAUTHENTICATED` envelope, so a client's error handling is exercised
rather than a generic failure.

### The token file

The written file carries the fields a Google package expects — `clientId`, `clientSecret`,
`authUrl`, `tokenUrl`, `scopes`, `accessToken`, `refreshToken`, `expiresAt`.

`tokenUrl` points back at this mock, so if a client decides to refresh, the refresh is served
locally instead of reaching Google. `expiresAt` defaults to the far future so nothing refreshes at
all; set it in the past to exercise the refresh path deliberately:

```bash
aux4 mock preset google auth --port 18901 --expiresAt 2020-01-01T00:00:00Z
```

### Options

```bash
aux4 mock preset google auth --port 18901 \
  --accessToken TOK123 --email devon@corp.io --user Devon \
  --scopes https://www.googleapis.com/auth/calendar
```

| Option | Description | Default |
|---|---|---|
| `--port` | Port of the running mock server | `7070` |
| `--name` | Stable handle of the server, instead of `--port` | |
| `--stateDir` | Explicit state directory | |
| `--tokenFile` | Where to write the token file | `google-token.json` |
| `--accessToken` | Token returned by `/token` and written to the file | `mock-access-token` |
| `--refreshToken` | Refresh token written to the file | `mock-refresh-token` |
| `--expiresAt` | Token expiry written to the file | `2099-12-31T23:59:59Z` |
| `--scopes` | Scopes written to the file | `.../auth/userinfo.email` |
| `--clientId` | OAuth client id written to the file | `mock-client-id` |
| `--clientSecret` | OAuth client secret written to the file | `mock-client-secret` |
| `--email` | Email returned by userinfo | `sally@example.com` |
| `--user` | Display name returned by userinfo | `Sally` |
| `--authUrl` | Authorization URL written to the file | Google's real one |
| `--tokenUrl` | Token URL written to the file | this mock's `/token` |

For a field the flags do not reach, replace a whole body: `--tokenFileBody`, `--tokenBody`,
`--identityBody`, and `--unauthBody` each take raw JSON.

### Layering your own stubs

Apply the preset, then add the API under test with `aux4 mock stub` — a later stub for the same
method and path wins, so any endpoint can be bent without hand-writing the rest:

```bash
aux4 mock preset google auth --port 18901
aux4 mock stub --port 18901 --method POST --path /gmail/v1/users/me/messages/send \
  --status 200 --body '{"id":"msg_1","labelIds":["SENT"]}'
```

## See also

- [community/mock-preset-google-calendar](https://hub.aux4.io/r/public/packages/community/mock-preset-google-calendar) — the Calendar API on top of this
- [aux4/mock](https://hub.aux4.io/r/public/packages/aux4/mock) — the mock server itself
