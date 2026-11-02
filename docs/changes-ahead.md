# Changes ahead

This repository documents the sandbox server as it is deployed today. A future versioned release will change the response shapes:

- **A consistent envelope.** Every tool result will have the same top-level structure.
- **Structured error codes.** Refusals and errors will carry machine-readable codes instead of prose.
- **Structured content.** Results will carry `structuredContent` alongside the text block.
- **Opaque IDs.** Company IDs will no longer have the `cs-N` form.

What to do now:

- Do not hard-code today's quirks (see [known-behaviour.md](known-behaviour.md)). Keep the code that handles them in one place, so it is easy to replace.
- Do not parse prose messages beyond what you need today.
- Treat IDs as opaque strings. Always take them from `list_startups`.

Changes will be announced in this repository, with notice before they take effect. Watch the repository for releases to be told.
