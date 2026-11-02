# The sandbox

Endpoint: `https://demo.capitalsnapshot.com/mcp`

Use only the sandbox endpoint in this repository.

## Companies

The sandbox holds 12 fictional companies, `cs-1` to `cs-12`. The companies, founders, people, investors and figures are all made up. Do not enter real information anywhere in the sandbox.

| id | name | sector | location | stage | Q&A |
|---|---|---|---|---|---|
| cs-1 | Kiezpay | fintech | Germany | seed | off |
| cs-2 | Croia Health | healthtech | Ireland | pre-seed | **on** |
| cs-3 | Tur Energy | climatetech | Ireland | pre-seed | off |
| cs-4 | Ferrybank Analytics | regtech | Ireland | seed | off |
| cs-5 | Spreewerk Robotics | robotics | Germany | seed | off |
| cs-6 | Anchor Line | logistics | Ireland | pre-seed | off |
| cs-7 | Mabel Labs | devtools | Germany | pre-seed | off |
| cs-8 | Gorse Bioworks | biotech | Ireland | seed | off |
| cs-9 | Verdipay | fintech | Ireland | seed | off |
| cs-10 | Harbour Loop | circular economy | Germany | pre-seed | off |
| cs-11 | Brackwater Foods | foodtech | Ireland | pre-seed | off |
| cs-12 | Lanta Studio | edtech | Germany | pre-seed | off |

[tools.md](tools.md#list_startups) has readiness scores and round terms as recorded on 2 October 2026.

Useful test cases:

- `cs-2` has Q&A switched on. `ask_startup` answers from one shared document, "Investor briefing.pdf". Every other company returns the "has not enabled agent Q&A" answer.
- `cs-9` is professionally verified (`score_label` "Verified").
- `cs-11` and `cs-12` have no current round (`raising` is `null`).
- `cs-5` (30), `cs-10` (25), `cs-11` and `cs-12` (10 each) had the lowest readiness scores on 2 October.

## Nightly reset

Every night at 03:30 UTC, the sandbox returns to its baseline:

- Company data goes back to the same values, under the same IDs.
- Visit records, Q&A history and feedback entries are wiped. Feedback you left yesterday is gone, and you can leave feedback again.
- OAuth registrations and tokens **survive** the reset. You do not need to register or consent again.

Because of the reset, dates such as `last_updated` in `get_data_room_status` show the most recent reset date.

## Redeploys

When the sandbox server is redeployed, open MCP sessions are dropped. The next request with an old session ID gets:

```
HTTP 404
{"jsonrpc":"2.0","error":{"code":-32001,"message":"Session not found"},"id":null}
```

Your client should then send a new `initialize` without a session ID, and repeat the request.

## Limits

These limits come from the server source. "Per client" means per registered OAuth client. Daily limits reset at midnight UTC. The hourly limit is a rolling 60 minutes.

### ask_startup

| Limit | Value | Answer text when hit |
|---|---|---|
| Question length | 1,000 characters | `That question is too long (<n> characters, limit 1000). Please ask a shorter, more focused question.` |
| Per client, per hour | 30 questions | `You have reached the limit of 30 agent questions per hour. Please try again a little later; the structured tools remain available.` |
| Per client, per day | 100 questions | `You have reached your daily limit of 100 agent questions. Please try again tomorrow; the structured tools remain available.` |
| All clients together, per day | 500 questions | `The agent Q&A service has reached its daily capacity across all users. Please try again tomorrow; the structured tools remain available.` |

If the answering service fails or times out, the answer is `<Company>'s agent is temporarily unavailable. Please try again shortly.` A failed question does not count against your limits.

The agent reads at most 30,000 characters of the company's shared documents, newest first.

All of these come back as normal results, with the text in `answer`. None is an error.

### leave_feedback

| Limit | Value | Message when hit |
|---|---|---|
| Minimum length | 10 characters, after trimming spaces | `Feedback too short to be useful to the founder. Please include your reasoning.` |
| Maximum length | 2,000 characters, after trimming spaces | `Feedback too long (<n> characters, limit 2000). Please summarise.` |
| Per company, per client, per day | 1 entry | `You have already left feedback for <Company> today. The founder has it.` |
| Per client, per day | 20 entries | `You have reached the daily limit of 20 feedback entries. Please continue tomorrow.` |
| `firm` | Stored up to 160 characters | |

If the entry cannot be stored, the message is `Feedback could not be recorded right now. Please try again shortly.`

All of these come back as normal results with `"recorded": false`. None is an error.

### Authorization

| Item | Value |
|---|---|
| Authorization code | 10 minutes, single use |
| Access token | 1 hour |
| Refresh token | 30 days from issue; each refresh issues a new one |
| Consent page | 10 minutes |
| `POST /register` | 20 requests per hour |
| `/authorize` | 100 requests per 15 minutes |
| `POST /token` | 50 requests per 15 minutes |

Wrong passcodes lock consent: 5 per client, 10 per IP address (IPv6: per /64), 1000 server-wide, each per sliding 15 minutes. A locked page returns HTTP 429 with `Retry-After`. See [auth.md](auth.md).

### Other tools

`list_startups`, `get_startup_profile`, `get_readiness_score`, `get_cap_table_summary` and `get_data_room_status` have no published per-client limits. Please call them at a reasonable pace.
