# Chapter 9: Extensibility via Protocols

You ship your agent. It reads files, writes code, runs tests. Users are happy—for about a week.

Then the requests start.

"Can it query our Jira board?" "We need it to talk to our internal docs system." "Can it work with our custom deployment pipeline?" "We have this internal tool nobody outside our company has heard of..."

You could add Jira support. And the internal docs API. And the deployment pipeline. But there's always another tool. Another API. Another internal system that only exists at one company. You can't keep up. Nobody can.

This is the extensibility problem, and if you don't solve it, your agent becomes either bloated or inadequate. Bloated if you try to support everything. Inadequate if you don't.

The solution isn't to work harder. It's to stop trying to predict what users need and let them bring their own capabilities.

## Why Hardcoding Fails

The hardcoded approach has four structural flaws:

**Scope explosion.** Every organization has its own tools, APIs, and internal systems. If you tried to support even a fraction of them, your agent would balloon with integrations that 99% of users never touch. Your codebase accumulates one-off connectors that most users never touch.

**The permission problem.** Adding Jira integration means shipping code that handles Jira credentials—for every user, even those who've never heard of Jira. Users rightly question why they should trust an agent with integration code for services they don't use.

**Velocity mismatch.** External services change. Slack updates its API. GitHub adds new features. If every capability lives in your codebase, you're perpetually chasing changes you don't control. Your roadmap becomes a reaction to other companies' release schedules.

**The trust barrier.** Somewhere, a brilliant engineer wrote the perfect integration for their company's internal tool. How do they share it with colleagues? Under the hardcoded model: they submit code to you, you review it, you ship it. That cycle takes months. The integration goes unused.

## The Protocol Alternative

The answer predates AI entirely. It's the same pattern that made VS Code extensible, that let web browsers run any website, that allowed LSP to transform code editors.

Instead of implementing every tool, you define a protocol—a contract specifying how tools announce themselves, how they're called, and how they return results. Your agent doesn't know anything about Jira. It knows how to speak the protocol.

```
┌─────────────┐                        ┌─────────────────┐
│             │       Protocol         │                 │
│    Agent    │ ←────────────────────→ │   Tool Server   │
│             │                        │                 │
└─────────────┘                        └─────────────────┘

The protocol defines:
  • How tools announce their capabilities
  • How the agent invokes a tool
  • How results return
  • How errors surface
  • How authentication works
```

The Language Server Protocol (LSP) proved this works at scale. Before LSP, every editor implemented language support from scratch. Want TypeScript in Vim? Someone wrote TypeScript analysis for Vim. Want it in Emacs? Write it again. Sublime Text? Again. Now there's one TypeScript language server, and any editor with an LSP client gets TypeScript support without reimplementing the analysis.

The Model Context Protocol (MCP) applies this pattern to AI agents. An MCP server exposes tools, resources, or prompts. An agent speaking MCP can use any server, regardless of who wrote it. The ecosystem compounds: every new tool benefits every compatible agent.

<Callout type="insight">
Protocol-based extension inverts the burden. Instead of you implementing everything, tool authors implement their domain expertise. You provide the platform; they provide the capabilities.
</Callout>

## Transports: How Messages Travel

A protocol defines *what* gets exchanged. The transport defines *how*. Different situations call for different transports.

### stdio: The Local Workhorse

For tools running on the same machine, standard input/output is hard to beat. The agent spawns the tool as a subprocess and communicates through stdin/stdout—the same pipes connecting shell commands.

```
┌─────────────┐         spawn          ┌─────────────────┐
│             │ ──────────────────────→│                 │
│    Agent    │        stdin           │   Local Tool    │
│   Process   │ ──────────────────────→│    Server       │
│             │        stdout          │   (subprocess)  │
│             │ ←──────────────────────│                 │
└─────────────┘                        └─────────────────┘
```

stdio is elegant in its simplicity: no network configuration, no ports, no firewall rules. The tool runs with the user's permissions, accessing local files and credentials naturally. When the agent exits, the subprocess exits—cleanup happens automatically.

The tradeoff is locality. stdio can't reach cloud services or shared infrastructure. For that, you need a network transport.

### HTTP: Stateless and Scalable

For remote tools, HTTP is the natural fit. Each tool call is a request; each result is a response. Familiar territory for web developers.

```
┌─────────────┐      HTTP POST /call     ┌─────────────────┐
│             │ ────────────────────────→│                 │
│    Agent    │                          │   Remote Tool   │
│             │      JSON Response       │     Server      │
│             │ ←────────────────────────│                 │
└─────────────┘                          └─────────────────┘
```

HTTP's statelessness is a feature: load balancers distribute requests automatically, and you deploy tool servers anywhere—cloud functions, containers, edge networks. Standard infrastructure handles TLS, rate limiting, and logging.

The limitation is the request-response model. Every call pays connection overhead, and streaming partial results requires workarounds.

### SSE: One-Way Streaming

Server-Sent Events extend HTTP for streaming. The client opens a connection; the server pushes events over time.

```
┌─────────────┐     GET (EventStream)    ┌─────────────────┐
│             │ ────────────────────────→│                 │
│    Agent    │     event: data          │   Tool Server   │
│             │ ←────────────────────────│                 │
│             │     event: data          │                 │
│             │ ←────────────────────────│                 │
│             │     event: done          │                 │
│             │ ←────────────────────────│                 │
└─────────────┘                          └─────────────────┘
```

SSE suits tools that produce incremental results. A database query streams rows as they arrive. A search tool returns matches progressively. The agent processes early results while later ones are still coming.

The constraint: SSE is unidirectional. The server streams to the client, but the client can't send mid-stream. For true bidirectional communication, you need WebSocket.

### WebSocket: Full Duplex

WebSocket maintains a persistent, bidirectional channel. Either side sends messages at any time.

```
┌─────────────┐      WebSocket Upgrade       ┌─────────────────┐
│             │ ────────────────────────────→│                 │
│    Agent    │                              │   Tool Server   │
│             │ ←──────── messages ────────→ │                 │
│             │                              │                 │
└─────────────┘    (persistent connection)   └─────────────────┘
```

WebSocket suits interactive tools with back-and-forth communication—debugging sessions, collaborative editing, real-time notifications. The persistent connection eliminates per-call latency.

The cost is complexity. WebSocket connections are stateful, requiring servers to track connection state. Load balancing needs sticky sessions. Dropped connections need reconnection logic.

### Transport Agnosticism

The protocol stays the same regardless of transport. A tool describes its capabilities identically whether reached via stdio or HTTP. The agent's tool-calling logic doesn't change based on how messages travel.

When implementing protocol support, design your transport layer as a pluggable abstraction. Core protocol handling—parsing, dispatching, error handling—should be transport-agnostic. Transport adapters handle delivery mechanics.

## Discovery: Learning What's Available

With built-in tools, you know the capabilities at compile time. Extensibility requires discovery at runtime—the agent learns what's available when it connects.

When an agent connects to a tool server, the server advertises what it offers:

```
Agent connects to server
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Server Capabilities Response                           │
├─────────────────────────────────────────────────────────┤
│  tools: [                                               │
│    {                                                    │
│      name: "query_database",                            │
│      description: "Execute SQL queries against the DB", │
│      inputSchema: { ... JSON Schema ... }               │
│    },                                                   │
│    {                                                    │
│      name: "list_tables",                               │
│      description: "List all tables in the database",    │
│      inputSchema: { ... }                               │
│    }                                                    │
│  ],                                                     │
│  resources: [                                           │
│    { uri: "db://schema", description: "DB schema" }     │
│  ],                                                     │
│  prompts: [                                             │
│    { name: "sql_expert", description: "SQL guidance" }  │
│  ]                                                      │
└─────────────────────────────────────────────────────────┘
```

The agent presents these capabilities to the language model, which decides when to use them.

Discovery isn't one-time. Servers can notify agents when capabilities change—a new tool added, an old one deprecated. The agent updates without restarting. Tools can even be ephemeral: available only during business hours, or only when a backing service is healthy.

Beyond tools, servers expose two other capability types:

**Resources** are data that enriches context. A documentation server exposes markdown files. A database server exposes schema information. A project tracker exposes issue lists. The agent pulls these into its context window—background knowledge without explicit tool calls.

**Prompts** are templates that shift agent behavior. A SQL server might provide a `sql_expert` prompt encoding best practices for queries. Users invoke prompts to specialize the agent for domain-specific tasks.

## Naming and Identity

With a single agent and a handful of tools, naming is trivial. Extensibility introduces collision risk: two servers might both offer a "search" tool. Which one gets called?

The solution is namespacing. A common pattern uses hierarchical names with double underscores:

```
{provider}__{server}__{tool}

Examples:
  github-mcp-server__github__search_code
  database__postgres__query
  company__internal-api__get_users
```

This convention delivers uniqueness (no collisions across hundreds of servers), parseability (agents extract server ownership), and permission granularity (allow one server's tools while restricting another).

Aliasing provides convenience. Users configure shortcuts: `search` maps to `github-mcp-server__github__search_code` in their environment. The canonical namespaced name remains authoritative; the alias saves keystrokes.

When tools evolve, deprecation paths matter. A server can advertise both old and new names during transition, marking the old one deprecated. Agents warn users while maintaining backward compatibility.

## Authentication Patterns

Tools often access protected resources on users' behalf. Extension authentication has unique challenges: the agent shouldn't hold permanent credentials, different tools need different permissions, and long-running sessions encounter token expiration.

### API Keys

The simplest pattern. User provides a key; the server uses it for requests.

```yaml
server: database
api_key: ${DB_API_KEY}   # From environment
```

API keys work for personal tools and internal services. They're easy to understand and implement. The downsides: keys are often long-lived (security exposure) and typically grant full access (no granularity).

### OAuth2

For third-party services, OAuth2 is the standard. Users explicitly grant tools permission to access specific resources.

```
┌─────────────┐      ┌─────────────────┐      ┌─────────────────┐
│    User     │      │   Tool Server   │      │  OAuth Provider │
└──────┬──────┘      └────────┬────────┘      └────────┬────────┘
       │                      │                        │
       │  "Connect GitHub"    │                        │
       │─────────────────────→│                        │
       │                      │                        │
       │      Redirect to authorization page           │
       │←─────────────────────────────────────────────│
       │                      │                        │
       │    User reviews scopes, grants permission     │
       │──────────────────────────────────────────────→│
       │                      │                        │
       │                      │   Authorization code   │
       │                      │←───────────────────────│
       │                      │                        │
       │                      │   Exchange for tokens  │
       │                      │───────────────────────→│
       │                      │                        │
       │                      │   Access + Refresh     │
       │                      │←───────────────────────│
       │                      │                        │
       │  "Connected!"        │                        │
       │←─────────────────────│                        │
```

Scoped access matters for security. When connecting GitHub, users see exactly what permissions the tool requests—maybe read access to public repos, not write access to everything. Informed consent.

### Token Refresh

OAuth access tokens typically expire within an hour. Long-running agents need refresh handling.

```
┌─────────────────┐                    ┌─────────────────┐
│   Tool Server   │                    │  OAuth Provider │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │  API call with access_token          │
         │─────────────────────────────────────→│
         │                                      │
         │  401 Unauthorized (expired)          │
         │←─────────────────────────────────────│
         │                                      │
         │  Refresh token request               │
         │─────────────────────────────────────→│
         │                                      │
         │  New access_token                    │
         │←─────────────────────────────────────│
         │                                      │
         │  Retry original request              │
         │─────────────────────────────────────→│
         │                                      │
         │  Success                             │
         │←─────────────────────────────────────│
```

Good tool servers handle refresh transparently. The agent doesn't know a token expired—the server refreshes and retries. Only if refresh fails (user revoked access) does the error reach the agent.

<Callout type="warning">
Cross-service authentication chains—where a tool server accesses another service on the agent's behalf—create trust chains that compound complexity. Each link needs appropriate scoping. Users should understand what access they're granting when enabling a tool that bridges to external services.
</Callout>

## Plugin Architecture

Raw protocol support enables extensibility, but users need a higher-level abstraction: plugins. A plugin bundles a tool server with metadata for installation, configuration, and lifecycle management.

### The Plugin Manifest

A manifest declares everything needed to run a tool server:

```yaml
name: github-integration
version: 2.1.0
author: octocat
description: GitHub integration for code search, PR management, and more

server:
  transport: stdio
  command: npx
  args: [-y, "@github/mcp-server"]
  
config:
  - name: GITHUB_TOKEN
    description: Personal access token
    required: true
    secret: true
  - name: DEFAULT_ORG
    description: Default organization for queries
    required: false

permissions:
  - network: api.github.com
  - read: ~/.gitconfig

capabilities:
  tools: true
  resources: true
  prompts: false
```

The manifest enables automated plugin management:

| Operation | What Happens |
|-----------|--------------|
| Install | Download package, verify signatures, store config |
| Configure | Present UI for user options |
| Start | Launch server with correct command and args |
| Update | Check versions, handle migrations |
| Remove | Shutdown cleanly, delete config, clear cache |

### Sandboxing

Plugins run code you didn't write. Security requires limiting what they can do.

**Network sandboxing** restricts which hosts a plugin can contact. A GitHub plugin needs `api.github.com`, but shouldn't exfiltrate data to arbitrary servers. The manifest declares required hosts; the runtime enforces the boundary.

**Filesystem sandboxing** limits file access. A plugin might need certain config files, but shouldn't read SSH keys or browser cookies. Declared permissions, enforced at runtime.

**Process sandboxing** isolates plugins from each other and the agent. A crashing plugin doesn't take down the agent. Container technologies, seccomp filters, or platform sandboxes provide isolation.

The tension is always security versus functionality. Too restrictive, and plugins can't do useful work. Too permissive, and you've created a security hole. The manifest-based model makes the tradeoff explicit: users see what a plugin needs and decide whether to trust it.

## Error Handling in Extensions

Extensions introduce failure modes that don't exist with built-in tools. Robust error handling is essential.

### Server Failures

A tool server might fail to start, crash mid-session, or become unresponsive. The agent should:

1. **Detect quickly** — Don't hang indefinitely waiting for a dead server
2. **Degrade gracefully** — Remove the server's tools from available capabilities
3. **Inform the model** — Let it adjust strategy knowing a capability is unavailable
4. **Attempt recovery** — Retry with exponential backoff

```
Agent attempts github__search_code
         │
         ▼
Server not responding
         │
         ▼
Return to language model:
  "Tool github__search_code temporarily unavailable.
   GitHub server not responding.
   Consider alternative approaches or retry later."
         │
         ▼
Model adapts (different tool, asks user, changes approach)
```

Transparency matters. The model needs to know a capability is down so it can plan accordingly. Silent failures or misleading errors produce confused behavior.

### Authentication Failures

Token expiration is common in long sessions. Good handling distinguishes auth issues from other errors:

```
Tool call returns 401 Unauthorized
         │
         ▼
Refreshable token?
    │           │
   Yes          No
    │           │
    ▼           ▼
Refresh      Surface auth error:
and retry    "Re-authentication required"
```

For refreshable tokens, renewal happens invisibly. For non-refreshable credentials, the error surfaces with clear guidance: "Your GitHub token has expired. Please reconnect your account."

### Tool Errors

Sometimes the tool runs but returns an error: query fails, file doesn't exist, API returns unexpected response. These errors return to the agent, not crash the server.

```json
{
  "error": {
    "code": "QUERY_FAILED",
    "message": "Table 'users' does not exist",
    "details": { ... }
  }
}
```

Language models handle tool errors better than you'd expect. They might realize they used the wrong table name and retry. Ask the user for clarification. Try a different approach. Your job is to report the error faithfully, not attempt recovery on their behalf.

### Version Mismatch

Protocols evolve. A server might use features the agent doesn't support. Version negotiation during connection prevents cryptic failures:

```
Agent:  { "protocolVersion": "2024.1" }
Server: { "protocolVersion": "2024.1", "capabilities": [...] }
```

When versions are incompatible, clear messaging helps: "This tool requires protocol version 2024.2, but your agent supports 2024.1. Please update your agent or use an older version of this tool."

## The Platform Mindset

Extensibility transforms your agent from a fixed capability set into a platform. Users adapt it to their unique needs without waiting for you to implement every possible integration.

The architecture we've explored—protocols defining contracts, transports delivering messages, discovery advertising capabilities, authentication protecting resources, manifests packaging tools, error handling ensuring resilience—provides the foundation.

If you're building an agent, prioritize protocol support early. Retrofitting extensibility is painful. Start with stdio for local tools (easiest to implement and debug), then add HTTP transports when you need remote capabilities.

If you're building tools for agents, follow established protocols. Your tool becomes instantly usable by any compatible agent—reaching users you'd never reach with agent-specific integrations.

The ecosystem effects compound. Every new tool server benefits every compatible agent. Every new agent benefits from every existing tool. This is how VS Code accumulated thousands of extensions, how browsers support billions of websites, and how agentic systems will scale beyond what any single team could build.

---

*Next chapter: How agents build trust through transparency—streaming responses, progress reporting, and responsive interfaces for long-running tasks.*
