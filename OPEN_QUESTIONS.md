# Open Questions — MCP Adoption

Decisions the team still needs to make before we can go/no-go on adopting the Model Context Protocol for our internal tooling. Each item below has a matching tracking issue filed under the parent "MCP adoption decision" issue.

## 1. Which protocol revision do we target — and how do we handle the 2026-07-28 breaking changes?

The spec has moved quickly: `2024-11-05` → `2025-03-26` (Streamable HTTP replaces HTTP+SSE) → `2025-11-25` → `2026-07-28`. The current `2026-07-28` revision makes the protocol **stateless**: it removes the `initialize`/`notifications/initialized` handshake, removes protocol-level sessions and the `Mcp-Session-Id` header from Streamable HTTP, replaces the GET stream endpoint with a new `subscriptions/listen` method, and removes SSE stream resumability (`Last-Event-ID`). Anything we write today that assumes sessions or the handshake will need rework.

- Do we build against the current `2026-07-28` revision, the widely-deployed `2025-11-25` revision, or both (with a compatibility matrix)?
- Do any tools in our existing stack still speak the deprecated HTTP+SSE transport, and if so, do we keep a compatibility path for them?

## 2. Which transport(s) do we standardize on — stdio vs. Streamable HTTP?

Both standard transports remain supported. stdio is the simplest for local, subprocess-style tooling (client launches the server); Streamable HTTP is required for centralized, multi-client internal services and for anything we'd want behind our gateway. A single repo with only stdio is fine for a quick demo, but our internal platform services would be Streamable HTTP.

- Do we pick one, or allow both with a clear rule for when each applies?
- If we adopt Streamable HTTP, are we prepared to implement the spec's security requirements — `Origin` header validation returning 403 on invalid origins, localhost-only binding for local servers, and authentication on all connections?

## 3. Which SDK / language stack do we build our internal servers with?

Ten official SDKs exist (TypeScript, Python, C#, Go, Rust, Java, Ruby, Swift, PHP, Kotlin), tiered by feature completeness and maintenance commitment. TypeScript, C#, Go and Rust are Tier 1; Python, Ruby and Java are Tier 1 or Tier 2 depending on the documentation revision; Swift, PHP and Kotlin are Tier 3.

- Which languages do we standardize on, and does the tier of our chosen language clear our bar for production support and long-term maintenance?
- Do we need to verify tier status and current protocol-revision support in the SDK we pick before committing?

## 4. How do we handle authentication and authorization?

The authorization spec is now deeply tied to OAuth 2.1 / RFC 9728 Protected Resource Metadata, with recent additions including OAuth Client ID Metadata Documents for URL-based client registration (SEP-991), incremental scope consent via `WWW-Authenticate` (SEP-835), OIDC Discovery support, plus authorization extensions for client credentials (SEP-1046) and enterprise IdP policy controls / Cross App Access (SEP-990).

- Which flow fits our internal IdP — do we rely on our existing enterprise IdP integration, or stand up MCP-specific auth?
- Do we need machine-to-machine (client credentials) authorization for service-to-service tools?

## 5. Should we run an internal MCP registry?

The project ships `modelcontextprotocol/registry`, a community-driven registry service, and has published an enterprise-registry vision with self-managed governance controls.

- Do we run our own registry so teams can discover approved internal MCP servers, or skip it for now?

## 6. What is our security and data-governance model for tool access?

MCP gives models arbitrary data-access and code-execution paths. The spec's core security principles require explicit user consent and control over data access and tool invocation, host-mediated data privacy, and user-approved sampling.

- Who owns the approval workflow for a new internal MCP server/tool?
- How do we audit tool invocations, and what consent UX (if any) do we need for internal users?
- What is our policy on tool output flowing into external model providers?

## 7. Scope of the first pilot

- Which tool/service do we pilot first, and what does "success" look like (usage, latency, reliability)?
- Do we need an MCP Inspector-based testing/QA practice (`modelcontextprotocol/inspector`) as part of our tooling?
