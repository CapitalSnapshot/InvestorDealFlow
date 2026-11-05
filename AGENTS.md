# AGENTS.md

Instructions for a coding agent building a client for the CapitalSnapshot sandbox, Access stage. This file is self-contained. The files in `docs/` give more detail.

## Ground rules

- Use only `https://demo.capitalsnapshot.com`. Use only the sandbox endpoint in this repository.
- A person types the passcode into the consent page in a browser. Never ask for the passcode, and never put it in code, configuration, environment files, logs, tests or commits.
- Never print, log or commit access tokens (`csa_...`), refresh tokens (`csr_...`) or authorization codes (`csc_...`). Store tokens where the user's other secrets live.
- All sandbox data is fictional and resets every night at 03:30 UTC.

## 1. Endpoint and transport

- MCP endpoint: `https://demo.capitalsnapshot.com/mcp`
- Transport: MCP Streamable HTTP.
- Protocol version: `2025-06-18` or `2025-11-25`. The server returns the version you send.
- Send a User-Agent header that names your client, for example my-agent/1.0. Requests with some generic library user agents are currently refused. This applies to every request: `/mcp`, discovery, `/register`, `/authorize` and `/token`.
- If an MCP SDK is available in your language, use its Streamable HTTP client and its OAuth support. Otherwise follow sections 2 and 3 by hand.

## 2. Authorization

1. **Discover.** `POST /mcp` without a token returns 401 with
   `WWW-Authenticate: Bearer resource_metadata="https://demo.capitalsnapshot.com/.well-known/oauth-protected-resource/mcp"`.
   Fetch that URL. Its `authorization_servers[0]` is `https://demo.capitalsnapshot.com/`. Then fetch `https://demo.capitalsnapshot.com/.well-known/oauth-authorization-server` for the endpoints.
2. **Register once.** `POST /register` (JSON):
   ```json
   {
     "client_name": "<your firm>'s research agent",
     "redirect_uris": ["http://127.0.0.1:<port>/callback"],
     "grant_types": ["authorization_code", "refresh_token"],
     "response_types": ["code"],
     "token_endpoint_auth_method": "none"
   }
   ```
   The response is 201 with a `client_id`. Persist `client_id` and reuse it. Founders see `client_name`, so ask the user what it should say.
3. **Authorize with PKCE S256.** Generate `code_verifier` and `state`. Open the user's browser at:
   ```
   https://demo.capitalsnapshot.com/authorize?response_type=code&client_id=<client_id>
     &redirect_uri=<registered uri>&code_challenge=<BASE64URL(SHA256(verifier))>
     &code_challenge_method=S256&state=<state>&resource=https://demo.capitalsnapshot.com/mcp
   ```
   Listen on the loopback redirect URI. The user enters the passcode on the consent page. The server then redirects to your callback with `code` and `state`. Reject the callback if `state` does not match. The consent page expires after 10 minutes.
4. **Exchange the code.** `POST /token`, `application/x-www-form-urlencoded`:
   `grant_type=authorization_code`, `client_id`, `code`, `code_verifier`, `redirect_uri`, `resource=https://demo.capitalsnapshot.com/mcp`.
   The response has `access_token` (1 hour), `token_type` `"bearer"`, `expires_in` 3600 and `refresh_token`. The code is single use and lasts 10 minutes.
5. **Refresh.** `POST /token` with `grant_type=refresh_token`, `client_id` and `refresh_token`. You get a new access token **and a new refresh token**. Save the new refresh token before doing anything else. Run one refresh at a time. Reusing a spent refresh token returns 400 `invalid_grant` and revokes every token from that consent. After `invalid_grant`, go back to step 3.

## 3. MCP session

1. `POST /mcp` with headers `Authorization: Bearer <access_token>`, `Content-Type: application/json` and `Accept: application/json, text/event-stream`. Send the body:
   ```json
   {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"<your client>","version":"0.1.0"}}}
   ```
   The reply is `text/event-stream` with an `mcp-session-id` header. The `data:` line holds the JSON-RPC response, with `serverInfo` `{"name":"capitalsnapshot-investor-agent","version":"0.1.0"}`.
2. Send `{"jsonrpc":"2.0","method":"notifications/initialized"}` with the same headers plus `mcp-session-id`.
3. Send every later request with `mcp-session-id` and `mcp-protocol-version` (the negotiated version).
4. **HTTP 404** with JSON-RPC error `-32001` "Session not found": the session is gone, for example after a redeploy. Initialise again without a session ID and retry the request once.
5. **HTTP 401**: refresh the access token (section 2, step 5) and retry once.

## 4. Tools

| Tool | Required input | Optional input |
|---|---|---|
| `list_startups` | none | `location` (string), `sector` (string), `stage` (`pre-seed` or `seed`), `min_readiness` (number 0 to 100) |
| `get_startup_profile` | `id` | |
| `get_readiness_score` | `id` | |
| `get_cap_table_summary` | `id` | |
| `get_data_room_status` | `id` | |
| `ask_startup` | `id`, `question` (at most 1,000 characters) | |
| `leave_feedback` | `id`, `interest` (`interested`, `maybe` or `pass`), `feedback` (10 to 2,000 characters) | `firm` |

Exact schemas and example responses are in [docs/tools.md](docs/tools.md). Read the schemas from `tools/list` at runtime rather than copying them.

## 5. Behaviour to handle

- Each result is one text block of JSON. Parse `result.content[0].text`.
- `isError: true` means failure. The text is a prose sentence (unknown ID, invalid input, unknown tool). There are no error codes.
- These are **successful** results that you must still check:
  - `ask_startup` puts refusals in `answer`: Q&A not enabled, question too long, limit reached, temporarily unavailable.
  - `leave_feedback` returns `"recorded": false` with a `message` when it refuses. Always check `recorded`.
- Most results carry a `demo_notice` string. Ignore it for logic.
- `location` filters by country name (`Ireland`, `Germany`), case insensitive. City names alone return nothing.
- `sector` must equal a value from `available_sectors`, which always lists every sector whatever the filters.
- `raising` may be `null`.
- IDs look like `cs-N` today but will change. Treat them as opaque strings from `list_startups`.
- Limits: `ask_startup` 30 per hour and 100 per day per client; `leave_feedback` one per company per day, 20 per day per client. Daily limits reset at midnight UTC. The full list is in [docs/sandbox.md](docs/sandbox.md#limits).
- `leave_feedback` writes something a founder reads. Call it only when the user asks for it, or after you have evaluated the company, and write real reasoning.

Response shapes will change in a future versioned release ([docs/changes-ahead.md](docs/changes-ahead.md)). Keep the handling above in one module.

## 6. Verify a working client

Run these in order against the sandbox. Each check states what you should see.

1. `POST /mcp` without a token: 401 with a `WWW-Authenticate` header containing `resource_metadata=`.
2. After consent and token exchange: `access_token` starts with `csa_`, `refresh_token` starts with `csr_`, `expires_in` is 3600.
3. `initialize`: `serverInfo.name` is `capitalsnapshot-investor-agent`, and an `mcp-session-id` header is present.
4. `tools/list`: 7 tools, named `list_startups`, `get_startup_profile`, `get_readiness_score`, `get_cap_table_summary`, `ask_startup`, `get_data_room_status`, `leave_feedback`.
5. `list_startups {}`: `result_count` is 12, and the IDs run from `cs-1` to `cs-12`.
6. `list_startups {"location": "Ireland", "stage": "seed"}`: three rows, `cs-4`, `cs-8`, `cs-9`.
7. `get_startup_profile {"id": "cs-2"}`: `profile.name` is "Croia Health".
8. `ask_startup {"id": "cs-2", "question": "What does the company build?"}`: an answer about remote cardiac monitoring.
9. `ask_startup {"id": "cs-3", "question": "What does the company build?"}`: a successful result whose `answer` says Tur Energy has not enabled agent Q&A.
10. `get_startup_profile {"id": "cs-9999"}`: `isError: true`, with text starting "No startup found".
11. A request with a made-up `mcp-session-id`: HTTP 404, error code `-32001`. Your client re-initialises and succeeds.
12. Refresh: a new `refresh_token`, different from the old one. Store it, then make one more `list_startups` call with the new access token.

Optional, because it writes something a founder reads: `leave_feedback` on `cs-2` with real reasoning should return `"recorded": true`. A second entry the same day should return `"recorded": false`. The nightly reset clears it.

Do not test refresh token reuse. It revokes your tokens, and the user has to consent again.

## 7. Using Claude instead

Add a custom connector in Claude with the URL `https://demo.capitalsnapshot.com/mcp`. Claude registers itself and opens the consent page, where the user enters the passcode. Then ask Claude, for example: "List the seed-stage startups in Ireland and summarise each one's readiness gaps."
