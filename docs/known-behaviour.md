# Known behaviour

What an agent must handle on the sandbox today. Some of it will change in a future versioned release (see [changes-ahead.md](changes-ahead.md)). Keep the handling for these cases in one place in your code.

## Results

1. **Results are JSON inside text.** Each result is one `text` content block. Parse `content[0].text` as JSON. There is no `structuredContent` and no `outputSchema`.
2. **`demo_notice` on most results.** Every result from `list_startups`, `get_startup_profile`, `get_readiness_score`, `get_cap_table_summary`, `get_data_room_status` and `ask_startup` has a `demo_notice` string. `leave_feedback` results and error results do not. Show it to a person if you like, but never parse it. On the sandbox it reads "NOTE: Live data from the CapitalSnapshot platform, ...". Even so, every sandbox company is fictional.
3. **Shapes differ by tool.** Most tools name the company in `startup` (a string). `get_startup_profile` nests everything under `profile`, and `list_startups` returns `startups` rows. There is no common envelope.

## Errors and refusals

4. **Errors are prose with `isError`.** An unknown ID, invalid input and an unknown tool all return a tool result with `"isError": true` and a sentence in the text, never a JSON-RPC error. Input validation errors embed the raw validator output after `MCP error -32602: Input validation error:`. There are no error codes to switch on. Treat any `isError` result as a failure, and show the text to a person.
5. **Limit hits, Q&A off and feedback refusals are successful results.** They have no `isError`:
   - `ask_startup` puts the reason in `answer`: Q&A not enabled, question too long, hourly, daily or global limit reached, temporarily unavailable. Your agent cannot tell these from a real answer by shape alone. If you need to, match the fixed sentences listed in [sandbox.md](sandbox.md#limits).
   - `leave_feedback` returns `"recorded": false` with a `message`. Check `recorded`, not the absence of `isError`.
6. **Unknown IDs of any form** (`cs-9999`, `not-an-id`) give the same "No startup found" error.

## list_startups

7. **`location` matches countries, not cities.** It is a case-insensitive containment test against the country name (`Ireland`, `Germany`). It matches when the country name contains your text or your text contains the country name. So `Ireland`, `ireland` and `Dublin, Ireland` match Irish companies. `Dublin` or `Berlin` alone return nothing, even though the tool description gives them as examples.
8. **`sector` is a whole-value match**, case insensitive. Take values from `available_sectors`.
9. **`available_sectors` ignores the filters.** It always lists every sector in the network, whatever you filtered on. It also includes sectors that the `sector` description does not mention (`circular economy`, `edtech`, `foodtech`).
10. **`raising` can be `null`**, for a company with no current round (`cs-11`, `cs-12`).

## IDs

11. **IDs are `cs-N` today** (`cs-1` to `cs-12` in the sandbox). Treat them as opaque strings. Take them from `list_startups`, and do not build or parse them. The `get_startup_profile` input description shows `cs-1041` as an example, but no such company is in the sandbox.

## Small inconsistencies

12. With no flags, `get_startup_profile` returns `"cap_table_flags": []`, but `get_cap_table_summary` returns `"flags": ["No structural flags"]` for the same company.
13. `get_readiness_score` category names (`Runway`, `Gross margin`, `Retention (NRR)`, ...) differ from the categories listed in its description. Each value is an `"earned/max"` string, not a number.
14. `get_cap_table_summary` has no vesting field, although its description mentions vesting status.
15. `ask_startup` answers are model-written prose and may contain Markdown. They vary from call to call.

## Protocol and session

16. **Protocol versions.** `initialize` with `2025-06-18` or `2025-11-25` gets the same version back. `tools/list` is the same under both.
17. **`execution.taskSupport` is `"forbidden"`** on every tool. Call tools synchronously.
18. **Sessions do not survive a redeploy.** A request with an old `mcp-session-id` gets HTTP 404 with JSON-RPC error `-32001` "Session not found". Initialise again and retry.
19. **Refresh tokens rotate.** Reusing a spent refresh token revokes the whole chain. See [auth.md](auth.md#7-refresh-and-rotation).
