# Authorization

The sandbox uses OAuth 2.1 with dynamic client registration and PKCE. Every request to `/mcp` needs a bearer access token. A token is issued only after a person enters the sandbox passcode on a consent page. Your CapitalSnapshot contact gives you the passcode privately.

All URLs below are on `https://demo.capitalsnapshot.com`.

## 1. Discovery

An unauthenticated request to `/mcp` returns HTTP 401:

```
HTTP/2 401
content-type: application/json; charset=utf-8
www-authenticate: Bearer resource_metadata="https://demo.capitalsnapshot.com/.well-known/oauth-protected-resource/mcp"

{"error":"Unauthorized"}
```

A request with an invalid or expired token also gets HTTP 401 and the same body, but `www-authenticate` carries `error="invalid_token"` (RFC 6750 section 3.1):

```
www-authenticate: Bearer error="invalid_token", error_description="The access token is invalid or expired", resource_metadata="https://demo.capitalsnapshot.com/.well-known/oauth-protected-resource/mcp"
```

Use the `error="invalid_token"` attribute to tell "refresh or re-authorize" (a token was sent and rejected) from "start authorization" (no token was sent).

### Protected resource metadata (RFC 9728)

`GET /.well-known/oauth-protected-resource/mcp`

```json
{
  "resource": "https://demo.capitalsnapshot.com/mcp",
  "authorization_servers": ["https://demo.capitalsnapshot.com/"],
  "resource_name": "CapitalSnapshot Investor Agent"
}
```

The metadata is served only at the path form above (with `/mcp` appended). `GET /.well-known/oauth-protected-resource` without the suffix returns 404.

### Authorization server metadata (RFC 8414)

`GET /.well-known/oauth-authorization-server`

```json
{
  "issuer": "https://demo.capitalsnapshot.com/",
  "authorization_endpoint": "https://demo.capitalsnapshot.com/authorize",
  "response_types_supported": ["code"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint": "https://demo.capitalsnapshot.com/token",
  "token_endpoint_auth_methods_supported": ["client_secret_post", "none"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "registration_endpoint": "https://demo.capitalsnapshot.com/register"
}
```

The issuer ends with a slash. Compare it exactly as served. The metadata lists no scopes and no revocation endpoint, and the flow needs no `scope` parameter.

## 2. Dynamic client registration (RFC 7591)

`POST /register` with a JSON body. Register a public client:

```json
{
  "client_name": "Example Capital research agent",
  "redirect_uris": ["http://127.0.0.1:53682/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

The response is HTTP 201 with `cache-control: no-store`. It echoes your metadata and adds `client_id` (a UUID) and `client_id_issued_at`. A public client gets no secret.

`client_name` matters: the consent page shows it, and founders see it as the name of the agent that viewed their company or left feedback. Choose a name that tells the founder who you are.

Register once and keep the `client_id`. Registrations survive the nightly sandbox reset.

## 3. Authorization request, PKCE S256

Create a random `code_verifier`. Set `code_challenge = BASE64URL(SHA256(code_verifier))`. Then send the user's browser to:

```
GET /authorize
  ?response_type=code
  &client_id=<client_id>
  &redirect_uri=<a registered redirect URI>
  &code_challenge=<code_challenge>
  &code_challenge_method=S256
  &state=<random>
  &resource=https://demo.capitalsnapshot.com/mcp
```

- `S256` is the only supported challenge method.
- `resource` (RFC 8707) names the MCP endpoint. Always send it.
- `redirect_uri` must be one of the URIs you registered.

## 4. Consent with the passcode

`/authorize` returns an HTML consent page. It names your client and asks for the access passcode. A person enters the passcode and approves. The server then redirects to your `redirect_uri` with `code` and your `state`:

```
HTTP/2 302
location: http://127.0.0.1:53682/callback?code=csc_<hex>&state=<your state>
```

Check that `state` matches the value you sent.

- The consent page is valid for 10 minutes. After that, start again from `/authorize`.
- A wrong passcode shows the form again with "Wrong passcode. Try again." (HTTP 401).
- Repeated wrong passcodes lock consent: 5 per client, 10 per IP address (IPv6: per /64) and 1000 server-wide, each counted over a sliding 15 minutes. The locked consent page answers HTTP 429 with `Retry-After`, and says which limit applied and for how long. Wait for `Retry-After`. Do not retry in a loop.
- Never put the passcode in code, configuration, logs or a repository. A person types it into the consent page.

## 5. Token request

`POST /token` with `content-type: application/x-www-form-urlencoded`:

```
grant_type=authorization_code
&client_id=<client_id>
&code=<code>
&code_verifier=<code_verifier>
&redirect_uri=<the same redirect URI>
&resource=https://demo.capitalsnapshot.com/mcp
```

Response, HTTP 200 (token values elided):

```json
{
  "access_token": "csa_...",
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "csr_..."
}
```

The response has no `scope` field. `token_type` is lowercase `bearer`. Send it as `Authorization: Bearer <access_token>`.

## 6. Lifetimes and prefixes

| Item | Prefix | Lifetime |
|---|---|---|
| Authorization code | `csc_` | 10 minutes, single use |
| Access token | `csa_` | 1 hour (`expires_in` 3600) |
| Refresh token | `csr_` | 30 days from the moment that refresh token was issued |
| Consent page | | 10 minutes |

Treat tokens as opaque strings. The prefixes are stable, but do not parse what follows them.

## 7. Refresh and rotation

`POST /token`, form encoded:

```
grant_type=refresh_token
&client_id=<client_id>
&refresh_token=<current refresh token>
```

Each refresh returns a new access token and a **new** refresh token, valid for 30 days. The refresh token you presented is now spent.

Presenting a spent refresh token again returns:

```
HTTP/2 400
{"error":"invalid_grant","error_description":"Invalid or expired refresh token"}
```

It also revokes the whole chain: every access and refresh token descended from that consent stops working, including the newest ones. The client must go through consent again (section 4).

What this means for your client:

- Store the new refresh token as soon as you receive it, before you use the new access token.
- Allow only one refresh at a time per client. Two workers refreshing with the same token revoke the chain.
- After `invalid_grant`, do not retry with the same token. Start a new authorization.
- An expired or unknown refresh token gets the same `invalid_grant` response.

## 8. Endpoint rate limits

The OAuth endpoints use the default limits of the MCP TypeScript SDK the server is built on:

| Endpoint | Limit |
|---|---|
| `POST /register` | 20 requests per hour |
| `/authorize` | 100 requests per 15 minutes |
| `POST /token` | 50 requests per 15 minutes |

`/authorize` responses carry `ratelimit-policy`, `ratelimit-limit`, `ratelimit-remaining` and `ratelimit-reset` headers (for example `ratelimit-policy: 100;w=900`). Over a limit, the endpoint returns HTTP 429. Wait for the window to pass. A well-behaved client registers once and refreshes about once an hour, so it never gets near these limits.

## 9. Using Claude instead of your own client

Claude handles all of the above for you. Add a custom connector with the URL `https://demo.capitalsnapshot.com/mcp`. Claude registers itself, then opens the consent page. Enter the passcode there.
