# MCP Knowledgebase

> A practitioner's guide to the Model Context Protocol - what it is, how it works, how to build it well, and how to secure it.
>
> *Last updated: September 2026. Current protocol revision: **`2026-07-28`**.*

---

> ### ⚠️ If you last read this before August 2026, start here
>
> The `2026-07-28` revision is the largest breaking change since MCP launched. **MCP is now a stateless request/response protocol.** The `initialize` handshake, the `Mcp-Session-Id` header, server-initiated requests, and SSE stream resumability are all gone. Roots, Sampling, and Logging are deprecated. Tasks moved out of the core into an extension.
>
> Section [3](#3-how-mcp-works-end-to-end) explains the new model, and section [12](#12-migrating-from-2025-11-25-to-2026-07-28) is a migration checklist.

---

## Table of Contents

1. [What is MCP](#1-what-is-mcp)
2. [The Building Blocks of MCP](#2-the-building-blocks-of-mcp)
3. [How MCP Works (End-to-End)](#3-how-mcp-works-end-to-end)
4. [What to Consider When Building an MCP Server](#4-what-to-consider-when-building-an-mcp-server)
5. [Why MCP and Not Just APIs](#5-why-mcp-and-not-just-apis)
6. [How Agents Interact with MCP](#6-how-agents-interact-with-mcp)
7. [Security Designs for Your MCP Server](#7-security-designs-for-your-mcp-server)
8. [How to Design MCP When You Already Have Rich APIs](#8-how-to-design-mcp-when-you-already-have-rich-apis)
9. [State Management - Life After Protocol-Level Sessions](#9-state-management---life-after-protocol-level-sessions)
10. [Architectural Best Practices for Designing an MCP Server](#10-architectural-best-practices-for-designing-an-mcp-server)
11. [The Extension Ecosystem](#11-the-extension-ecosystem)
12. [Migrating from 2025-11-25 to 2026-07-28](#12-migrating-from-2025-11-25-to-2026-07-28)
13. [References and Further Reading](#13-references-and-further-reading)

---

## 1. What is MCP

The **Model Context Protocol (MCP)** is an open, JSON-RPC 2.0 standard that gives large language model (LLM) applications a single, uniform way to call tools, read data, and use reusable prompts from any compliant external system. A useful analogy is that **MCP is the "USB-C for AI"**: one universal connector that replaces the dozens of bespoke integrations every model used to need.

### A short history

- **November 2024** - Anthropic publishes the first MCP specification (`2024-11-05`) and open-sources reference SDKs.
- **2025** - OpenAI, Google DeepMind, Microsoft, AWS, and most major IDEs adopt it. Revisions `2025-03-26` (Streamable HTTP, OAuth) and `2025-06-18` (elicitation, structured tool output) land.
- **November 2025** - The `2025-11-25` revision ships: experimental tasks, URL-mode elicitation, and further authorization work.
- **December 2025** - Anthropic donates MCP to the **Agentic AI Foundation (AAIF)** under the Linux Foundation, co-founded with Block and OpenAI, reinforcing MCP as a vendor-neutral open standard.
- **March 2026** - The 2026 roadmap names transport scalability, agent communication, governance, and enterprise readiness as priorities. A formal Extensions framework (SEP-2133) is introduced.
- **June 2026** - **Enterprise-Managed Authorization** reaches stable as an official extension, adopted by Anthropic, Microsoft, and Okta.
- **28 July 2026** - The **`2026-07-28`** revision ships: a stateless protocol core, Multi Round-Trip Requests, header-based routing, cacheable list results, authorization hardening, the extensions framework, and a formal deprecation policy. All four Tier 1 SDKs (TypeScript, Python, Go, C#) speak it on day one; the Rust SDK follows in beta.
- **August 2026** - A new roadmap sets five priority areas for the next cycle (see §11.5).

At the `2026-07-28` release, the maintainers reported close to **half a billion downloads a month** across the Tier 1 SDKs, with both the TypeScript and Python SDKs crossing **1 billion total downloads**.

### Protocol revisions at a glance

| Revision | Era | Headline change |
|---|---|---|
| `2024-11-05` | Legacy | Initial specification; stdio and HTTP+SSE transports |
| `2025-03-26` | Legacy | Streamable HTTP; OAuth 2.1 authorization; HTTP+SSE deprecated |
| `2025-06-18` | Legacy | Elicitation; structured tool output; Resource Indicators required |
| `2025-11-25` | Legacy | Experimental tasks; URL-mode elicitation |
| **`2026-07-28`** | **Modern** | **Stateless core; no handshake, no session IDs; MRTR; extensions** |

The spec now calls revisions `2025-11-25` and earlier **legacy** (they establish a session with an `initialize` handshake), and `2026-07-28` and later **modern** (version, identity, and capabilities travel as per-request metadata). An implementation that supports both is **dual-era**.

### Why it exists

Before MCP, every AI host (Claude Desktop, Cursor, ChatGPT, an internal agent) had to write a custom adapter for every external system it wanted to use. That's an **M x N integration problem** - M hosts times N tools. MCP collapses it to **M + N**: build one MCP client (per host) and one MCP server (per tool), and they all interoperate.

### The mental model

Think of MCP as having three roles:

- **Host** - the AI application the user interacts with (Claude Desktop, an IDE, a custom agent).
- **Client** - a connector living inside the host that speaks MCP to exactly one server.
- **Server** - the program that exposes capabilities (tools, data, prompts) from some external system (GitHub, your database, a SaaS API, the local filesystem).

One host typically runs many clients in parallel, one per server it's connected to.

---

## 2. The Building Blocks of MCP

MCP organizes its capabilities into a small set of named **primitives**. Knowing them cold is the foundation for everything else.

### 2.1 Server-side primitives (what your server exposes)

**Tools** - Functions the model can invoke. Each tool has a name, a natural-language description, and a JSON Schema for its inputs and outputs. Tools are *model-controlled*: the LLM decides when to call them based on the user's request. Examples: `github_create_issue`, `db_run_query`, `stripe_refund_charge`.

Since `2026-07-28`, `inputSchema` and `outputSchema` accept **any JSON Schema 2020-12 keywords** - including `oneOf`, `anyOf`, `allOf`, and `$ref` composition (SEP-2106). `structuredContent` may be any JSON value. Implementations must not auto-dereference external `$ref` URIs.

**Resources** - Read-only data the model (or the user) can pull into context. Each resource is identified by a URI (`file:///...`, `postgres://...`, `https://...`). Resources are typically *application-controlled* - the host or user decides what to attach. Examples: file contents, database rows, API responses, configuration blobs.

**Prompts** - Reusable, parameterized prompt templates that the server provides to the host. They're *user-controlled* - the user picks one from a menu (often a slash command). Examples: a "code review" template, a "summarize this incident" template.

### 2.2 Client-side primitives (what your server can ask the host to do)

These flip the direction: the server asks the client to do something. **As of `2026-07-28` they are no longer delivered as server-initiated requests.** The server cannot call the client out of band; it returns an interim result saying "I need this input," and the client retries. That pattern is [Multi Round-Trip Requests](#33-multi-round-trip-requests-mrtr).

**Elicitation** *(active)* - The server can ask the user a structured follow-up question mid-flow (e.g., "Which environment do you want to deploy to?"). This is the only client feature the modern spec still lists as active.

**Sampling** *(deprecated in `2026-07-28`)* - The server asks the client to run an LLM completion on its behalf (`sampling/createMessage`). Still functional during the deprecation window, but new implementations should integrate directly with an LLM provider API instead.

**Roots** *(deprecated in `2026-07-28`)* - The client tells the server "you may operate within these URIs." Migration path: pass directories or files via tool parameters, resource URIs, or server configuration.

Both remain in the spec until at least **2027-07-28** under the twelve-month deprecation window (SEP-2577).

### 2.3 Protocol-level primitives

**Per-request metadata (`_meta`)** - Replaces the old handshake. Every request carries `io.modelcontextprotocol/protocolVersion` (required) and `io.modelcontextprotocol/clientCapabilities` (required); clients *should* also send `io.modelcontextprotocol/clientInfo`, and servers *should* return `io.modelcontextprotocol/serverInfo` in each result's `_meta`.

**`server/discover`** - A single RPC that advertises a server's supported protocol versions, capabilities, identity, and instructions. Servers **must** implement it; clients **may** call it for up-front version selection, or skip it and handle `UnsupportedProtocolVersionError` inline.

**`subscriptions/listen`** - One long-lived POST-response stream for opted-in server-to-client change notifications. Clients opt in per type: `toolsListChanged`, `promptsListChanged`, `resourcesListChanged`, and `resourceSubscriptions` (an array of resource URIs). The server **must not** push types the client did not request, and acknowledges with `notifications/subscriptions/acknowledged` carrying a `io.modelcontextprotocol/subscriptionId`. This replaces the HTTP GET endpoint and `resources/subscribe` / `resources/unsubscribe`.

**Caching hints** - Results from `server/discover`, `tools/list`, `prompts/list`, `resources/list`, `resources/templates/list`, and `resources/read` **must** carry `ttlMs` (freshness in milliseconds, HTTP `max-age` semantics) and `cacheScope` (`"public"` or `"private"`). This is what lets clients stop polling. MRTR retries (requests carrying `inputResponses` or `requestState`) must never be cached.

**Result types** - Every result carries a required `resultType`: `"complete"` for a final result, `"input_required"` for an MRTR interim result. Results from legacy servers that omit the field **must** be treated as `"complete"`.

**Capability negotiation** - `ClientCapabilities` and `ServerCapabilities` both carry an `extensions` map for opt-in features beyond the core protocol (see §11).

**Logging** *(deprecated in `2026-07-28`)* - Log level is now set per-request via `io.modelcontextprotocol/logLevel` in `_meta`, and servers **must not** emit `notifications/message` for requests that did not include it. `logging/setLevel` is removed. Migration path: write to `stderr` on stdio, or use OpenTelemetry for structured observability.

**Tracing** - W3C Trace Context keys (`traceparent`, `tracestate`, `baggage`) are now documented conventions for `_meta`, so a tool call can join a distributed trace spanning the host, the server, and the downstream system (SEP-414).

---

## 3. How MCP Works (End-to-End)

MCP is built on **JSON-RPC 2.0** - every message is either a request, a response, or a notification, serialized as JSON. Two transports are standardized:

- **stdio** - the host spawns the server as a subprocess and they talk over stdin/stdout, one newline-delimited JSON-RPC message per line. Used for local, single-user integrations (IDE plugins, desktop apps). `stderr` is free for logging. The same framing works over Unix domain sockets or TCP for custom transports.
- **Streamable HTTP** - the server runs as a remote service. Clients POST JSON-RPC requests; the server may stream notifications related to that request back over Server-Sent Events on the same response. This is how production, multi-tenant MCP servers run.

The legacy **HTTP+SSE** transport (deprecated since `2025-03-26`) is now formally Deprecated under the lifecycle policy and is eligible for removal three months after SEP-2596 reaches Final. Do not build on it.

### 3.1 There is no session, and no handshake

This is the defining change of `2026-07-28`. Every request is self-describing:

- No `initialize` / `notifications/initialized` exchange (SEP-2575).
- No `Mcp-Session-Id` header (SEP-2567).
- `tools/list`, `resources/list`, and `prompts/list` no longer vary per connection.
- No SSE stream resumability - the `Last-Event-ID` header and SSE event IDs are gone. A broken response stream loses the in-flight request, and the client **must** re-issue it with a new request ID.
- `ping` and `notifications/roots/list_changed` are removed.

The practical consequence: **any request can land on any instance behind a plain round-robin load balancer.** No sticky routing, no shared session store, no drained sessions on redeploy.

A server that genuinely needs cross-call state mints an explicit handle and hands it back as an ordinary tool argument - a cursor, a job ID, a draft ID. State becomes part of your domain model instead of a property of the connection.

### 3.2 The lifecycle of a modern MCP exchange

```
   ┌────────┐                                    ┌────────┐
   │ Client │                                    │ Server │
   └───┬────┘                                    └────┬───┘
       │  0. server/discover  (OPTIONAL)              │
       │ ───────────────────────────────────────────► │
       │ ◄─────────────────────────────────────────── │
       │     supportedVersions, capabilities,         │
       │     serverInfo, ttlMs, cacheScope            │
       │                                              │
       │  1. tools/list                               │
       │     _meta: protocolVersion,                  │
       │            clientCapabilities, clientInfo    │
       │ ───────────────────────────────────────────► │
       │ ◄─────────────────────────────────────────── │
       │     resultType: "complete", ttlMs, cacheScope│
       │                                              │
       │  2. tools/call   (LLM decides)               │
       │     headers: Mcp-Method, Mcp-Name            │
       │ ───────────────────────────────────────────► │
       │ ◄─────────────────────────────────────────── │
       │     resultType: "complete"                   │
       │       ...or "input_required" → see §3.3      │
       │                                              │
       │  3. subscriptions/listen  (OPTIONAL)         │
       │     notifications: { toolsListChanged: true }│
       │ ───────────────────────────────────────────► │
       │ ◄─────────────────────────────────────────── │
       │     ack, then a long-lived notification stream│
```

1. **Discover (optional)** - the client calls `server/discover` to learn supported versions and capabilities up front. Or it skips this and handles `UnsupportedProtocolVersionError` (`-32022`) inline, which lists the versions the server does support.
2. **List** - `tools/list`, `resources/list`, `prompts/list`. Results are cacheable; servers **should** return tools in a deterministic order so client caches and LLM prompt caches both hit.
3. **Call** - the LLM emits a `tools/call`; the client sends it with `_meta` and the required routing headers.
4. **Resolve** - the server returns `resultType: "complete"`, or asks for more input (§3.3), or returns a task handle if the Tasks extension is in play (§11.2).
5. **Subscribe (optional)** - one `subscriptions/listen` stream carries opted-in change notifications. Request-scoped notifications like `notifications/progress` and `notifications/message` still flow on the response stream of the request they belong to, *not* on this stream.

### 3.3 Multi Round-Trip Requests (MRTR)

Server-initiated requests needed an open, sticky stream, so they could not survive a stateless core. MRTR (SEP-2322) inverts them.

When a server needs an elicitation, a sampling completion, or a roots list, it does **not** call the client. It returns an interim result:

```json
{
  "resultType": "input_required",
  "requestState": "<opaque server-encoded state>",
  "inputRequests": {
    "github_login": {
      "method": "elicitation/create",
      "params": {
        "mode": "form",
        "message": "Please provide your GitHub username",
        "requestedSchema": {
          "type": "object",
          "properties": { "name": { "type": "string" } },
          "required": ["name"]
        }
      }
    }
  }
}
```

The client gathers the answers and **retries the original request** - new request ID, same parameters - now carrying `inputResponses` keyed by the same identifiers, plus the echoed `requestState`:

```json
{
  "inputResponses": {
    "github_login": { "action": "accept", "content": { "name": "octocat" } }
  },
  "requestState": "<echoed back verbatim>"
}
```

Because `requestState` is server-minted and travels with the retry, the second call can land on a completely different instance. Note the corollary: `notifications/elicitation/complete` and the `elicitationId` field of URL-mode elicitation (both added in `2025-11-25`) are removed - a server that needs to correlate an out-of-band interaction encodes its own identifier inside `requestState`.

### 3.4 Header-based routing

Streamable HTTP POSTs **must** now carry standard headers so gateways, rate limiters, and WAFs can route and meter without parsing the JSON body (SEP-2243):

| Header | Source | Required on |
|---|---|---|
| `MCP-Protocol-Version` | the request's protocol version | all requests |
| `Mcp-Method` | `method` | all requests |
| `Mcp-Name` | `params.name` or `params.uri` | `tools/call`, `resources/read`, `prompts/get` |

```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: get_weather
```

Values that aren't safe as plain ASCII use a Base64 sentinel encoding (`=?base64?...?=`). A mismatch between a header and the body is a `HeaderMismatch` error (`-32020`). Tools may additionally promote individual parameters into HTTP headers with an `x-mcp-header` annotation in the input schema - useful for routing on a tenant or region without cracking the payload open.

### 3.5 Error codes

The JSON-RPC server-error range is now partitioned. `-32000` to `-32019` is implementation-defined and grandfathered (don't allocate new codes there); `-32020` to `-32099` belongs to the MCP specification.

| Code | Name |
|---|---|
| `-32020` | `HeaderMismatch` |
| `-32021` | `MissingRequiredClientCapability` |
| `-32022` | `UnsupportedProtocolVersion` |

"Resource not found" also moved from `-32002` to the standard `-32602` (Invalid Params).

### 3.6 Why JSON-RPC

JSON-RPC was chosen over REST or gRPC because it is transport-agnostic and frames everything as discrete messages - which makes streaming and notifications consistent across stdio and HTTP. What changed in `2026-07-28` is not the message format but the *conversation*: the protocol no longer assumes a long-lived, bidirectional, stateful channel.

---

## 4. What to Consider When Building an MCP Server

Building an MCP server is deceptively easy - a "hello world" is 50 lines. Building one that an agent actually loves to use is much harder. Here is the checklist that separates toys from production-grade servers.

### 4.1 Define the agent story first, not the endpoint list

Write down the user-and-agent stories you actually want to support: *"As a support engineer, I want the agent to find the failing customer's last three orders and refund the one tagged 'damaged'."* Then design the minimum set of tools that lets that story happen in **one or two calls**, not seven.

### 4.2 Pick the right transport

- **stdio** if your server is local-only, runs on the user's machine, and trusts the host process.
- **Streamable HTTP** if it is a remote service, multi-tenant, or behind a load balancer.

The roadmap is heading toward unifying on HTTP-native transports everywhere - including local servers speaking Streamable HTTP over stdio - so new code should avoid assumptions that only hold for one of the two.

### 4.3 Design for statelessness - it is now the protocol's default

This is no longer a deployment preference; it is what the protocol assumes. Every handler must be independently servable by any instance. If a workflow spans calls, mint an explicit handle and pass it back as a tool argument. (See §9.)

### 4.4 Budget your tool surface

In practice, large tool surfaces make selection harder for agents. **Five to eight tools per server** is the sweet spot. If you have more, split into multiple domain-scoped servers (`billing-mcp`, `inventory-mcp`, `support-mcp`) rather than one mega-server. Progressive discovery - letting a server expose a small entry point and reveal more as the conversation narrows - is an active roadmap item, but it is not in the spec yet.

### 4.5 Write tool descriptions for the LLM, not for humans

Every tool description goes straight into the model's context window. It should be:

- Action-first ("Create a new GitHub issue in the given repo...")
- Disambiguating (when do I call this vs. its sibling?)
- Honest about side effects and idempotency
- Short - typically under 200 tokens

### 4.6 Design response payloads for context economy

A REST endpoint can return 5 KB of metadata "just in case". An MCP tool must not - every byte costs tokens. Return only what the agent needs to make the next decision. Offer a `verbose` flag or a separate `get_details` tool if a power user needs more.

### 4.7 Set caching hints honestly

`ttlMs` and `cacheScope` are required on list and read results, and clients will act on them. A generous `ttlMs` on a genuinely stable catalog saves the agent a round trip on every turn. A generous `ttlMs` on a per-user, per-permission tool list is a correctness bug - and `cacheScope: "public"` on anything user-specific is a data-leak bug, because shared intermediaries may cache it. When in doubt: `"private"`, and a short TTL.

### 4.8 Handle errors as guidance, not just failures

Bad: `"403 Forbidden"`. Good: `"Permission denied. The MCP server needs an API token with the 'repo:write' scope. Ask the user to reauthenticate, or call set_token with a refreshed token."` Error messages are part of the agent's reasoning loop. Use the allocated code ranges correctly (§3.5) and never invent codes in `-32020`..`-32099`.

### 4.9 Version and evolve gracefully

Tools you ship today will be called by agents you cannot update. Treat tool names and schemas as a public API. Add new fields as optional; never repurpose an existing parameter. Return a deterministic tool order so client and prompt caches stay warm.

### 4.10 Plan your dual-era story

If you have existing clients, you will be serving legacy (`2025-11-25` and earlier) and modern (`2026-07-28`) callers at the same time for a while. Decide deliberately whether you support both, and let `server/discover` and `UnsupportedProtocolVersionError` do the negotiating. On stdio, `server/discover` doubles as a backward-compatibility probe.

### 4.11 Observability from day one

Structured logs, per-tool latency, per-tenant call counts, and a clear audit trail of "who called what, when, with which arguments, and what came back." Propagate `traceparent` from `_meta` so the trace spans the host, your server, and the downstream system. This is not optional in enterprise deployments.

---

## 5. Why MCP and Not Just APIs

The fastest way to misuse MCP is to think of it as "an API with extra steps." It isn't. MCP and REST solve different problems for different consumers.

| Dimension | REST API | MCP (`2026-07-28`) |
|---|---|---|
| **Primary consumer** | Human developer writing code | LLM agent at runtime |
| **Discovery** | Read docs at build time | `server/discover` + `tools/list` at runtime |
| **State model** | Stateless request/response | Stateless request/response (since `2026-07-28`) |
| **Cross-call state** | Your own IDs and cursors | Explicit server-minted handles as tool arguments |
| **Communication** | Client -> server only | Client -> server, plus MRTR interim results and opt-in notification streams |
| **Schema** | OpenAPI (for humans) | JSON Schema 2020-12 + natural-language descriptions (for LLMs) |
| **Error semantics** | HTTP status codes | Guidance text the model can act on, over allocated JSON-RPC codes |
| **Caching** | HTTP `Cache-Control` | `ttlMs` + `cacheScope` on list and read results |
| **Auth** | Many flavors, ad hoc | OAuth 2.1 with PKCE + Resource Indicators, CIMD registration |
| **Best for** | Deterministic system-to-system calls | Open-ended agent workflows |

Note how much of that table converged on REST in `2026-07-28`. That is the point: a remote MCP server is now, operationally, just another HTTP workload. What still distinguishes MCP is *who the consumer is* and *how the contract is described*.

### When to reach for MCP

- An agent will dynamically choose what to do, and you do not know the call sequence in advance.
- Three or more tools/data sources need to be combined inside one chat or agent runtime.
- You want the same integration to work across Claude, ChatGPT, Cursor, your in-house agent, and whatever comes next.
- You need user-consent flows for tool execution (a server can't act without the host mediating).

### When to keep using REST

- A scheduled job or backend pipeline calls a specific endpoint with known parameters.
- A mobile app needs deterministic CRUD with strict latency SLOs.
- The caller is human-written code, not a model.

The honest framing: **MCP sits on top of your APIs, it doesn't replace them.** Your REST API is still the system of record. The MCP server is the agent-friendly facade in front of it.

---

## 6. How Agents Interact with MCP

The lifecycle from the agent's point of view looks like this:

### 6.1 Connection and discovery

When the host starts, it spawns or connects to each configured MCP server. There is no handshake to run; it may call `server/discover` to pin a protocol version and read capabilities, then calls `tools/list`, `resources/list`, `prompts/list`. The discovered tool definitions - name, description, JSON Schema - are injected into the system prompt of the agent. To the LLM, they look identical to natively defined functions. Because list results carry `ttlMs`, a well-behaved host caches them rather than re-listing on every turn.

### 6.2 Decision-making

The agent receives the user's message together with the catalog of available tools. Standard tool-calling behavior takes over: the model decides whether to answer directly or to emit a tool call. Because MCP tool descriptions are written for model use, they can improve tool-selection accuracy compared with raw endpoint catalogs.

### 6.3 Invocation

The host's client formats the model's chosen call as a `tools/call` JSON-RPC request, attaches `_meta` (protocol version, client capabilities, client info) and the `Mcp-Method` / `Mcp-Name` headers, and sends it. The server executes - usually by calling the underlying API, database, or system - and returns a structured result. The result is fed back to the model as a tool message, and the loop continues.

### 6.4 Mid-flight input, progress, and notifications

A long-running tool streams `notifications/progress` on the response stream of its own request. If the server needs something from the user or the host's model, it returns `resultType: "input_required"` and the client retries with `inputResponses` (§3.3) - from the agent's perspective, one logical tool call that took two HTTP round trips. For genuinely long work, the Tasks extension returns a durable handle the client polls with `tasks/get` (§11.2). Catalog changes arrive over an opt-in `subscriptions/listen` stream.

### 6.5 Termination

There is no session to close. The client stops sending requests, and cancels any open `subscriptions/listen` stream. Nothing on the server needs to be torn down.

The key insight: **the agent never directly speaks HTTP to your API.** It speaks MCP to a client; the client speaks MCP to your server; your server speaks whatever it likes to the underlying system. That decoupling is what makes the protocol portable.

---

## 7. Security Designs for Your MCP Server

MCP servers sit between an autonomous agent and a real system holding real data, often with real money attached. Public MCP deployments have already shown familiar security failures: leaked credentials, broad tokens, weak audit trails, and prompt-injection exposure. Treat the security bar as high from day one.

### 7.1 The threat model

The five recurring attack classes:

1. **Prompt injection** - Malicious instructions hidden in data the agent reads (an issue title, an email body, a webpage) that hijack the agent into calling tools it shouldn't.
2. **Tool poisoning / "rug pulls"** - A hosted MCP server ships benign tool descriptions, then later mutates them to include hidden directives. Pinning tool definitions and verifying signatures defends against this.
3. **Over-privileged tokens** - A user grants a broad scope once, and every agent action runs with the union of all those scopes. Least-privilege scoping is essential.
4. **Credential sprawl** - Every developer spins up their own MCP server with its own API tokens stored insecurely.
5. **Audit blind spots** - No record of which agent called which tool, on whose behalf, with what arguments.

### 7.2 Authentication: OAuth 2.1 is the default

The `2026-07-28` specification defines authorization for HTTP-based transports. Authorization is optional overall, but when an HTTP MCP server protects user data, use the standard OAuth-based flow. The practical baseline:

- **Use PKCE (Proof Key for Code Exchange)** for authorization-code flows, with S256 when supported.
- **Resource Indicators (RFC 8707)** must bind every access token to the specific MCP server URI it was issued for. The server must reject tokens whose audience claim does not match its own URI. This kills cross-server token replay.
- **Validate the `iss` parameter (RFC 9207)** - new in `2026-07-28` (SEP-2468). Authorization servers **should** return `iss` in the authorization response, and clients **must** validate a present `iss` against the recorded issuer *before* redeeming the code. This closes an authorization-server mix-up hole.
- **Short-lived access tokens** (15-60 minutes) paired with refresh tokens.
- **TLS for HTTP deployments**, including internal systems unless a tightly controlled local development setup is explicitly isolated.

For machine-to-machine or local stdio servers, Bearer tokens or API keys can be acceptable, but they should still be scoped, rotated, and stored in a secret manager, never in source. There is also an official `io.modelcontextprotocol/oauth-client-credentials` extension for the M2M case.

### 7.3 Client registration: prefer CIMD over DCR

`2026-07-28` **deprecates OAuth Dynamic Client Registration (RFC 7591)** in favor of **Client ID Metadata Documents (CIMD)**. The priority order for clients that support everything:

1. Pre-registered client credentials, if you have them for that server.
2. **CIMD**, if the authorization server advertises `client_id_metadata_document_supported`.
3. DCR as a fallback, if the server advertises a `registration_endpoint`.
4. Prompt the user to enter client information.

With CIMD the `client_id` *is* an HTTPS URL with a path component (e.g. `https://app.example.com/oauth/client.json`) that serves a JSON document containing at least `client_id`, `client_name`, and `redirect_uris`. The `client_id` inside the document **must** match the URL exactly, and the authorization server **must** validate that and the presented redirect URIs. This is the answer to the very common MCP case where client and server have no prior relationship.

Two related hardening changes:

- **`application_type` in DCR** (SEP-837) - clients must declare it, so authorization servers stop rejecting `localhost` redirects for desktop and CLI apps.
- **Issuer-bound credentials** (SEP-2352) - clients **must** key persisted credentials by issuer identifier, **must not** reuse them with a different authorization server, and **must** re-register when the authorization server changes.

For organizations, the **Enterprise-Managed Authorization** extension (stable since June 2026) removes the per-server OAuth dance entirely: the client obtains an Identity Assertion JWT Authorization Grant (ID-JAG) from the IdP during SSO and exchanges it for an access token at the MCP server's authorization server. Users get the servers their group membership entitles them to on first login.

### 7.4 Authorization: scope every tool call

Authentication tells you *who* - authorization decides *what they can do*. Each tool call should re-check the caller's scopes against a policy. This matters more now, not less: with no session, there is no "already authorized at connection time," so **every single request** must be independently authenticated and authorized.

- **Role-based access control (RBAC)** - agents inherit the roles of the user they're acting on behalf of.
- **Attribute-based access control (ABAC)** - combine role, resource, and request attributes (e.g., "support agents can refund orders under $500 placed in the last 30 days").
- **Deny by default** - new tools must be explicitly granted, not implicitly allowed.

### 7.5 Defending against prompt injection and tool poisoning

- **Treat all data the agent reads as untrusted.** Tag external content so the model can be prompted to ignore embedded instructions in it.
- **Pin tool definitions.** Hash the tools list and alert (or fail closed) if it changes unexpectedly. With cacheable list results this is easier than it used to be - you already have a stable, deterministically ordered payload to hash.
- **Human-in-the-loop for destructive actions.** Refunds, deletes, money movement, and any irreversible action should require explicit user confirmation in the host UI - not just "the agent decided to."
- **Allow-list outbound calls.** If your MCP server makes outbound network calls (it usually does), restrict them to an explicit set of hostnames. Block egress to user-supplied URLs unless that is the literal point of the tool.
- **Never auto-dereference external `$ref` URIs** in tool schemas, and bound the resources you spend resolving composition keywords.

### 7.6 Treat server-minted handles as capabilities

This is new work created by the stateless core. Anything you hand back for the client to pass into a later call - a cursor, a job ID, an MRTR `requestState` blob - is now a token travelling through the model's context. Assume it will be logged, summarized, and possibly replayed.

- Sign or encrypt `requestState`; never trust its contents on the retry without verifying integrity.
- Bind every handle to the authenticated principal and re-check that binding on use, so handle A issued to user A cannot be replayed by user B.
- Give handles an expiry.
- Put nothing sensitive in them in plaintext.

### 7.7 Input validation and output sanitization

- Validate every tool input against its JSON Schema - never trust the LLM's argument generation.
- Strip secrets, PII, internal IDs, and stack traces from tool outputs before returning them. Anything you return goes into the model's context, and from there can be exfiltrated to the user or another tool.
- Length-cap every field. A 200 MB response will blow up the context window and probably the host.
- Validate promoted headers. If you use `x-mcp-header`, remember the values originate in model-generated tool arguments, and reject header mismatches rather than trusting whichever copy is convenient.

### 7.8 Auditing and observability

Log structured records for **every** tool invocation: timestamp, tenant, user, agent session, tool name, arguments (PII-redacted), result summary, latency. Ship them to a SIEM. Alert on anomalies - tool calls outside normal patterns are the earliest signal of a compromised agent or a poisoned tool. Use the `Mcp-Method` and `Mcp-Name` headers to meter and alert at the gateway, before the request even reaches your handler.

### 7.9 The compact rule

> Authenticate every request. Authorize every tool call. Validate every input. Sanitize every output. Encrypt every connection. Sign every handle. Log every action.

---

## 8. How to Design MCP When You Already Have Rich APIs

This is the most common starting point - and the most commonly bungled. The trap is to wrap one MCP tool around each REST endpoint. The result technically works, but the agent struggles: simple goals take five calls, the context fills with irrelevant fields, and tool descriptions read like a stack-trace.

### 8.1 The rule: do not mirror your API surface

Your REST API was designed for human developers who have a debugger, persistent memory, and documentation open in another tab. Your agent has none of those. Designing the MCP server is a **product exercise**, not an export job.

### 8.2 Start with the workflow, not the endpoint list

For each high-value agent workflow, ask:

1. What is the desired outcome?
2. What is the *minimum* set of tool calls that achieves it?
3. What context does each call need, and what context should it return?

A single MCP tool can - and often should - call three or four REST endpoints under the hood and return one consolidated, agent-shaped result. That is the whole point of the layer.

### 8.3 Patterns that work

**Workflow tools, not CRUD tools.** Instead of `list_issues`, `get_issue`, `update_issue`, `list_comments`, `add_comment`, expose `triage_issue` that does the obvious orchestrated thing for the common case, and keep the granular tools as escape hatches.

**Service-prefixed, action-oriented names.** Pattern: `{service}_{verb}_{noun}` - `slack_send_message`, `linear_create_issue`, `stripe_refund_charge`. This avoids collisions when many servers are loaded.

**Domain-sharded servers for big APIs.** A 400-endpoint REST API does not become one MCP server with 400 tools. It becomes 10-20 small servers, each scoped to a coherent domain (`billing`, `subscriptions`, `customers`, `disputes`), each fitting comfortably under the ~10-tool budget. Hosts load the ones a given agent needs.

**Generated scaffolding, hand-tuned surface.** Tools like Stainless, FastMCP, and Azure API Management can auto-generate tool stubs from an OpenAPI spec. Treat the generated code as a starting point, then hand-curate: rename tools, collapse endpoints, prune fields, rewrite descriptions for an LLM audience.

**Pagination, filtering, and ranking become first-class.** A REST endpoint returning 500 results is fine. An MCP tool returning 500 results is broken - the model will pick badly and the context will burn. Build server-side ranking, default page sizes around 5-20, and let the model ask for more.

**Map your API's own caching onto `ttlMs`.** If your catalog endpoint is already behind a CDN with a known max-age, you have your `ttlMs`. If it is per-user, that is your signal for `cacheScope: "private"`.

**Reach for Tasks instead of holding the connection.** If the underlying API is a long-running job API, do not block a `tools/call` on it. Return a task handle (§11.2) and let the client poll.

### 8.4 Anti-patterns to avoid

- Exposing every endpoint "for completeness."
- Letting tool responses include raw API JSON unchanged.
- Returning HTTP status codes as tool errors with no guidance.
- Requiring the agent to know internal IDs (look them up server-side from human-friendly identifiers when possible).
- Using your REST API's auth model directly instead of layering OAuth 2.1 / resource indicators on top.
- Reaching for Sampling to "just get a summary." It is deprecated - call an LLM provider directly from your server, or return the raw material and let the host's model do it.
- Storing per-connection state in a process-local map. There is no connection to key it on any more.

### 8.5 A migration path

1. Stand up a thin generated server from your OpenAPI spec to validate plumbing.
2. Instrument it: log which tools the agent actually uses for real workflows.
3. Identify the top 3-5 workflows by frequency.
4. Replace the cluster of fine-grained tools serving each workflow with one workflow tool.
5. Retire the unused fine-grained tools, or move them behind a "power user" capability flag.
6. Re-evaluate quarterly - usage patterns shift as agents and users learn what is possible.

---

## 9. State Management - Life After Protocol-Level Sessions

> **This section changed substantially in `2026-07-28`.** The protocol no longer has sessions. The stateful-vs-stateless debate that dominated MCP architecture through 2025 and early 2026 has been settled by the spec: stateless won.

### 9.1 What was removed, and why

Under `2025-11-25` and earlier, a session was the period between `initialize` and shutdown. It carried capability negotiation results, granted roots, subscriptions, cached auth context, and sometimes server-side workflow state. It was identified by an `Mcp-Session-Id` header, which meant every request in a session had to reach the same instance.

That was the single biggest obstacle to running MCP servers like ordinary web services. Sticky routing fights load balancers, blue/green deploys, serverless platforms, and Kubernetes rolling updates; a redeploy dropped every open session; capacity planning had to model concurrent sessions rather than request rate.

SEP-2567 and SEP-2575 removed the session and the handshake together. What replaced them:

| Old mechanism | Modern replacement |
|---|---|
| `initialize` / `notifications/initialized` | Per-request `_meta` (protocol version, client capabilities, client info) |
| Capabilities learned at handshake | `server/discover` (optional, cacheable) |
| `Mcp-Session-Id` header | Nothing - requests are self-describing |
| Per-connection `tools/list` results | Connection-independent, cacheable list results |
| Server-initiated requests | MRTR interim results + client retry |
| `resources/subscribe` + HTTP GET stream | `subscriptions/listen` |
| SSE resumability (`Last-Event-ID`) | Nothing - re-issue the request with a new ID |
| Long-blocking calls | Tasks extension (durable handles, polling) |

### 9.2 Where state actually lives now

State did not disappear; it became explicit. The spec's guidance is that servers needing cross-call state **use server-minted handles passed as ordinary tool arguments**. Three flavors, in ascending order of weight:

1. **Cursors and opaque tokens.** Pagination, a partially built query, a draft. The server mints an identifier, returns it in the result, and the agent passes it to the next call. Backed by Redis or your database, readable by any instance.
2. **`requestState` (MRTR).** Scoped to one logical request that needed user input. The server encodes what it needs, the client echoes it back verbatim on the retry. Sign it (§7.6).
3. **Task handles (Tasks extension).** For work that outlives a single HTTP exchange. Durable by definition - the client can crash, restart, and resume polling `tasks/get` with the same `taskId`.

The design test is simple: **could a second, freshly started instance of my server serve this request correctly, given only the request itself and shared storage?** If not, you have hidden state.

### 9.3 The practical recipe

- Handlers are pure functions of `(request, shared storage)`. No process-local session maps.
- Authenticate and authorize on **every** request. There is no connection-scoped "already checked."
- Mint explicit handles for anything spanning calls; sign them, scope them to a principal, expire them.
- Put shared state in a fast store (Redis, DynamoDB, Postgres) keyed by the handle, not by a connection.
- Hold heavy shared resources - warmed indexes, connection pools, model caches - at the process or sidecar level, **read-only**, never keyed to a caller.
- Reap stale handles and tasks on a timer, the way you would expire any other token.
- Use `ttlMs` so clients stop re-listing; use `subscriptions/listen` only for clients that genuinely need push.

### 9.4 Per-agent isolation is still your job

Removing protocol sessions did not remove the need for **isolation between agents and tenants**. It moved it into your handle design. Everything that used to be dangerous about a shared session is now dangerous about a shared handle:

- Cross-tenant data leakage if a handle isn't bound to its principal.
- Auth confusion - a handle minted under one user's scopes must never be usable under another's.
- Concurrent-modification bugs in whatever the handle points at.

The old advice still applies, restated: **share the resource, not the identity.** A warmed index in a process-level cache is fine. A handle that any caller can present is not.

### 9.5 If you still support legacy clients

Dual-era servers keep the old session machinery alive for `2025-11-25` and earlier callers while serving modern requests statelessly. Two things to watch:

- Do not let a legacy session's cached state leak into modern request handling; they are separate paths.
- Sticky routing that a legacy client still needs is a property of that one path, not of your whole deployment. Route on the era, not on the service.

---

## 10. Architectural Best Practices for Designing an MCP Server

A consolidated checklist drawn from the `2026-07-28` specification, the August 2026 roadmap, and what production teams have learned the hard way.

### 10.1 Scope and structure

- **One server, one domain.** If your server's elevator pitch needs the word "and," split it.
- **5-8 tools per server** is the sweet spot; never exceed 12 without a very good reason.
- **Service-prefixed, verb-oriented tool names** for global uniqueness across loaded servers.
- **Workflow tools over CRUD tools** - collapse common multi-step flows into single tools.

### 10.2 Schema and contracts

- **Every tool has a strict JSON Schema** for inputs and outputs. JSON Schema 2020-12 composition is now allowed - use it for clarity, not cleverness.
- **Descriptions are written for the LLM** - action-first, disambiguating, <= 200 tokens.
- **Schemas are versioned.** Add fields as optional; never repurpose existing ones.
- **Return structured content**, not blobs of prose, so the agent can pattern-match.
- **Deterministic ordering** in `tools/list` so client caches and prompt caches hit.

### 10.3 Transport and deployment

- **stdio for local single-user, Streamable HTTP for remote multi-tenant.** Do not build on HTTP+SSE.
- **Stateless by default** - it is the protocol's model, not just a deployment choice.
- **Emit the routing headers** (`MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name`) and let your gateway use them.
- **Set `ttlMs` and `cacheScope` deliberately** on every cacheable result.
- **Health checks, readiness probes, graceful shutdown.** This is a normal HTTP service; treat it like one.
- **Horizontal scaling assumed**, behind plain round-robin. If you still need sticky routing, you have hidden state.

### 10.4 Security

- **OAuth 2.1 with PKCE and Resource Indicators** for any user-data server.
- **Audience-bind every token** to your server's URI; reject anything else.
- **Validate `iss`** per RFC 9207 before redeeming an authorization code.
- **Prefer CIMD over DCR** for client registration; bind credentials to their issuer.
- **Short-lived access tokens, refresh-token-backed.**
- **Least-privilege scopes per tool**, re-checked on every request.
- **Sign, scope, and expire every server-minted handle**, including `requestState`.
- **Human confirmation for destructive operations.**
- **Allow-list outbound network calls.**
- **Strip PII and secrets from every output.**

### 10.5 Reliability

- **Idempotency keys** on side-effecting tools so the agent can safely retry. This matters more now: there is no stream resumability, so a dropped response means the client *will* re-issue the request.
- **Timeouts and circuit breakers** on every external dependency.
- **Tasks for anything slow.** Do not hold a connection open for minutes.
- **Structured, classified errors** with remediation guidance the agent can act on, using the allocated code ranges.
- **Never expose raw stack traces or internal error messages** to the model.
- **Backpressure** for streaming or long-running tools.

### 10.6 Observability and governance

- **Structured logs** for every call: tenant, user, agent identity, tool, arguments, result, latency.
- **Metrics** per tool: call count, p50/p95/p99 latency, error rate.
- **Distributed traces** via `traceparent` in `_meta`, spanning the agent, the MCP server, and the downstream system.
- **Audit trail** suitable for compliance review - who did what, when, on whose behalf.
- **Tool-definition pinning** so a `tools/list_changed` notification triggers review rather than silent acceptance.
- **`stderr` or OpenTelemetry, not the Logging primitive** - it is deprecated.

### 10.7 Developer experience

- **A README that opens with three concrete agent prompts** that should "just work."
- **A `tools/list` that reads naturally** - read it out loud; if it doesn't make sense, the agent won't either.
- **A local dev mode** (stdio + a fake auth provider) that runs against the real server logic.
- **Contract tests** that verify the JSON Schema of every tool's response, run on every commit.
- **A conformance test run** against the official suite (SEP-2484) - Standards Track SEPs can no longer reach Final without matching test scenarios, and the same discipline is worth borrowing.

### 10.8 Evolution

- **Quarterly review** of which tools agents actually call, removing the ones that aren't used.
- **Capability flags and extensions** so a beta feature can be exposed to a subset of clients before becoming default. Extensions are always opt-in and disabled by default.
- **A deprecation policy** - tools live for a documented minimum, then are removed with notice. The spec's own policy (Active → Deprecated → Removed, twelve-month minimum window) is a reasonable template.

---

## 11. The Extension Ecosystem

`2026-07-28` formalized **extensions** (SEP-2133) as the way to add capability without destabilizing the core. Understanding this is now part of understanding MCP, because several things that used to be core features live here.

### 11.1 How extensions work

- Identified by reverse-DNS: `{vendor-prefix}/{extension-name}`. Official ones use `io.modelcontextprotocol/`; third parties use a domain they own (`com.example/my-extension`).
- Negotiated through the `extensions` map in `ClientCapabilities` and `ServerCapabilities`.
- Versioned independently of the core spec.
- **Always disabled by default** and require explicit opt-in from the developer. SDKs may or may not implement a given extension; check the SDK docs.
- Official extensions live in `ext-`-prefixed repositories in the MCP GitHub org; incubating ones live in `experimental-ext-` repositories tied to a Working Group.

### 11.2 Tasks (`io.modelcontextprotocol/tasks`)

Experimental tasks left the core protocol and became an extension (SEP-2663), reshaped around statelessness. Use it for anything that won't finish inside one HTTP exchange: CI pipelines, batch jobs, human approvals.

- The client opts in once via the extension capability; the server decides **per request** whether to create a task.
- A supported request returns a `CreateTaskResult` (`resultType: "task"`) with a `taskId`, initial status, TTL, and suggested polling interval. The task is durably created *before* the response is sent.
- The client polls `tasks/get`. Statuses: `working`, `input_required`, `completed`, `failed`, `cancelled`.
- Mid-flight input arrives as an `inputRequests` map on a `tasks/get` response; the client answers with **`tasks/update`** - no second connection, no unsolicited server-to-client message.
- `tasks/cancel` cancels. `tasks/result` (the old blocking method) and `tasks/list` are gone; the latter has no coherent scope without sessions.
- Because the handle is durable, a client can disconnect, restart, and resume polling with the same `taskId`.

Maturing Tasks enough to fold it back into the specification is an explicit roadmap goal.

### 11.3 MCP Apps (`ext-apps`)

Servers ship interactive HTML interfaces - charts, forms, dashboards, video players - that the host renders inline in the conversation inside a **sandboxed iframe**. A tool declares a UI resource via `_meta.ui.resourceUri` in its description; the host preloads and renders it in place of a plain text result.

Why this beats linking to a web app: the UI stays in the conversation next to the discussion that produced it; the app can call tools on the MCP server and receive pushed results over existing MCP patterns; it can delegate actions to the host and reuse the user's already-connected integrations; and the sandbox means a host can render a third-party app without fully trusting its author. Actions flow through the same JSON-RPC path as direct tool calls, so the audit trail stays consistent.

### 11.4 Authorization extensions (`ext-auth`) and Skills (`ext-skills`)

- **OAuth Client Credentials** - the machine-to-machine flow, for callers with no human in the loop.
- **Enterprise-Managed Authorization** - stable since June 2026; ID-JAG token exchange during SSO so an organization provisions MCP access centrally through its IdP (§7.3).
- **Skills over MCP** - discover Agent Skills (workflow instructions and their supporting files) through MCP resources.

### 11.5 The road ahead (August 2026 roadmap)

Five priority areas guide the next release cycle. SEPs inside them get expedited review:

1. **Agentic messaging primitives** - server-initiated events (webhooks and channels) so clients stop polling for results; a composition review across the Agents, Transports, and Triggers & Events Working Groups; maturing Tasks toward core inclusion.
2. **HTTP-native transport unification and hardening** - stretching the "an MCP server is just an HTTP workload" model to every deployment mode, including local servers speaking Streamable HTTP over stdio.
3. **Agent identity and enterprise-ready security** - moving past "a person approves in a browser" to workloads with their own identity: finalizing DPoP (RFC 9449), Workload Identity Federation, the ID-JAG grant, and standard token exchange, with continued engagement in the IETF OAuth and WIMSE working groups.
4. **Improved primitives** - one clear contract for tool result handling (today a `tools/call` response can carry the same output in more than one form, and the server author can't know which the client will show the model), plus **progressive discovery** so a server can offer a small entry point and reveal more of its catalog as the conversation narrows.
5. **Improved SDK developer experience** - ergonomics, conformance testing, and documentation, which matter more now that many developers build MCP servers by pointing an agent at the libraries.

The **Server Card** Working Group is separately working on `.well-known` metadata conventions, so a server can be discovered and reasoned about without connecting to it.

---

## 12. Migrating from 2025-11-25 to 2026-07-28

A practical checklist. Budget real time for this - it is a breaking change, though the SDKs absorb much of it.

### 12.1 Server checklist

- [ ] **Delete the handshake.** Remove `initialize` / `notifications/initialized` handling. Read protocol version and client capabilities from `_meta` on every request.
- [ ] **Implement `server/discover`.** It is mandatory. Return `supportedVersions`, `capabilities`, `instructions`, `serverInfo` in `_meta`, plus `ttlMs` / `cacheScope`.
- [ ] **Remove all per-connection state.** Delete session maps keyed by `Mcp-Session-Id`. Mint explicit handles instead, and sign them.
- [ ] **Stop making server-initiated requests.** Rewrite every `elicitation/create`, `sampling/createMessage`, and `roots/list` call site to return `resultType: "input_required"` with an `inputRequests` map and a `requestState` blob, and to resume from `inputResponses` on the retry.
- [ ] **Add `resultType` to every result.** `"complete"` unless it's an MRTR interim result.
- [ ] **Add `ttlMs` and `cacheScope`** to `server/discover`, `tools/list`, `prompts/list`, `resources/list`, `resources/templates/list`, and `resources/read`. Never set them on interim results, and never cache an MRTR retry.
- [ ] **Return tools in a deterministic order.**
- [ ] **Replace `resources/subscribe` and the HTTP GET endpoint** with `subscriptions/listen`, honoring the client's per-type opt-in and sending the acknowledgment first.
- [ ] **Drop `ping`, `logging/setLevel`, and `notifications/roots/list_changed`.** Read `io.modelcontextprotocol/logLevel` from `_meta`, and emit `notifications/message` only for requests that set it.
- [ ] **Drop SSE resumability.** Remove `Last-Event-ID` handling and event IDs. Make side-effecting tools idempotent, because clients will re-issue lost requests.
- [ ] **Validate the routing headers** (`MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name`) and return `HeaderMismatch` (`-32020`) on a mismatch.
- [ ] **Renumber error codes.** `-32001` → `-32020`, `-32003` → `-32021`, `-32004` → `-32022`; resource-not-found `-32002` → `-32602`. Stop allocating in `-32000`..`-32019`.
- [ ] **Move long-running work to the Tasks extension** rather than blocking.
- [ ] **Re-check auth on every request.** There is no connection-scoped authorization any more.

### 12.2 Client checklist

- [ ] **Send `_meta` on every request**: `protocolVersion` and `clientCapabilities` are required; send `clientInfo` too.
- [ ] **Handle `UnsupportedProtocolVersionError` (`-32022`)** by picking a version from the `supported` list and retrying.
- [ ] **Implement the MRTR retry loop**: on `resultType: "input_required"`, fulfil each entry in `inputRequests`, then re-issue the original request with a **new request ID**, the same params, `inputResponses`, and the echoed `requestState`.
- [ ] **Treat a missing `resultType` as `"complete"`** so legacy servers keep working.
- [ ] **Honor `ttlMs` and `cacheScope`.** Never serve a cached response across a different method or params, and never cache an MRTR retry result.
- [ ] **Send the routing headers**, mirroring body values exactly, with the Base64 sentinel encoding where needed.
- [ ] **Mirror `x-mcp-header` annotations** from tool schemas into HTTP headers on HTTP transports.
- [ ] **Re-issue, don't resume**, when a response stream breaks.
- [ ] **Migrate client registration to CIMD** where the authorization server supports it; set `application_type` if you still fall back to DCR; key persisted credentials by issuer and re-register when the issuer changes.
- [ ] **Validate `iss`** before redeeming an authorization code.

### 12.3 Things you can stop doing

- Provisioning sticky sessions at the load balancer.
- Running a shared session store just to survive a redeploy.
- Holding an SSE stream open to receive elicitations.
- Polling `tools/list` on every turn.

### 12.4 Things not to rush

Roots, Sampling, Logging, DCR, and the `includeContext` values `"thisServer"` / `"allServers"` are **deprecated, not removed**. They remain fully functional, and the earliest any of them can be removed is the first revision released on or after **2027-07-28**. Do not add new dependencies on them; do plan the migration; don't panic-rewrite working code this quarter.

---

## 13. References and Further Reading

### Official spec and roadmap

- [Model Context Protocol - Specification (2026-07-28)](https://modelcontextprotocol.io/specification/2026-07-28)
- [Key Changes: 2025-11-25 → 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [Versioning and Compatibility](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)
- [Multi Round-Trip Requests](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [Subscriptions](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions)
- [Caching](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/caching)
- [Streamable HTTP transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [`server/discover`](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [Deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)
- [Feature lifecycle and deprecation policy](https://modelcontextprotocol.io/community/feature-lifecycle)
- [The 2026-07-28 Specification - MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [The New MCP Roadmap (August 2026) - MCP Blog](https://blog.modelcontextprotocol.io/posts/mcp-roadmap/)
- [Specification GitHub repository](https://github.com/modelcontextprotocol/modelcontextprotocol)

### Extensions

- [Extensions overview](https://modelcontextprotocol.io/extensions/overview)
- [Tasks extension](https://modelcontextprotocol.io/extensions/tasks/overview) ([ext-tasks](https://github.com/modelcontextprotocol/ext-tasks))
- [MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview) ([ext-apps](https://github.com/modelcontextprotocol/ext-apps))
- [Skills over MCP](https://modelcontextprotocol.io/extensions/skills/overview) ([ext-skills](https://github.com/modelcontextprotocol/ext-skills))
- [Authorization extensions](https://github.com/modelcontextprotocol/ext-auth)
- [Client implementation matrix](https://modelcontextprotocol.io/extensions/client-matrix)

### Authorization and identity

- [Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [Client Registration (CIMD, pre-registration, DCR)](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration)
- [Authorization security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)
- [Enterprise-Managed Authorization: Zero-touch OAuth for MCP - MCP Blog](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/)
- [RFC 9207 - OAuth 2.0 Authorization Server Issuer Identification](https://datatracker.ietf.org/doc/html/rfc9207)
- [RFC 8707 - Resource Indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707)
- [RFC 9449 - OAuth 2.0 Demonstrating Proof of Possession (DPoP)](https://www.rfc-editor.org/rfc/rfc9449)
- [OAuth Client ID Metadata Document (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document-00)

### SDKs

- [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Go SDK](https://github.com/modelcontextprotocol/go-sdk)
- [C# SDK](https://github.com/modelcontextprotocol/csharp-sdk)
- [Rust SDK](https://github.com/modelcontextprotocol/rust-sdk) (beta support for `2026-07-28`)
- [Beta SDKs for the 2026-07-28 Release Candidate - MCP Blog](https://blog.modelcontextprotocol.io/posts/sdk-betas-2026-07-28/)

### Vendor and platform guides

- [What is Model Context Protocol (MCP)? - Google Cloud](https://cloud.google.com/discover/what-is-model-context-protocol)
- [Expose REST API as MCP server - Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/export-rest-mcp-server)
- [MCP protocol contract - AWS Bedrock AgentCore docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp-protocol-contract.html)
- [Protecting against indirect prompt injection - Microsoft for Developers](https://developer.microsoft.com/blog/protecting-against-indirect-injection-attacks-mcp)
- [MCP security: authentication and authorization - Red Hat](https://www.redhat.com/en/blog/mcp-security-implementing-robust-authentication-and-authorization)

### Design and best practices

- [15 Best Practices for Building MCP Servers in Production - The New Stack](https://thenewstack.io/15-best-practices-for-building-mcp-servers-in-production/)
- [Top 5 MCP Server Best Practices - Docker](https://www.docker.com/blog/mcp-server-best-practices/)
- [MCP is Not the Problem, It's Your Server - Phil Schmid](https://www.philschmid.de/mcp-best-practices)
- [Stop Converting Your REST APIs to MCP - Mostly Harmless](https://jlowin.dev/blog/stop-converting-rest-apis-to-mcp)
- [Designing an MCP server from a REST API - WorkOS](https://workos.com/blog/designing-mcp-server-from-rest-api)
- [From REST API to MCP Server - Stainless](https://www.stainless.com/mcp/from-rest-api-to-mcp-server)
- [Should you wrap MCP around your existing API? - Scalekit](https://www.scalekit.com/blog/wrap-mcp-around-existing-api)

### Comparisons and explainers

- [MCP vs. REST: What's the right way to connect AI agents to your API? - WorkOS](https://workos.com/blog/mcp-vs-rest)
- [A Deep Dive Into MCP and the Future of AI Tooling - a16z](https://a16z.com/a-deep-dive-into-mcp-and-the-future-of-ai-tooling/)
- [Why Model Context Protocol uses JSON-RPC - Daniel Avila](https://medium.com/@dan.avila7/why-model-context-protocol-uses-json-rpc-64d466112338)

### Security deep dives

- [Security Best Practices - Model Context Protocol](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)
- [MCP Security: The Complete Guide - Practical DevSecOps](https://www.practical-devsecops.com/mcp-security-guide/)
- [MCP Security Vulnerabilities: Prompt Injection & Tool Poisoning - Practical DevSecOps](https://www.practical-devsecops.com/mcp-security-vulnerabilities/)
- [New Prompt Injection Attack Vectors Through MCP Sampling - Palo Alto Unit 42](https://unit42.paloaltonetworks.com/model-context-protocol-attack-vectors/)
- [Top 10 MCP Security Risks - Prompt Security](https://prompt.security/blog/top-10-mcp-security-risks)

---

*This document is meant to be a living reference. The protocol is moving fast - re-check the official spec link before quoting any specific RFC requirement, and follow the roadmap link for what's landing next.*
