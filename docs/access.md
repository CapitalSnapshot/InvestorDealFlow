# The Access stage

Access is the first stage of the investor dealflow. An investor's agent connects to CapitalSnapshot, finds startups, looks at the facts founders keep in their workspace, and tells founders what it thinks. This page describes the stage as it is deployed on the sandbox today.

## What your agent can do

| Step | Tool | What you get |
|---|---|---|
| Discover | `list_startups` | Summary rows for every visible startup, with optional filters for location, sector, stage and minimum readiness score |
| Profile | `get_startup_profile` | One startup's full snapshot: about, team size, traction, current raise, readiness score, cap table flags, data room completeness |
| Readiness | `get_readiness_score` | The VC readiness score, its breakdown by category, top gaps, professional verification and score confidence |
| Cap table | `get_cap_table_summary` | Holders with percentages, convertibles outstanding with caps and discounts, structural flags |
| Data room | `get_data_room_status` | Completeness percentage and status for each section. Never document contents. |
| Ask | `ask_startup` | A written answer to a free-text question, drawn from documents the founder has marked shareable. Only where the founder has switched Q&A on. |
| Respond | `leave_feedback` | Records your interest level (`interested`, `maybe`, `pass`) and reasoning for the founder |

Full schemas, examples and error shapes are in [tools.md](tools.md).

A typical run:

1. `list_startups` with filters that fit your thesis.
2. For each candidate, `get_startup_profile`, then `get_readiness_score`, `get_cap_table_summary` and `get_data_room_status` as needed.
3. `ask_startup` for questions the structured data does not answer, where Q&A is on.
4. `leave_feedback` once you have a view.

## What the founder sees

- Founders choose whether their company is visible to investor agents, and whether `ask_startup` is switched on.
- `ask_startup` uses only documents the founder has marked shareable. Raw documents are never returned.
- Calls about a specific company are recorded against that company with your client's registered name. The founder can see which agents looked.
- Feedback appears in the founder's CapitalSnapshot workspace, attributed to your client's registered name and, if you send it, your firm.

Choose a `client_name` at registration that tells a founder who you are. See [auth.md](auth.md#2-dynamic-client-registration-rfc-7591).

## What it is not

- Six of the seven tools only read. `leave_feedback` is the only write.
- No documents, files or downloads. Data room tools report status only.
- No ranking. `list_startups` returns rows in ID order. Your agent applies your own criteria.

## Connection summary

| Item | Value |
|---|---|
| Endpoint | `https://demo.capitalsnapshot.com/mcp` |
| Protocol | Model Context Protocol, Streamable HTTP transport |
| Protocol versions | `2025-06-18` and `2025-11-25` (the server returns the version you ask for) |
| Server | `capitalsnapshot-investor-agent` version `0.1.0`, capability `tools` with `listChanged: true` |
| Auth | OAuth 2.1, dynamic client registration, PKCE S256, passcode at consent ([auth.md](auth.md)) |
| Data | 12 fictional companies, reset every night ([sandbox.md](sandbox.md)) |
| Status | Documented against the sandbox as of 2 November 2026 |

## Later stages

Selection, queries, term sheet negotiation and the deal room will be added to this repository later.
