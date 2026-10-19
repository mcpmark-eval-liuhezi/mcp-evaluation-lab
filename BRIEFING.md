# MCP Evaluation Lab — Briefing

Prepared for the Thursday team sync: go/no-go on adopting the Model Context Protocol (MCP) for our internal tooling. All facts below were verified against current, primary sources (listed under **Sources**) at the time of writing.

## What is MCP

The Model Context Protocol (MCP) is an open-source standard for connecting AI applications to external systems. Using MCP, AI applications (hosts such as Claude, ChatGPT, or our own internal assistants) can connect to data sources (files, databases), tools (search engines, calculators) and workflows (templated prompts) — so the model can access key information and perform tasks.

- The common analogy: **MCP is like a USB-C port for AI applications** — a standardized plug that replaces one-off custom integrations.
- It was **introduced and open-sourced by Anthropic on November 25, 2024**, alongside the specification, SDKs, local MCP server support in the Claude Desktop apps, and an open-source repository of reference servers.
- **Architecture.** MCP uses JSON-RPC 2.0 messages between three participants: **Hosts** (LLM applications that initiate connections), **Clients** (connectors inside the host), and **Servers** (services that provide context and capabilities). MCP takes inspiration from the Language Server Protocol (LSP), which standardized language support across IDEs; MCP does the same for context and tools across AI applications.
- **Core server primitives:** Resources (context/data), Prompts (templated messages/workflows), Tools (functions the model can execute). **Client-side primitive:** Sampling (server-initiated, consent-gated LLM interactions).
- **Ecosystem.** It is no longer a single-vendor protocol — clients such as Claude, ChatGPT, Visual Studio Code, Cursor and others support it, and the specification is maintained in the open in the `modelcontextprotocol` GitHub organization, with a formal governance structure, working groups and Spec Enhancement Proposals (SEPs).

## Transports

A "transport" is the binding that frames and delivers MCP's JSON-RPC messages; protocol semantics are identical on every transport. The **current spec defines exactly two standard built-in transports**:

1. **stdio** — the client launches the MCP server as a subprocess; the server reads newline-delimited JSON-RPC from `stdin` and writes to `stdout`. `stderr` may be used for logging. Clients SHOULD support stdio whenever possible.
2. **Streamable HTTP** — the server is an independent process handling multiple clients over a **single HTTP endpoint** (the "MCP endpoint", e.g. `https://example.com/mcp`). Every JSON-RPC message is its own HTTP POST with an `Accept: application/json, text/event-stream` header; the server answers with either a single JSON object or a Server-Sent Events (SSE) stream scoped to that request.

**Confirming your recollection: yes — the old HTTP+SSE transport is deprecated.** The two-endpoint HTTP+SSE transport from protocol version `2024-11-05` (a long-lived SSE channel plus a separate POST endpoint) was **replaced by Streamable HTTP in protocol version `2025-03-26`**. It remains formally classified as **Deprecated** under the spec's feature lifecycle policy (SEP-2596): new implementations SHOULD NOT adopt it, existing implementations SHOULD migrate to Streamable HTTP, and it is eligible for removal in a future revision. SSE is still used *inside* Streamable HTTP as an optional response/streaming format — what was removed is the old dedicated-SSE-channel transport. The spec also publishes a backwards-compatibility recipe for servers that must keep hosting the old SSE+POST endpoints for older clients.

**Revision-by-revision transport status:**

| Revision | HTTP transport |
| --- | --- |
| `2024-11-05` | HTTP+SSE (original) |
| `2025-03-26` | **Streamable HTTP** (replaces HTTP+SSE; HTTP+SSE deprecated) |
| `2025-11-25` | Streamable HTTP (unchanged; clarifications — e.g. servers MUST return HTTP 403 for invalid `Origin` headers) |
| `2026-07-28` | Streamable HTTP (breaking: removes the GET stream endpoint and protocol-level sessions/`Mcp-Session-Id`; SSE resumability removed) |

**Streamable HTTP security requirements** (relevant to our internal deployment): servers MUST validate the `Origin` header on incoming connections and return HTTP 403 Forbidden when invalid (DNS-rebinding defense); locally-run servers SHOULD bind only to localhost rather than `0.0.0.0`; and servers SHOULD implement authentication on all connections.

## SDKs

Official SDKs live in the `modelcontextprotocol` organization on GitHub and are **classified into tiers** based on feature completeness, protocol support and maintenance commitment. All SDKs cover building servers (tools/resources/prompts) and clients, plus local and remote transports.

| SDK | Repository | Tier |
| --- | --- | --- |
| TypeScript | modelcontextprotocol/typescript-sdk | Tier 1 |
| Python | modelcontextprotocol/python-sdk | Tier 2 |
| C# | modelcontextprotocol/csharp-sdk | Tier 1 |
| Go | modelcontextprotocol/go-sdk | Tier 1 |
| Rust | modelcontextprotocol/rust-sdk | Tier 1 |
| Java | modelcontextprotocol/java-sdk | Tier 2 |
| Ruby | modelcontextprotocol/ruby-sdk | Tier 2 |
| Swift | modelcontextprotocol/swift-sdk | Tier 3 |
| PHP | modelcontextprotocol/php-sdk | Tier 3 |
| Kotlin | modelcontextprotocol/kotlin-sdk | Tier 3 |

> **Caveat:** tier assignments differ slightly between documentation revisions. The Tier 1 set is consistently TypeScript, C#, Go and Rust; **Python, Ruby and Java are the ones that move between Tier 1/Tier 2 across revisions** — verify the tier of your preferred language against the live SDKs page before committing to a stack. Swift, PHP and Kotlin are consistently Tier 3.

All ten languages above are officially maintained; several notable adjacent projects are also in the org: `modelcontextprotocol/servers` (reference server implementations), `modelcontextprotocol/inspector` (visual testing tool for MCP servers), `modelcontextprotocol/registry` (community/enterprise registry service) and `modelcontextprotocol/modelcontextprotocol` (spec + docs + SEPs).

## Sources

URLs actually read while preparing this briefing:

- https://www.anthropic.com/news/model-context-protocol — Anthropic's original announcement (Nov 25, 2024): MCP introduced and open-sourced by Anthropic.
- https://modelcontextprotocol.io/docs/getting-started/intro — "What is the Model Context Protocol?": definition, USB-C analogy, ecosystem support.
- https://modelcontextprotocol.io/specification/2024-11-05 — original 2024-11-05 specification (JSON-RPC, hosts/clients/servers, LSP inspiration).
- https://modelcontextprotocol.io/specification/2025-03-26/basic/transports — Streamable HTTP replaces the HTTP+SSE transport; two standard transports.
- https://modelcontextprotocol.io/specification/2025-11-25/basic/transports — stdio + Streamable HTTP requirements; Origin/403, localhost binding.
- https://modelcontextprotocol.io/specification/2025-11-25/changelog — key changes since 2025-06-18 (OIDC discovery, SDK tiering SEP-1730, RFC 9728 alignment, governance SEPs).
- https://modelcontextprotocol.io/specification/2026-07-28/basic/transports — current transport overview: transport-agnostic binding model, stdio + Streamable HTTP.
- https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http — HTTP+SSE classified Deprecated under the feature lifecycle policy (SEP-2596); GET endpoint and sessions removed.
- https://modelcontextprotocol.io/specification/2026-07-28/changelog — key changes since 2025-11-25 (stateless protocol, `server/discover`, `subscriptions/listen`, MRTR).
- https://modelcontextprotocol.io/docs/sdk.md — official SDK list and tiers.
- https://modelcontextprotocol.io/docs/2026-07-28/sdk — SDK tier table (revision-specific tiering).
- https://modelcontextprotocol.io/community/contributing — repository layout of the `modelcontextprotocol` org.
- https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/ — "One Year of MCP" (Nov 2025 release notes; authorization and registry direction).
- https://cloud.google.com/discover/what-is-model-context-protocol — Google Cloud explainer corroborating that MCP was introduced by Anthropic in November 2024.
- https://github.com/modelcontextprotocol — the MCP GitHub organization (SDK and tooling repositories).
