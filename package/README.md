# community/mock-preset-google

A **service preset** for [`aux4/mock`](https://hub.aux4.io/package/aux4/mock) that stands up Google's OAuth auth dance on a running mock server with a single command: the token endpoint, both versions of the userinfo endpoint (bearer-gated), and Google's real `401` error envelope. Install it and `aux4 mock preset google` becomes available — the mock core stays preset-agnostic and knows nothing about Google.

This is a pure `.aux4` package (no binary, no bundle). It works entirely by contributing a `google` command into `aux4/mock`'s `mock:preset` profile via aux4's `global.aux4` profile merge.

## Installation

```bash
aux4 aux4 pkger install community/mock-preset-google
```

Installing this package pulls in `aux4/mock` as a dependency.

## Usage

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

Confirm it is installed and visible to the mock:

```bash
aux4 mock presets
```

```text
Installed presets (apply with: aux4 mock preset <service> --port <port>):
  google
```

## What it stubs

| Method + Path | Status | Behavior |
|---------------|--------|----------|
| `POST /token` | `200` | Standard OAuth token response (`access_token`, `refresh_token`, `scope`, `token_type`) |
| `GET /oauth2/v2/userinfo` | `200` | Identity JSON — **only** when `Authorization: Bearer <token>` is present |
| `GET /oauth2/v2/userinfo` | `401` | Google `UNAUTHENTICATED` envelope — fallback when no bearer token |
| `GET /oauth2/v3/userinfo` | `200` | Same identity, gated the same way (Google exposes userinfo at v2 and v3) |
| `GET /oauth2/v3/userinfo` | `401` | Same `401` envelope fallback |

Each userinfo endpoint is registered as **two** stubs on the same path: a happy path gated on `Authorization: Bearer *` (any token) and a bare fallback returning the `401`. A request with a bearer token gets the identity; one without falls through to Google's real unauthenticated error:

```bash
# with a bearer token → 200 identity
curl -s http://localhost:8080/api/oauth2/v2/userinfo -H 'Authorization: Bearer x'
# {"email":"sally@example.com","email_verified":true,"family_name":"Example","given_name":"Sally","name":"Sally","picture":"https://example.com/photo.jpg","sub":"1234567890"}

# without one → 401 Google error envelope
curl -s http://localhost:8080/api/oauth2/v2/userinfo
# {"error":{"code":401,"message":"Request is missing required authentication credential...","status":"UNAUTHENTICATED"}}
```

The preset is **additive** — it only calls `aux4 mock stub`, so it never clears existing stubs. Layer the business endpoints your test exercises on top:

```bash
aux4 mock preset google --port 8080
aux4 mock stub --port 8080 --method POST \
  --path /gmail/v1/users/me/messages/send \
  --status 200 --body '{"id":"msg_1","labelIds":["SENT"]}'
```

## Overrides

All optional, with sane defaults, so a test can assert a known identity:

```bash
aux4 mock preset google --port 8080 \
  --accessToken TOK123 --email sally@corp.io --user "Sally"
```

- `--accessToken` — token returned by `POST /token` (default `mock-access-token`)
- `--email` — email in the userinfo response (default `sally@example.com`)
- `--user` — display/given name in the userinfo response (default `Sally`)

Also accepts `--name` and `--stateDir` to address a server the same way every other `aux4/mock` command does (precedence: `--stateDir` > `--name` > `--port`).

## Base URLs and paths

The preset stubs Google's real paths (`/token`, `/oauth2/v2/userinfo`, `/oauth2/v3/userinfo`). Point the code-under-test's base URL at `http://localhost:<port>/api` and let it append the real path — `aux4/mock` strips the `/api` mount prefix before matching.

## Using in tests

`aux4/mock` and this preset are plain CLIs, so they drop straight into an [`aux4/test`](https://hub.aux4.io/package/aux4/test) `.test.md`. In CI, declare both packages on the test job:

```yaml
- uses: aux4/action@v1
  with:
    command: test
    packages: aux4/mock,community/mock-preset-google
```

## This package as a template

`community/mock-preset-google` is the reference implementation for writing your own `community/mock-preset-<service>` package. The pattern — depend on `aux4/mock`, re-declare the `mock` profile with a `preset` routing command, and add a `<service>` command under `mock:preset` whose `execute` is a sequence of `aux4 mock stub` calls — is documented in the [aux4/mock README](https://hub.aux4.io/package/aux4/mock) under "Writing your own preset package".
