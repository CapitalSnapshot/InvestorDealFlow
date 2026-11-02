# Tools

The sandbox server has seven tools. Six are read only. `leave_feedback` writes a feedback entry for the founder.

Names, titles, descriptions and input schemas below are exactly what `tools/list` returns. Each tool in `tools/list` has the keys `name`, `title`, `description`, `inputSchema` and `execution` (`{"taskSupport": "forbidden"}`). Every tool also declares `annotations`: the five read tools have `readOnlyHint: true`, `idempotentHint: true`, `openWorldHint: false`; `ask_startup` has `readOnlyHint: true`, `idempotentHint: false`, `openWorldHint: false`; `leave_feedback` has `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: false`. There is no `outputSchema`.

Example responses were recorded on the sandbox on 2 October 2026 and verified against it again on 5 October 2026. Where a response had changed (a date, a model-written answer), the example shows the 5 October response. Sandbox data is fictional and returns to the same baseline every night, so you should see the same values, except where noted.

## How results arrive

Every tool result is one text content block holding pretty-printed JSON (or plain prose, for errors). There is no `structuredContent`. On the wire:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "content": [
      { "type": "text", "text": "{\n  \"demo_notice\": \"NOTE: ...\",\n  ..." }
    ]
  }
}
```

Parse `result.content[0].text` as JSON. Under each tool below, "Example response" shows that parsed text. The examples repeat `demo_notice` in full only once, in the first example. After that it appears as `"NOTE: ..."`.

Errors arrive the same way, with `"isError": true` and a prose text. See [Errors and refusals](#errors-and-refusals) at the end.

---

## list_startups

**Title:** List startups in the CapitalSnapshot network

**Description:** Search the CapitalSnapshot startup network. Filter by location (e.g. Dublin, Berlin), sector, stage (pre-seed, seed), and minimum VC readiness score (0 to 100). Returns summary rows with ids for use with the drill-down tools. Call with no filters to see everything.

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "location": {
      "description": "City filter, e.g. Dublin or Berlin",
      "type": "string"
    },
    "sector": {
      "description": "Sector filter, e.g. fintech, healthtech, regtech, climatetech, devtools, robotics, logistics, biotech",
      "type": "string"
    },
    "stage": {
      "description": "Funding stage",
      "type": "string",
      "enum": [
        "pre-seed",
        "seed"
      ]
    },
    "min_readiness": {
      "description": "Minimum overall VC readiness score",
      "type": "number",
      "minimum": 0,
      "maximum": 100
    }
  }
}
```

No input is required. Filters combine with AND.

| Filter | How it matches today |
|---|---|
| `location` | Case insensitive. A company matches when its location contains your text, or your text contains its location. Locations are country names (`Ireland`, `Germany`), so `Ireland` and `ireland` match. A city alone such as `Dublin` matches nothing, despite the description's example. |
| `sector` | Case insensitive, whole value. `fintech` matches, `fin` does not. Use a value from `available_sectors`. |
| `stage` | `pre-seed` or `seed`. Any other value is an input validation error. |
| `min_readiness` | Number from 0 to 100. Companies whose score is at least this value. |

**Example call:** `list_startups {}`

**Example response** (the first row in full; the other eleven rows are in the table below):

```json
{
  "demo_notice": "NOTE: Live data from the CapitalSnapshot platform, founder-maintained and consent-scoped. Document contents are available only where the founder has explicitly shared that document with investor agents.",
  "result_count": 12,
  "available_sectors": [
    "biotech", "circular economy", "climatetech", "devtools", "edtech", "fintech",
    "foodtech", "healthtech", "logistics", "regtech", "robotics"
  ],
  "startups": [
    {
      "id": "cs-1",
      "name": "Kiezpay",
      "tagline": "Payment orchestration for European marketplace sellers.",
      "sector": "fintech",
      "location": "Germany",
      "stage": "seed",
      "readiness_score": 100,
      "raising": {
        "amount_eur": 1800000,
        "instrument": "Equity",
        "status": "in_progress",
        "expected_close": "2027-02-01"
      }
    }
  ]
}
```

All twelve rows from the same call. Each row also has `tagline`, which the table leaves out.

| id | name | sector | location | stage | readiness_score | raising |
|---|---|---|---|---|---|---|
| cs-1 | Kiezpay | fintech | Germany | seed | 100 | 1,800,000 EUR, Equity |
| cs-2 | Croia Health | healthtech | Ireland | pre-seed | 100 | 600,000 EUR, Convertible Note |
| cs-3 | Tur Energy | climatetech | Ireland | pre-seed | 56 | 450,000 EUR, Convertible Note |
| cs-4 | Ferrybank Analytics | regtech | Ireland | seed | 90 | 1,200,000 EUR, Convertible Note |
| cs-5 | Spreewerk Robotics | robotics | Germany | seed | 30 | 2,500,000 EUR, Equity |
| cs-6 | Anchor Line | logistics | Ireland | pre-seed | 45 | 350,000 EUR, Convertible Note |
| cs-7 | Mabel Labs | devtools | Germany | pre-seed | 80 | 750,000 EUR, Convertible Note |
| cs-8 | Gorse Bioworks | biotech | Ireland | seed | 71 | 2,000,000 EUR, Equity |
| cs-9 | Verdipay | fintech | Ireland | seed | 100 | 1,500,000 EUR, Equity |
| cs-10 | Harbour Loop | circular economy | Germany | pre-seed | 25 | 250,000 EUR, Convertible Note |
| cs-11 | Brackwater Foods | foodtech | Ireland | pre-seed | 10 | `null` |
| cs-12 | Lanta Studio | edtech | Germany | pre-seed | 10 | `null` |

Every non-null `raising` had `"status": "in_progress"` and `"expected_close": "2027-02-01"`.

Other calls on the same day:

- `list_startups {"location": "Ireland", "stage": "seed"}` returned 3 rows: `cs-4`, `cs-8`, `cs-9`.
- `list_startups {"location": "Dublin"}` returned `"result_count": 0` and `"startups": []`.
- `available_sectors` was the same eleven values in every call, filtered or not.

---

## get_startup_profile

**Title:** Get a startup's full profile

**Description:** Full snapshot of one startup: team size, traction, current raise terms, readiness score, cap table summary, and data room completeness. Use the id from list_startups.

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Startup id, e.g. cs-1041"
    }
  },
  "required": [
    "id"
  ]
}
```

**Example call:** `get_startup_profile {"id": "cs-2"}`

**Example response:**

```json
{
  "demo_notice": "NOTE: ...",
  "feedback_hint": "If you have evaluated this company, you can leave the founder structured feedback (interest level and reasoning) with the leave_feedback tool. Founders read it in their workspace.",
  "profile": {
    "id": "cs-2",
    "name": "Croia Health",
    "tagline": "Remote cardiac monitoring for post-discharge patients.",
    "about": "Remote cardiac monitoring for post-discharge patients. Croia pairs a wearable patch with a triage dashboard for hospital cardiology teams.",
    "sector": "healthtech",
    "location": "Ireland",
    "stage": "pre-seed",
    "founded": 2024,
    "team_size": 6,
    "raising": {
      "amount_eur": 600000,
      "instrument": "Convertible Note",
      "status": "in_progress",
      "expected_close": "2027-02-01"
    },
    "traction": "Early revenue. Pilot with two Irish hospital groups, 340 monitored patients, HSE procurement conversation open",
    "readiness_score": 100,
    "cap_table_flags": [],
    "data_room_completeness_pct": 100
  }
}
```

`cap_table_flags` is `[]` when there are no flags. For the same company, `get_cap_table_summary` returns `["No structural flags"]`.

---

## get_readiness_score

**Title:** Get VC readiness score breakdown

**Description:** The CapitalSnapshot VC Readiness Score for one startup, broken down by category (team, financials, legal, market, data_room) with the top gaps identified. Scores are 0 to 100, computed from the founder's live workspace. Professional verification (accountant, legal) and score confidence (projected model vs observed data) are reported as separate fields.

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Startup id from list_startups"
    }
  },
  "required": [
    "id"
  ]
}
```

**Example call:** `get_readiness_score {"id": "cs-2"}`

**Example response:**

```json
{
  "demo_notice": "NOTE: ...",
  "startup": "Croia Health",
  "overall_score": 100,
  "scale": "0 to 100; professional verification is reported separately in the verification field, not added to the score",
  "score_label": "Fundable",
  "verification": {
    "accountant_verified": false,
    "legal_verified": false,
    "note": "not independently verified"
  },
  "score_confidence": {
    "level": "medium",
    "label": "Based on entered revenue data, with projections filling gaps",
    "detail": null
  },
  "composition": {
    "base_categories": "103/105 earned, counted as 100/100 (base is capped at 100)",
    "verification_bonus": "0/15"
  },
  "category_breakdown": {
    "Runway": "25/25",
    "Gross margin": "10/10",
    "Retention (NRR)": "3/5",
    "Cap table": "5/5",
    "Team bios": "5/5",
    "Round terms": "5/5",
    "MRR growth": "10/10",
    "Data room": "20/20",
    "VC interview": "20/20"
  },
  "top_gaps": [
    "Retention (NRR) (3/5): 5 pts at 100%+, 3 at 80%+. Lower churn = higher NRR."
  ]
}
```

The category names in `category_breakdown` are the ones the server returns. They differ from the list in the tool description. Each value is an `"earned/max"` string.

A verified company, `get_readiness_score {"id": "cs-9"}` (Verdipay), on the same day:

```json
{
  "score_label": "Verified",
  "verification": {
    "accountant_verified": true,
    "legal_verified": true,
    "note": "accountant verified +10, legal verified +5 (platform-internal score with bonuses: 115)"
  },
  "composition": {
    "base_categories": "103/105 earned, counted as 100/100 (base is capped at 100)",
    "verification_bonus": "15/15 (accountant verified +10, legal verified +5)"
  }
}
```

(Only these fields are shown. Apart from `startup`, every other field matched the cs-2 response, including `overall_score` 100.)

---

## get_cap_table_summary

**Title:** Get cap table summary

**Description:** Cap table summary for one startup: holders with percentages and vesting status, option pool, convertibles outstanding with caps, and any structural flags (dead equity, missing IP assignments, anti-dilution issues, pool sizing).

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Startup id from list_startups"
    }
  },
  "required": [
    "id"
  ]
}
```

**Example call:** `get_cap_table_summary {"id": "cs-2"}`

**Example response:**

```json
{
  "demo_notice": "NOTE: ...",
  "startup": "Croia Health",
  "total_shares": 100000,
  "holders": [
    { "name": "Aoife Kerrigan", "type": "founder", "share_class": "Ordinary", "pct": 42 },
    { "name": "Rian Moloney", "type": "founder", "share_class": "Ordinary", "pct": 42 },
    { "name": "Option pool", "type": "option_holder", "share_class": "ESOP", "pct": 10 },
    { "name": "Angel syndicate", "type": "angel", "share_class": "Ordinary", "pct": 6 }
  ],
  "convertibles_outstanding": [
    {
      "holder": "Regional Enterprise Fund",
      "instrument": "convertible_note",
      "amount_eur": 50000,
      "cap_eur": 3000000,
      "discount_pct": 20,
      "issue_date": "2025-03-01"
    }
  ],
  "flags": ["No structural flags"]
}
```

Holder rows have `name`, `type`, `share_class` and `pct`. The description also mentions vesting status, but no vesting field appears in this response. With no flags, `flags` is `["No structural flags"]`, never empty.

---

## ask_startup

**Title:** Ask a startup's agent a question

**Description:** Ask a free-text question about one startup. The startup's agent answers using data the founder has marked shareable. Returns a reasoned answer, never raw documents.

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Startup id from list_startups"
    },
    "question": {
      "type": "string",
      "description": "Natural language question about the startup, answered by the startup's own AI agent from its live workspace"
    }
  },
  "required": [
    "id",
    "question"
  ]
}
```

In the sandbox, only `cs-2` (Croia Health) has Q&A switched on. The answer is written by a language model from the company's facts and the one document its founder has shared. The wording differs from call to call.

**Example call:** `ask_startup {"id": "cs-2", "question": "Which documents has the founder shared, and what does the company build?"}`

**Example response:**

```json
{
  "demo_notice": "NOTE: ...",
  "startup": "Croia Health",
  "answer": "The founder has shared one document in the data room: **Investor Briefing.pdf**.\n\nThe data room is fully complete (100% completeness as of October 2026) and includes the following sections, all marked complete: TL;DR, company, team, product, traction, financials, market, and legal. The financials are generated live from the company's financial model, and the cap table is rendered live on the platform.\n\n---\n\n**What Croia Health builds:**\n\nCroia Health develops a remote cardiac monitoring solution for post-discharge patients. The product pairs a wearable patch with a triage dashboard designed for hospital cardiology teams. The goal is to allow cardiology staff to monitor patients remotely after they leave hospital, reducing gaps in cardiac care during the post-discharge period.\n\nThe company is based in Ireland, founded in 2024, and currently has a team of 6. It is running pilots with two Irish hospital groups, has 340 patients being monitored, and has an open procurement conversation with the HSE (Ireland's national health service)."
}
```

`answer` is free text and may contain Markdown.

**Q&A switched off**, `ask_startup {"id": "cs-1", "question": "What does the company build?"}`. This is a normal result, not an error:

```json
{
  "demo_notice": "NOTE: ...",
  "startup": "Kiezpay",
  "answer": "Kiezpay has not enabled agent Q&A. The founder controls this setting; the structured tools (profile, readiness, cap table, data room status) remain available."
}
```

Limit hits and temporary failures also come back as normal results with the reason in `answer`. The texts are listed in [sandbox.md](sandbox.md#limits).

---

## get_data_room_status

**Title:** Get data room completeness

**Description:** Data room status for one startup: overall completeness percentage, section-by-section status, and specifically which documents are missing. Returns document status only, never document contents.

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Startup id from list_startups"
    }
  },
  "required": [
    "id"
  ]
}
```

**Example call:** `get_data_room_status {"id": "cs-2"}`

**Example response:**

```json
{
  "demo_notice": "NOTE: ...",
  "startup": "Croia Health",
  "completeness_pct": 100,
  "last_updated": "2026-10-05",
  "sections": {
    "tldr": { "complete": true, "missing": [] },
    "company": {
      "complete": true,
      "missing": [],
      "note": "Cap table entered on the platform and rendered live for investors (ownership summary, exit waterfall); see get_cap_table_summary"
    },
    "team": { "complete": true, "missing": [] },
    "product": { "complete": true, "missing": [] },
    "traction": { "complete": true, "missing": [] },
    "financials": {
      "complete": true,
      "missing": [],
      "note": "Generated live from the company's financial model (investor pack, P&L, cash flow, balance sheet)"
    },
    "market": { "complete": true, "missing": [] },
    "legal": { "complete": true, "missing": [] }
  }
}
```

Each section has `complete` and `missing`, and sometimes `note`. In the sandbox, `last_updated` is the date of the most recent nightly reset, so you will see a later date than the one above.

---

## leave_feedback

**Title:** Leave feedback for a startup's founder

**Description:** After evaluating a startup, leave the founder short structured feedback: your interest level (interested, maybe, pass) and the reasoning behind it. The founder reads it in their CapitalSnapshot workspace, attributed to your agent. Optionally name the firm you evaluate for. One feedback entry per startup per day.

**Input schema:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Startup id from list_startups"
    },
    "interest": {
      "type": "string",
      "enum": [
        "interested",
        "maybe",
        "pass"
      ],
      "description": "Your overall verdict: interested (would take a meeting), maybe (watching, not now), pass"
    },
    "feedback": {
      "type": "string",
      "description": "Constructive reasoning for the founder: what stood out, what is missing, what would change your verdict"
    },
    "firm": {
      "description": "Name of the fund or firm you evaluate for, if you can share it",
      "type": "string"
    }
  },
  "required": [
    "id",
    "interest",
    "feedback"
  ]
}
```

The founder sees the feedback attributed to the `client_name` your client registered, plus `firm` if you send it.

**Example call:**

```json
{
  "id": "cs-2",
  "interest": "maybe",
  "feedback": "Clear product story and a credible clinical pathway; we would want to see observed revenue before taking a meeting.",
  "firm": "Example Capital"
}
```

**Example response:**

```json
{
  "startup": "Croia Health",
  "recorded": true,
  "message": "Thank you. Your feedback has been delivered to Croia Health's founder, attributed to Example Capital research agent (Example Capital)."
}
```

`leave_feedback` results have no `demo_notice`. The message names your registered `client_name`, then `firm` in brackets when you send one.

**Refusals.** These are normal results with `"recorded": false`, not errors:

A second entry for the same company on the same day:

```json
{
  "startup": "Croia Health",
  "recorded": false,
  "message": "You have already left feedback for Croia Health today. The founder has it."
}
```

Feedback under 10 characters, `leave_feedback {"id": "cs-3", "interest": "pass", "feedback": "short"}`:

```json
{
  "startup": "Tur Energy",
  "recorded": false,
  "message": "Feedback too short to be useful to the founder. Please include your reasoning."
}
```

Other refusal messages are listed in [sandbox.md](sandbox.md#limits).

---

## Errors and refusals

Some failures are `isError: true` results with a prose message. Others are ordinary results that carry the refusal in their fields. None of them is a JSON-RPC `error` object, except the session error. The HTTP status of every tool result is 200.

### Unknown company

`get_startup_profile {"id": "cs-9999"}`. Every tool that takes an `id` gives the same response for an unknown ID, including a value that is not in `cs-N` form, such as `not-an-id`:

```json
{
  "result": {
    "content": [
      { "type": "text", "text": "No startup found with id \"cs-9999\". Use list_startups to see available ids." }
    ],
    "isError": true
  },
  "jsonrpc": "2.0",
  "id": 17
}
```

### Invalid input

`get_startup_profile {}` (missing `id`):

```json
{
  "result": {
    "content": [
      { "type": "text", "text": "MCP error -32602: Input validation error: Invalid arguments for tool get_startup_profile: [\n  {\n    \"expected\": \"string\",\n    \"code\": \"invalid_type\",\n    \"path\": [\n      \"id\"\n    ],\n    \"message\": \"Invalid input: expected string, received undefined\"\n  }\n]" }
    ],
    "isError": true
  },
  "jsonrpc": "2.0",
  "id": 19
}
```

`list_startups {"stage": "series-a"}`:

```json
{
  "result": {
    "content": [
      { "type": "text", "text": "MCP error -32602: Input validation error: Invalid arguments for tool list_startups: [\n  {\n    \"code\": \"invalid_value\",\n    \"values\": [\n      \"pre-seed\",\n      \"seed\"\n    ],\n    \"path\": [\n      \"stage\"\n    ],\n    \"message\": \"Invalid option: expected one of \\\"pre-seed\\\"|\\\"seed\\\"\"\n  }\n]" }
    ],
    "isError": true
  },
  "jsonrpc": "2.0",
  "id": 20
}
```

### Unknown tool

```json
{
  "result": {
    "content": [ { "type": "text", "text": "MCP error -32602: Tool no_such_tool not found" } ],
    "isError": true
  },
  "jsonrpc": "2.0",
  "id": 24
}
```

### Refusals returned as normal results

| Tool | Situation | Where the reason is |
|---|---|---|
| `ask_startup` | Q&A switched off, question too long, a limit reached, Q&A temporarily unavailable | `answer` |
| `leave_feedback` | Too short, too long, already sent today, daily limit reached, could not be recorded | `message`, with `"recorded": false` |

### Session and auth errors (HTTP level)

| Situation | HTTP | Body |
|---|---|---|
| Unknown or expired session ID | 404 | `{"jsonrpc":"2.0","error":{"code":-32001,"message":"Session not found"},"id":null}` |
| Missing access token | 401 | `{"error":"Unauthorized"}` with `WWW-Authenticate: Bearer resource_metadata="https://demo.capitalsnapshot.com/.well-known/oauth-protected-resource/mcp"` |
| Invalid or expired access token | 401 | Same body, with `WWW-Authenticate: Bearer error="invalid_token", error_description="The access token is invalid or expired", resource_metadata="https://demo.capitalsnapshot.com/.well-known/oauth-protected-resource/mcp"` |

After a 404, send a new `initialize` without a session ID. After a 401, refresh the access token (see [auth.md](auth.md)).
