# InvestorDealFlow

A guide for an investor's tech team to connect an AI agent to CapitalSnapshot's investor sandbox and use the **Access** stage: find startups, read their profile, readiness score, cap table and data room status, ask questions, and leave feedback for founders.

It documents the sandbox server exactly as it is deployed today. Response shapes will change in a future versioned release. See [docs/changes-ahead.md](docs/changes-ahead.md).

## What you need

- The sandbox endpoint: `https://demo.capitalsnapshot.com/mcp`. Use only the sandbox endpoint in this repository.
- The sandbox passcode. It comes privately from your CapitalSnapshot contact. A person enters it once, on the consent page in a browser. Never put it in code or share it in an issue.
- Claude, or a coding agent and an MCP client library.

The sandbox holds 12 fictional companies and resets every night. See [docs/sandbox.md](docs/sandbox.md).

**Production moved to `https://mcp.capitalsnapshot.com/mcp` on 30 October 2026. The sandbox endpoint is unchanged.** This repository still covers only the sandbox. Production needs a production passcode and is not documented here.

## Quick start

### With Claude

1. Add a custom connector with the URL `https://demo.capitalsnapshot.com/mcp`.
2. When the consent page opens, enter the passcode and approve.
3. Ask, for example: "List the seed-stage startups in Ireland and summarise each one's readiness gaps."

### With a coding agent

Point your coding agent at this repository and tell it:

> Read AGENTS.md and build a client for the CapitalSnapshot sandbox in <your language>. Run the checks in section 6. I will type the passcode into the consent page myself.

[AGENTS.md](AGENTS.md) is self-contained: endpoint, transport, OAuth flow, tools, the behaviour to handle, and a checklist to verify a working client.

Send a User-Agent header that names your client, for example my-agent/1.0. Requests with some generic library user agents are currently refused.

## Documentation

| File | Contents |
|---|---|
| [AGENTS.md](AGENTS.md) | End-to-end instructions for a coding agent |
| [docs/access.md](docs/access.md) | The Access stage: what an agent can do and what founders see |
| [docs/auth.md](docs/auth.md) | Discovery, dynamic client registration, PKCE, consent, tokens, refresh rotation |
| [docs/tools.md](docs/tools.md) | All seven tools: schemas, example responses, errors and refusals |
| [docs/sandbox.md](docs/sandbox.md) | The fictional companies, nightly reset, redeploys, limits |
| [docs/known-behaviour.md](docs/known-behaviour.md) | What an agent must handle today |
| [docs/changes-ahead.md](docs/changes-ahead.md) | What will change, and how you will hear about it |

## Roadmap

Access is the first stage. Selection, queries, term sheet negotiation and the deal room will be added to this repository later.

## Feedback

Open an issue in this repository. There are templates for feedback, bugs and questions. You can also tell your CapitalSnapshot contact. Never paste passcodes, tokens or client IDs into an issue.

## Licence

Code in this repository is licensed under the [Apache License 2.0](LICENSE). Documentation (this README, AGENTS.md and everything in `docs/`) is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
