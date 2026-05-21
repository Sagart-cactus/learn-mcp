# MCP Knowledgebase

> A practitioner's guide to the Model Context Protocol - what it is, how it works, how to build it well, and how to secure it.
>
> *Last updated: May 2026*

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
9. [Session Management - Shared vs. Per-Agent](#9-session-management--shared-vs-per-agent)
10. [Architectural Best Practices for Designing an MCP Server](#10-architectural-best-practices-for-designing-an-mcp-server)
11. [References and Further Reading](#11-references-and-further-reading)

---

## 1. What is MCP

The **Model Context Protocol (MCP)** is an open, JSON-RPC 2.0 standard that gives large language model (LLM) applications a single, uniform way to call tools, read data, and use reusable prompts from any compliant external system. A useful analogy is that **MCP is the "USB-C for AI"**: one universal connector that replaces the dozens of bespoke integrations every model used to need.

### A short history

- **November 2024** - Anthropic publishes the first MCP specification and open-sources reference SDKs.
- **2025** - OpenAI, Google DeepMind, Microsoft, AWS, and most major IDEs adopt it.
- **December 2025** - Anthropic donates MCP to the **Agentic AI Foundation (AAIF)** under the Linux Foundation, co-founded with Block and OpenAI, reinforcing MCP as a vendor-neutral open standard.
- **2026** - Anthropic reports 97M+ monthly SDK downloads across Python and TypeScript, and 10,000+ active public MCP servers across the ecosystem.

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

**Resources** - Read-only data the model (or the user) can pull into context. Each resource is identified by a URI (`file:///...`, `postgres://...`, `https://...`). Resources are typically *application-controlled* - the host or user decides what to attach. Examples: file contents, database rows, API responses, configuration blobs.

**Prompts** - Reusable, parameterized prompt templates that the server provides to the host. They're *user-controlled* - the user picks one from a menu (often a slash command). Examples: a "code review" template, a "summarize this incident" template.

### 2.2 Client-side primitives (what your server can ask the host to do)

These flip the direction: the server calls back into the client.

**Sampling** - The server asks the client to run an LLM completion on its behalf (`sampling/createMessage`). This lets an agentic server reason without shipping its own model API key. The user must always see and approve what the server is asking for.

**Roots** - The client tells the server "you may operate within these URIs." Roots are filesystem paths, repo URLs, namespaces - boundaries the server must respect. Roots can be updated dynamically as the user switches projects.

**Elicitation** - The server can ask the user a structured follow-up question mid-flow (e.g., "Which environment do you want to deploy to?").

### 2.3 Protocol-level primitives

**Notifications** - Server-pushed events ("tools changed", "resource updated", "log line emitted"). The host doesn't have to poll.

**Capability negotiation** - At connection time, client and server exchange capability declarations so each side knows what features the other supports.

**Logging** - A standardized channel for the server to stream structured log records back to the client, with levels (`debug`, `info`, `warning`, `error`).

---

## 3. How MCP Works (End-to-End)

MCP is built on **JSON-RPC 2.0** - every message is either a request, a response, or a notification, serialized as JSON. Two transports are standardized:

- **stdio** - the host spawns the server as a subprocess and they talk over stdin/stdout. Used for local, single-user integrations (IDE plugins, desktop apps).
- **Streamable HTTP** - the server runs as a remote service. Clients POST JSON-RPC requests; the server may stream responses (and unsolicited notifications) back over Server-Sent Events (SSE) on the same connection. This is how production, multi-tenant MCP servers run.

### 3.1 The lifecycle of an MCP session

```
   ┌────────┐                          ┌────────┐
   │ Client │                          │ Server │
   └───┬────┘                          └────┬───┘
       │  1. initialize (capabilities)      │
       │ ─────────────────────────────────► │
       │                                    │
       │  2. initialize result (caps)       │
       │ ◄───────────────────────────────── │
       │                                    │
       │  3. notifications/initialized      │
       │ ─────────────────────────────────► │
       │                                    │
       │  4. tools/list, resources/list ...   │
       │ ─────────────────────────────────► │
       │ ◄───────────────────────────────── │
       │                                    │
       │  5. tools/call  (LLM decides)      │
       │ ─────────────────────────────────► │
       │                                    │
       │  6. result + structured content    │
       │ ◄───────────────────────────────── │
       │                                    │
       │  7. notifications (tools/changed...) │
       │ ◄───────────────────────────────── │
       │                                    │
       │  8. shutdown                       │
       │ ─────────────────────────────────► │
```

1. **Initialize** - client and server exchange protocol version and capability flags.
2. **Discover** - client calls `tools/list`, `resources/list`, `prompts/list`.
3. **Operate** - the LLM, given the discovered toolset, decides to call `tools/call` with arguments matching the tool's schema.
4. **Stream** - server returns a structured result (text, JSON, embedded resources, or image content).
5. **Notify** - at any point, either side can push a notification (e.g., "tool list changed", "log line").
6. **Shutdown** - graceful close.

### 3.2 Why JSON-RPC

JSON-RPC was chosen over REST or gRPC because it is transport-agnostic, supports bidirectional notifications natively, and frames everything as discrete messages - which makes streaming, batching, and notifications trivial to implement consistently across stdio and HTTP.

---

## 4. What to Consider When Building an MCP Server

Building an MCP server is deceptively easy - a "hello world" is 50 lines. Building one that an agent actually loves to use is much harder. Here is the checklist that separates toys from production-grade servers.

### 4.1 Define the agent story first, not the endpoint list

Write down the user-and-agent stories you actually want to support: *"As a support engineer, I want the agent to find the failing customer's last three orders and refund the one tagged 'damaged'."* Then design the minimum set of tools that lets that story happen in **one or two calls**, not seven.

### 4.2 Pick the right transport

- **stdio** if your server is local-only, runs on the user's machine, and trusts the host process.
- **Streamable HTTP** if it is a remote service, multi-tenant, or behind a load balancer.

### 4.3 Choose stateful vs. stateless deliberately

(See §9 for the full discussion.) The short version: prefer stateless request handling and externalize any session state to Redis/DynamoDB/Postgres unless you have a hard reason to be stateful (long-lived authenticated workflows, very large preloaded context).

### 4.4 Budget your tool surface

In practice, large tool surfaces make selection harder for agents. **Five to eight tools per server** is the sweet spot. If you have more, split into multiple domain-scoped servers (`billing-mcp`, `inventory-mcp`, `support-mcp`) rather than one mega-server.

### 4.5 Write tool descriptions for the LLM, not for humans

Every tool description goes straight into the model's context window. It should be:

- Action-first ("Create a new GitHub issue in the given repo...")
- Disambiguating (when do I call this vs. its sibling?)
- Honest about side effects and idempotency
- Short - typically under 200 tokens

### 4.6 Design response payloads for context economy

A REST endpoint can return 5 KB of metadata "just in case". An MCP tool must not - every byte costs tokens. Return only what the agent needs to make the next decision. Offer a `verbose` flag or a separate `get_details` tool if a power user needs more.

### 4.7 Handle errors as guidance, not just failures

Bad: `"403 Forbidden"`. Good: `"Permission denied. The MCP server needs an API token with the 'repo:write' scope. Ask the user to reauthenticate, or call set_token with a refreshed token."` Error messages are part of the agent's reasoning loop.

### 4.8 Version and evolve gracefully

Tools you ship today will be called by agents you cannot update. Treat tool names and schemas as a public API. Add new fields as optional; never repurpose an existing parameter.

### 4.9 Observability from day one

Structured logs, per-tool latency, per-tenant call counts, and a clear audit trail of "who called what, when, with which arguments, and what came back." This is not optional in enterprise deployments.

---

## 5. Why MCP and Not Just APIs

The fastest way to misuse MCP is to think of it as "an API with extra steps." It isn't. MCP and REST solve different problems for different consumers.

| Dimension | REST API | MCP |
|---|---|---|
| **Primary consumer** | Human developer writing code | LLM agent at runtime |
| **Discovery** | Read docs at build time | `tools/list` at runtime |
| **State model** | Stateless request/response | Stateful JSON-RPC session (often) |
| **Communication** | Client -> server only | Bidirectional (server can push, sample, elicit) |
| **Schema** | OpenAPI (for humans) | JSON Schema + natural-language descriptions (for LLMs) |
| **Error semantics** | HTTP status codes | Guidance text the model can act on |
| **Auth** | Many flavors, ad hoc | OAuth 2.1 with PKCE + Resource Indicators (standard) |
| **Best for** | Deterministic system-to-system calls | Open-ended agent workflows |

### When to reach for MCP

- An agent will dynamically choose what to do, and you do not know the call sequence in advance.
- Three or more tools/data sources need to be combined inside one chat or agent runtime.
- You want the same integration to work across Claude, ChatGPT, Cursor, your in-house agent, and whatever comes next.
- You need user-consent flows for tool execution (a server can't silently call back without the host mediating).

### When to keep using REST

- A scheduled job or backend pipeline calls a specific endpoint with known parameters.
- A mobile app needs deterministic CRUD with strict latency SLOs.
- The caller is human-written code, not a model.

The honest framing: **MCP sits on top of your APIs, it doesn't replace them.** Your REST API is still the system of record. The MCP server is the agent-friendly facade in front of it.

---

## 6. How Agents Interact with MCP

The lifecycle from the agent's point of view looks like this:

### 6.1 Connection and discovery

When the host starts, it spawns or connects to each configured MCP server, runs the initialize handshake, and calls `tools/list`, `resources/list`, `prompts/list`. The discovered tool definitions - name, description, JSON Schema - are injected into the system prompt of the agent. To the LLM, they look identical to natively defined functions.

### 6.2 Decision-making

The agent receives the user's message together with the catalog of available tools. Standard tool-calling behavior takes over: the model decides whether to answer directly or to emit a tool call. Because MCP tool descriptions are written for model use, they can improve tool-selection accuracy compared with raw endpoint catalogs.

### 6.3 Invocation

The host's client formats the model's chosen call as a `tools/call` JSON-RPC request and sends it to the server. The server executes (which usually means calling the underlying API, database, or system) and returns a structured result. The result is fed back to the model as a tool message, and the loop continues.

### 6.4 Streaming, notifications, and callbacks

A long-running tool can stream partial progress over SSE. The server can push `notifications/tools/list_changed` if its capabilities mutate (e.g., a new permission unlocks new tools). It can call `sampling/createMessage` to get the host's LLM to classify or summarize something, or `elicitation/create` to ask the user a question.

### 6.5 Termination

When the host shuts down or the user removes the integration, the client gracefully closes the session.

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

The 2025-11-25 MCP specification defines authorization for HTTP-based transports. Authorization is optional overall, but when an HTTP MCP server protects user data, use the standard OAuth-based flow. The practical baseline:

- **Use PKCE (Proof Key for Code Exchange)** for authorization-code flows, with S256 when supported.
- **Resource Indicators (RFC 8707)** must bind every access token to the specific MCP server URI it was issued for. The server must reject tokens whose audience claim does not match its own URI. This kills cross-server token replay.
- **Short-lived access tokens** (15-60 minutes) paired with refresh tokens.
- **TLS for HTTP deployments**, including internal systems unless a tightly controlled local development setup is explicitly isolated.

For machine-to-machine or local stdio servers, Bearer tokens or API keys can be acceptable, but they should still be scoped, rotated, and stored in a secret manager, never in source.

### 7.3 Authorization: scope every tool call

Authentication tells you *who* - authorization decides *what they can do*. Each tool call should re-check the caller's scopes against a policy:

- **Role-based access control (RBAC)** - agents inherit the roles of the user they're acting on behalf of.
- **Attribute-based access control (ABAC)** - combine role, resource, and request attributes (e.g., "support agents can refund orders under $500 placed in the last 30 days").
- **Deny by default** - new tools must be explicitly granted, not implicitly allowed.

### 7.4 Defending against prompt injection and tool poisoning

- **Treat all data the agent reads as untrusted.** Tag external content so the model can be prompted to ignore embedded instructions in it.
- **Pin tool definitions.** Hash the tools list at first connection; alert (or fail closed) if it changes unexpectedly mid-session.
- **Human-in-the-loop for destructive actions.** Refunds, deletes, money movement, and any irreversible action should require explicit user confirmation in the host UI - not just "the agent decided to."
- **Allow-list outbound calls.** If your MCP server makes outbound network calls (it usually does), restrict them to an explicit set of hostnames. Block egress to user-supplied URLs unless that is the literal point of the tool.

### 7.5 Input validation and output sanitization

- Validate every tool input against its JSON Schema - never trust the LLM's argument generation.
- Strip secrets, PII, internal IDs, and stack traces from tool outputs before returning them. Anything you return goes into the model's context, and from there can be exfiltrated to the user or another tool.
- Length-cap every field. A 200 MB response will blow up the context window and probably the host.

### 7.6 Auditing and observability

Log structured records for **every** tool invocation: timestamp, tenant, user, agent session, tool name, arguments (PII-redacted), result summary, latency. Ship them to a SIEM. Alert on anomalies - tool calls outside normal patterns are the earliest signal of a compromised agent or a poisoned tool.

### 7.7 The compact rule

> Authenticate every request. Authorize every tool call. Validate every input. Sanitize every output. Encrypt every connection. Log every action.

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

### 8.4 Anti-patterns to avoid

- Exposing every endpoint "for completeness."
- Letting tool responses include raw API JSON unchanged.
- Returning HTTP status codes as tool errors with no guidance.
- Requiring the agent to know internal IDs (look them up server-side from human-friendly identifiers when possible).
- Using your REST API's auth model directly instead of layering OAuth 2.1 / resource indicators on top.

### 8.5 A migration path

1. Stand up a thin generated server from your OpenAPI spec to validate plumbing.
2. Instrument it: log which tools the agent actually uses for real workflows.
3. Identify the top 3-5 workflows by frequency.
4. Replace the cluster of fine-grained tools serving each workflow with one workflow tool.
5. Retire the unused fine-grained tools, or move them behind a "power user" capability flag.
6. Re-evaluate quarterly - usage patterns shift as agents and users learn what is possible.

---

## 9. Session Management - Shared vs. Per-Agent

Session design is one of the most consequential decisions in MCP architecture today. The official 2026 roadmap names transport scalability, stateless operation, and explicit session handling as priority areas.

### 9.1 What "session" even means in MCP

A session is the period between `initialize` and `shutdown`. It carries:

- Capability negotiation results (so subsequent calls are compact and fast).
- Roots the client has granted.
- Subscriptions to resources or notifications.
- Cached authentication / authorization context.
- Sometimes server-side workflow state (a long-running plan, a partially filled form).

### 9.2 Stateful sessions - the benefits

- **Performance** - the initial handshake is paid once; every subsequent call is small and fast.
- **Continuity** - multi-step workflows can keep server-side state (a partially constructed query, a paginated cursor, an opened transaction).
- **Bidirectional flow** - notifications, sampling, and elicitation all rely on the connection staying open.

### 9.3 Stateful sessions - the costs

- **Sticky routing** - every request from a session has to land on the same server instance, which fights load balancers, blue/green deploys, serverless platforms, and Kubernetes rolling updates.
- **Restart fragility** - a redeploy drops all sessions unless you externalize state.
- **Horizontal scaling** - capacity planning has to account for concurrent open sessions, not just request rate.

Managed runtimes may require stricter session behavior than the base protocol. For example, some production platforms push state into external stores such as DynamoDB, Redis, S3, or managed memory services so HTTP workers can scale horizontally.

### 9.4 The emerging consensus

For most server authors, the right default in 2026 is:

1. **Default to stateless request handling.** Each `tools/call` should be independently servable by any instance.
2. **Externalize any necessary state.** Put session metadata, cursors, and workflow checkpoints in a fast shared store (Redis is the common choice).
3. **Reserve true stateful sessions for narrow cases** that genuinely need them - long-running interactive workflows, very expensive preloaded context, server-initiated callbacks.
4. **Watch the 2026 roadmap and SEPs** for the evolving stateless and migratable session model.

### 9.5 Shared session vs. one session per agent

This is a different axis from stateful/stateless, and it matters separately.

**One session per agent / user / chat (default and recommended).** Each agent session gets its own MCP session. State is isolated. A misbehaving agent can't pollute another's context. Auth scopes are bound to the calling user. This is what virtually all production hosts do.

**Shared session across agents.** A single long-lived MCP session reused by multiple agents. Tempting for caching expensive setup (large preloaded indexes, warmed model caches) but dangerous:

- Cross-tenant data leakage if you cache the wrong thing.
- Auth confusion - whose scopes apply to a given call?
- Concurrent-modification bugs in any shared workflow state.

If you genuinely need to share expensive resources, **share the resource, not the session.** Hold the warmed index in a process-level cache or a sidecar, but give each agent its own MCP session whose handlers consult that shared resource read-only.

### 9.6 Practical recipe

- One MCP session per `(user, agent_session)` pair.
- Server handles each request statelessly; session metadata in Redis keyed by the session ID.
- Authentication context re-validated on every request (don't trust an old cached token).
- Heavy shared resources held outside the session lifecycle.
- A session-cleanup job that reaps idle sessions after a sensible timeout (15 minutes of inactivity is a common default).

---

## 10. Architectural Best Practices for Designing an MCP Server

A consolidated checklist drawn from the official spec, the 2026 roadmap, and what production teams have learned the hard way.

### 10.1 Scope and structure

- **One server, one domain.** If your server's elevator pitch needs the word "and," split it.
- **5-8 tools per server** is the sweet spot; never exceed 12 without a very good reason.
- **Service-prefixed, verb-oriented tool names** for global uniqueness across loaded servers.
- **Workflow tools over CRUD tools** - collapse common multi-step flows into single tools.

### 10.2 Schema and contracts

- **Every tool has a strict JSON Schema** for inputs and outputs.
- **Descriptions are written for the LLM** - action-first, disambiguating, <= 200 tokens.
- **Schemas are versioned.** Add fields as optional; never repurpose existing ones.
- **Return structured content**, not blobs of prose, so the agent can pattern-match.

### 10.3 Transport and deployment

- **stdio for local single-user, Streamable HTTP for remote multi-tenant.**
- **Stateless request handling by default**, with externalized session state.
- **Health checks, readiness probes, graceful shutdown.** This is a normal service; treat it like one.
- **Horizontal scaling assumed**, even if you start with one instance.

### 10.4 Security

- **OAuth 2.1 with PKCE and Resource Indicators** for any user-data server.
- **Audience-bind every token** to your server's URI; reject anything else.
- **Short-lived access tokens, refresh-token-backed.**
- **Least-privilege scopes per tool.**
- **Human confirmation for destructive operations.**
- **Allow-list outbound network calls.**
- **Strip PII and secrets from every output.**

### 10.5 Reliability

- **Idempotency keys** on side-effecting tools so the agent can safely retry.
- **Timeouts and circuit breakers** on every external dependency.
- **Structured, classified errors** (`CLIENT_ERROR`, `SERVER_ERROR`, `EXTERNAL_ERROR`) with remediation guidance the agent can act on.
- **Never expose raw stack traces or internal error messages** to the model.
- **Backpressure** for streaming or long-running tools.

### 10.6 Observability and governance

- **Structured logs** for every call: tenant, user, agent session, tool, arguments, result, latency.
- **Metrics** per tool: call count, p50/p95/p99 latency, error rate.
- **Distributed traces** spanning the agent, the MCP server, and the downstream system.
- **Audit trail** suitable for compliance review - who did what, when, on whose behalf.
- **Tool-definition pinning** so a `tools/list_changed` notification triggers review rather than silent acceptance.

### 10.7 Developer experience

- **A README that opens with three concrete agent prompts** that should "just work."
- **A `tools/list` that reads naturally** - read it out loud; if it doesn't make sense, the agent won't either.
- **A local dev mode** (stdio + a fake auth provider) that runs against the real server logic.
- **Contract tests** that verify the JSON Schema of every tool's response, run on every commit.

### 10.8 Evolution

- **Quarterly review** of which tools agents actually call, removing the ones that aren't used.
- **Capability flags** so a beta tool can be exposed to a subset of clients before becoming default.
- **A deprecation policy** - tools live for a documented minimum, then are removed with notice.

---

## 11. References and Further Reading

### Official spec and roadmap

- [Model Context Protocol - Specification (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25)
- [Architecture overview - Model Context Protocol](https://modelcontextprotocol.io/docs/learn/architecture)
- [The 2026 MCP Roadmap - Model Context Protocol Blog](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)
- [Specification GitHub repository](https://github.com/modelcontextprotocol/modelcontextprotocol)
- [Security Best Practices - Model Context Protocol](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)
- [Understanding Authorization in MCP](https://modelcontextprotocol.io/docs/tutorials/security/authorization)
- [Logging spec](https://modelcontextprotocol.io/specification/draft/server/utilities/logging)

### Vendor and platform guides

- [What is Model Context Protocol (MCP)? - Google Cloud](https://cloud.google.com/discover/what-is-model-context-protocol)
- [Expose REST API as MCP server - Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/export-rest-mcp-server)
- [Stateful MCP client capabilities on Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/introducing-stateful-mcp-client-capabilities-on-amazon-bedrock-agentcore-runtime/)
- [MCP protocol contract - AWS Bedrock AgentCore docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp-protocol-contract.html)
- [Protecting against indirect prompt injection - Microsoft for Developers](https://developer.microsoft.com/blog/protecting-against-indirect-injection-attacks-mcp)
- [MCP security: authentication and authorization - Red Hat](https://www.redhat.com/en/blog/mcp-security-implementing-robust-authentication-and-authorization)

### Design and best practices

- [15 Best Practices for Building MCP Servers in Production - The New Stack](https://thenewstack.io/15-best-practices-for-building-mcp-servers-in-production/)
- [Top 5 MCP Server Best Practices - Docker](https://www.docker.com/blog/mcp-server-best-practices/)
- [MCP is Not the Problem, It's Your Server - Phil Schmid](https://www.philschmid.de/mcp-best-practices)
- [Stop Converting Your REST APIs to MCP - Mostly Harmless](https://jlowin.dev/blog/stop-converting-rest-apis-to-mcp)
- [Designing an MCP server from a REST API - WorkOS](https://workos.com/blog/designing-mcp-server-from-rest-api)
- [Understanding MCP features: Tools, Resources, Prompts, Sampling, Roots, Elicitation - WorkOS](https://workos.com/blog/mcp-features-guide)
- [From REST API to MCP Server - Stainless](https://www.stainless.com/mcp/from-rest-api-to-mcp-server)
- [Should you wrap MCP around your existing API? - Scalekit](https://www.scalekit.com/blog/wrap-mcp-around-existing-api)

### Comparisons and explainers

- [MCP vs. REST: What's the right way to connect AI agents to your API? - WorkOS](https://workos.com/blog/mcp-vs-rest)
- [MCP vs API: When to Use Each for AI Agent Integration in 2026 - Atlan](https://atlan.com/know/when-to-use-mcp-vs-api/)
- [A Deep Dive Into MCP and the Future of AI Tooling - a16z](https://a16z.com/a-deep-dive-into-mcp-and-the-future-of-ai-tooling/)
- [Why Model Context Protocol uses JSON-RPC - Daniel Avila](https://medium.com/@dan.avila7/why-model-context-protocol-uses-json-rpc-64d466112338)

### Security deep dives

- [MCP Security: The Complete Guide - Practical DevSecOps](https://www.practical-devsecops.com/mcp-security-guide/)
- [MCP Security Vulnerabilities: Prompt Injection & Tool Poisoning - Practical DevSecOps](https://www.practical-devsecops.com/mcp-security-vulnerabilities/)
- [New Prompt Injection Attack Vectors Through MCP Sampling - Palo Alto Unit 42](https://unit42.paloaltonetworks.com/model-context-protocol-attack-vectors/)
- [Top 10 MCP Security Risks - Prompt Security](https://prompt.security/blog/top-10-mcp-security-risks)
- [The New MCP Authorization Specification (OAuth 2.1, Resource Indicators)](https://dasroot.net/posts/2026/04/mcp-authorization-specification-oauth-2-1-resource-indicators/)

### Session management

- [Building Stateful MCP Servers: A Complete Guide (2026) - Fastio](https://fast.io/resources/building-stateful-mcp-servers/)
- [Managing Stateful MCP Server Sessions - CodeSignal](https://codesignal.com/learn/courses/developing-and-integrating-an-mcp-server-in-typescript/lessons/stateful-mcp-server-sessions)
- [MCP Roadmap 2026 - a2a-mcp.org](https://a2a-mcp.org/blog/mcp-2026-roadmap)

---

*This document is meant to be a living reference. The protocol is moving fast - re-check the official spec link before quoting any specific RFC requirement, and follow the roadmap link for what's landing next.*
