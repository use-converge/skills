# Converge MCP

Converge provides a hosted, HTTP-only MCP server at
`https://app.use-converge.com/mcp`. It uses Streamable HTTP with stateless JSON
responses. It is not a local stdio server.

## Discovery

The Server Card is available at
`https://app.use-converge.com/mcp/server-card` with media type
`application/mcp-server-card+json`. It advertises the real remote endpoint,
supported protocol versions, and the required Bearer credential. The card does
not enumerate tools; use the MCP handshake and `tools/list` for the live tool
set.

## Authentication

Send a Converge API key with the `api.full` scope as a static Bearer token:

```http
Authorization: Bearer cvg_...
```

The server does not accept session cookies or query-string tokens and does not
publish OAuth Protected Resource Metadata or OIDC discovery. Missing or
invalid credentials return `401` with a `WWW-Authenticate: Bearer` challenge;
a valid key without `api.full` returns `403`.

## Tools

The current server exposes these authenticated tools:

- `research_pulse_models` lists model selectors available to the account.
- `research_pulse_list` lists the account's runs with bounded pagination.
- `research_pulse_get` returns one account-owned run and its metadata.
- `research_pulse_markdown` returns the final Markdown artifact when ready.
- `research_pulse_create` starts an asynchronous, cost-bearing run.
- `research_pulse_request_summary` starts the existing summary workflow.

Tool schemas and structured results from `tools/list` are authoritative. Every
read is scoped to the account owning the Bearer key.

## Creating and retrying runs

`research_pulse_create` can spend provider or prepaid-credit budget. Always
send a stable client-generated `idempotency_key`; reuse the same key only for
the same request. Replaying an identical request returns the original run ID,
while changing the request returns an `idempotency_mismatch` error.

The server negotiates the MCP protocol version during `initialize`. The SDK
currently supports `2026-07-28`, `2025-11-25`, `2025-06-18`, `2025-03-26`, and
`2024-11-05`; use the version returned by the server for the session.
