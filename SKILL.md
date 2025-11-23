---
name: cloudflare-advisor
description: Comprehensive guide for Cloudflare development covering Workers, Pages, D1, KV, Durable Objects, R2, Hyperdrive, AI Gateway, Browser Rendering, Agents, and all other Cloudflare developer platform services. Use when the user needs help with Cloudflare products including deployment, configuration, API usage, choosing the right primitives, composing solutions, migrating from traditional servers, or understanding how Cloudflare products work together. Covers mental models for edge computing, solution architecture patterns, and best practices.
---

# Cloudflare Developer Skill

This skill helps developers build applications on the Cloudflare platform by teaching mental models, composition patterns, and best practices for edge computing.

## How to Use This Skill

This skill is organized in three layers:

1. **This file (SKILL.md)**: Core mental models, thinking patterns, and composition principles
2. **references/**: Product catalogs, solution patterns, code generation guides, and examples
3. **references/cloudflare-docs.md**: Complete documentation index (auto-updated llms.txt)

### For Conceptual Questions
When helping users understand Cloudflare:
- Start here (SKILL.md) for mental models and patterns
- Read `references/primitives-catalog.md` for product details
- Read `references/composition-patterns.md` for solution architectures
- Read `references/migration-guides.md` for migration guidance
- Read `references/cloudflare-docs.md` for complete documentation links
- Use `web_fetch` to retrieve specific documentation pages when needed

### For Code Generation
When generating Cloudflare Workers code:
- **Always start** with `references/workers-best-practices.md` for standards
- Read `references/workers-integrations.md` for service-specific integration details
- Read `references/workers-websockets-agents.md` for WebSocket or Agent implementations
- Read `references/workers-examples.md` for complete working examples
- Follow the Decision Tree below to load the right context

## Mental Model: From Servers to Workers

Cloudflare represents a fundamental shift from traditional server-based architecture to edge computing. Understanding this shift is critical.

### The Core Difference

**Traditional Servers:**
- Long-running processes in data centers
- Stateful applications with local filesystems
- Vertical scaling (bigger machines)
- Single-region deployment
- Always-on infrastructure

**Cloudflare Workers:**
- Request-scoped execution at the edge
- Stateless by default (or Durable Objects for state)
- Automatic horizontal scaling
- Global by default (300+ locations)
- Pay-per-request model

### Key Paradigm Shifts

#### 1. No Filesystem → Use Bindings

```javascript
// ❌ Server thinking
const fs = require('fs');
const config = fs.readFileSync('./config.json');

// ✅ Worker thinking
const config = await env.CONFIG_BUCKET.get('config.json'); // R2
// or
const config = await env.CONFIG_KV.get('settings'); // KV
```

#### 2. No In-Memory State → Use Durable Objects

```javascript
// ❌ Server thinking
let counter = 0;
app.get('/count', () => ++counter);

// ✅ Worker thinking
const id = env.COUNTER.idFromName('global');
const stub = env.COUNTER.get(id);
await stub.increment();
```

#### 3. Bindings Over Network Calls

```javascript
// ❌ Server thinking
const users = await fetch('http://db-service:3000/api/users');

// ✅ Worker thinking  
const users = await env.DB.prepare('SELECT * FROM users').all();
```

#### 4. Edge-First Architecture

```javascript
// ❌ Server thinking: Origin-centric
// User → Load Balancer → Origin → Database

// ✅ Worker thinking: Edge-centric
export default {
  async fetch(request, env) {
    // Run at edge, near user
    const cached = await caches.default.match(request);
    if (cached) return cached;
    
    // Only hit origin when necessary
    const response = await fetchFromOrigin();
    return response;
  }
}
```

#### 5. WebSockets in Durable Objects

```javascript
// ❌ Server thinking: WebSocket in main app
// app.js handles WebSocket connections

// ✅ Worker thinking: WebSocket in Durable Object
export default {
  async fetch(request, env) {
    const id = env.CHAT_ROOM.idFromName('room-123');
    const room = env.CHAT_ROOM.get(id);
    return room.fetch(request); // DO handles WebSocket
  }
}
```

#### 6. Keep Workers Lightweight

```javascript
// ❌ Server thinking
import massiveLibrary from 'huge-package';
const model = await loadMLModel(); // 500MB

// ✅ Worker thinking
export default {
  async fetch(request, env) {
    // Keep Workers fast and small
    // Defer heavy work to queues or Durable Objects
    if (needsHeavyProcessing) {
      await env.QUEUE.send({ task: 'process' });
      return new Response('Processing async');
    }
  }
}
```

## Cloudflare Primitives: Building Blocks

Cloudflare provides primitives that compose into complete solutions. Think of them as LEGO blocks, not monolithic services.

**Categories**: Compute (Workers, Durable Objects, Pages), Storage (KV, D1, R2), Network (Hyperdrive, Queues), AI (Workers AI, Agents, Vectorize), Platform (Service Bindings, Smart Placement), Security (WAF, Turnstile, Secrets Store), Observability (Analytics Engine, Workers Logs).

See `references/primitives-catalog.md` for complete product details.

### Choosing Storage: Decision Framework

**Use KV when:**
- Eventually consistent is acceptable
- Global read performance matters
- Simple key-value lookups
- Examples: sessions, feature flags, configuration

**Use D1 when:**
- Need SQL queries and relations
- Consistency is required
- Regional deployment is acceptable
- Examples: user accounts, product catalogs, transactions

**Use R2 when:**
- Storing files, media, or large objects
- Need S3-compatible API
- Examples: user uploads, backups, static assets

**Use Durable Object Storage when:**
- State tied to specific coordination logic
- Need transactional consistency per object
- Examples: chat room state, game session state, counters

### Composition Rules

1. **Workers are the glue** - Everything connects through Workers
2. **Pages Functions ARE Workers** - Just with file-based routing
3. **Durable Objects for coordination** - WebSockets, rate limiting, stateful logic
4. **Hyperdrive for external DBs** - Connection pooling to PostgreSQL/MySQL
5. **Queues for async work** - Background processing, retries

## Solution Patterns: How Primitives Compose

### Pattern 1: Content Site + Dynamic Features

```
Pages (static frontend)
├─→ Pages Functions (API routes)
├─→ D1 (structured data)
└─→ R2 (user uploads)
```

**Use when:** Blogs, documentation, marketing sites with forms/comments

### Pattern 2: Real-Time Collaboration

```
Worker (HTTP entry)
├─→ Durable Object (coordination)
│   ├─→ WebSocket connections
│   └─→ SQLite storage (state)
└─→ KV (user profiles)
```

**Use when:** Chat apps, multiplayer games, collaborative editing

**Key points:**
- Durable Objects handle WebSocket state
- One DO instance per room/session
- KV for user metadata

### Pattern 3: Full-Stack Application

```
Pages (React/Vue/Svelte frontend)
├─→ Pages Functions (API)
│   ├─→ D1 (primary database)
│   ├─→ R2 (file storage)
│   └─→ KV (cache, sessions)
└─→ Workers (background jobs via Queues)
```

**Use when:** SaaS apps, dashboards, internal tools

### Pattern 4: AI Application

```
Worker (orchestrator)
├─→ AI Gateway (model proxy) → OpenAI/Anthropic/etc
├─→ Vectorize (semantic search)
├─→ D1 (conversation history)
└─→ KV (user preferences)
```

**Use when:** Chatbots, RAG systems, AI assistants

### Pattern 5: External Database Integration

```
Worker (API) → Hyperdrive (connection pool) → PostgreSQL/MySQL (external)
```

**Use when:** Connecting existing databases, gradual migration

### Pattern 6: Background Processing Pipeline

```
Worker (trigger) → Queue → Worker (consumer)
  ├─→ Browser Rendering (screenshots)
  ├─→ R2 (store results)
  └─→ D1 (status tracking)
```

**Use when:** ETL, web scraping, batch jobs, report generation

### Pattern 7: Media Processing

```
Worker (upload handler)
  ├─→ R2 (store original)
  ├─→ Images (transform/serve)
  └─→ D1 (metadata)
```

**Use when:** User-generated content, media platforms

## Best Practices Summary

1. **Design for the Edge** - Cache aggressively, minimize backend calls, use global primitives
2. **Handle Cold Starts** - Keep Workers small (<1MB), avoid expensive initialization
3. **Error Handling** - Wrap in try-catch, log errors, return graceful failures
4. **Use ctx.waitUntil()** - Don't block responses for async work (logging, analytics)
5. **Implement Caching** - Use Cache API, don't await cache writes

See `references/workers-best-practices.md` for detailed code examples and security patterns.

## Evaluating New Cloudflare Products

When Cloudflare releases a new primitive, evaluate it using this framework:

1. **What category?** (Compute/Storage/Network/Intelligence/Observability)
2. **What problem does it solve?**
3. **How does it bind to Workers?** (Check for `env.BINDING_NAME` syntax)
4. **What are the limits?** (Always documented in docs)
5. **How does it compose?** (What other primitives does it work with?)
6. **Is there a REST API?** (For programmatic access)
7. **What's the pricing model?**

Every Cloudflare product follows this pattern:
- Docs at `developers.cloudflare.com/[product]`
- Getting started guide
- Workers binding API
- REST API reference
- Limits and pricing
- Examples and tutorials

To add a new product to this skill:
1. Update `references/primitives-catalog.md` with product details
2. Add composition patterns to `references/composition-patterns.md`
3. Update `references/cloudflare-docs.md` by re-fetching llms.txt
4. Only update this file if mental models need refinement

## When to Read References

### Mental Models & Strategy
- **Product details** → Read `references/primitives-catalog.md`
- **Solution architectures** → Read `references/composition-patterns.md`
- **Server migration** → Read `references/migration-guides.md`
- **Full documentation** → Read `references/cloudflare-docs.md` (then use web_fetch for specific pages)

### Code Generation & Implementation
- **Code standards and best practices** → Read `references/workers-best-practices.md`
- **Service integration details** → Read `references/workers-integrations.md`
- **Complete working examples** → Read `references/workers-examples.md`
- **WebSocket or Agent implementation** → Read `references/workers-websockets-agents.md`

## Decision Tree: Loading the Right Context

Use this decision tree to determine which reference files to load based on the user's request:

### 1. User asks a question about Cloudflare concepts
**Load:** Mental models and catalogs
```
- Read SKILL.md (already loaded)
- Read `references/primitives-catalog.md` for product details
- Read `references/composition-patterns.md` for architecture patterns
```

### 2. User asks "how do I migrate from X to Cloudflare?"
**Load:** Migration guides
```
- Read `references/migration-guides.md`
- Read `references/composition-patterns.md` for solution patterns
```

### 3. User asks you to generate code or build something
**Load:** Code generation references based on what they're building

#### Step 1: Load best practices first
```
- Read `references/workers-best-practices.md` (ALWAYS for code generation)
```

#### Step 2: Determine what services they need
```
If mentions KV, D1, R2, Queues, AI, etc.:
  → Read `references/workers-integrations.md`

If mentions WebSockets, real-time, chat, multiplayer:
  → Read `references/workers-websockets-agents.md`

If mentions AI Agent, state management, scheduling:
  → Read `references/workers-websockets-agents.md`
```

#### Step 3: Find similar examples (if helpful)
```
If you need a complete example pattern:
  → Read `references/workers-examples.md`
  → Find the most similar example to their use case
```

### 4. User asks about a specific service (KV, D1, R2, etc.)
**Load:** Service integration details
```
- Read `references/workers-integrations.md`
- Optionally read `references/workers-examples.md` for examples
```

### 5. User asks about WebSockets or Agents specifically
**Load:** Specialized protocols
```
- Read `references/workers-websockets-agents.md`
- Read `references/workers-examples.md` (examples #1, #2, #10)
```

### 6. User asks "show me an example of X"
**Load:** Examples directly
```
- Read `references/workers-examples.md`
- Search for the relevant example
```

### 7. User asks about security (WAF, Turnstile, rate limiting, secrets)
**Load:** Security integration patterns
```
- Read `references/security-patterns.md`
- For secrets: Also read `references/workers-integrations.md` (Secrets Store section)
```

### 8. User asks about wrangler commands or CLI
**Load:** Wrangler reference
```
- Read `references/wrangler-commands.md`
```

### 9. User mentions microservices, Service Bindings, or Smart Placement
**Load:** Platform features
```
- Read `references/workers-integrations.md` (Platform Features section)
- For architecture: Also read `references/composition-patterns.md`
```

## Code Generation Workflow

When generating Cloudflare Workers code, follow this workflow:

1. **Load best practices** (always)
   ```
   Read `references/workers-best-practices.md`
   ```

2. **Identify required services** from user request
   - Storage needed? → Which type? (KV, D1, R2, DO)
   - Real-time features? → WebSockets + Durable Objects
   - AI features? → Workers AI, Agents, or external SDKs
   - Async work? → Queues or Workflows
   - External DB? → Hyperdrive

3. **Load service-specific details** for identified services
   ```
   Read `references/workers-integrations.md`
   ```

4. **Check for specialized patterns**
   ```
   If WebSocket or Agent:
     Read `references/workers-websockets-agents.md`
   ```

5. **Find similar example** (optional but helpful)
   ```
   Read `references/workers-examples.md`
   Locate example closest to user's use case
   ```

6. **Generate code** following:
   - TypeScript by default
   - ES modules format
   - Complete wrangler.jsonc
   - Proper error handling
   - Security best practices
   - Observability enabled

## Quick Reference Chart

| User Request Contains... | Load These Files |
|-------------------------|------------------|
| "how does X work", "what is Y" | `primitives-catalog.md`, `composition-patterns.md` |
| "migrate from", "move from" | `migration-guides.md`, `composition-patterns.md` |
| "build", "create", "implement" | `workers-best-practices.md`, `workers-integrations.md` |
| "WebSocket", "real-time", "chat" | `workers-websockets-agents.md` |
| "Agent", "AI agent", "state management" | `workers-websockets-agents.md` |
| "security", "WAF", "Turnstile", "rate limit" | `security-patterns.md` |
| "Service Binding", "microservices", "Smart Placement" | `workers-integrations.md` (Platform Features) |
| "wrangler", "CLI", "deploy", "commands" | `wrangler-commands.md` |
| "KV", "D1", "R2", "Queue", specific service | `workers-integrations.md` |
| "show me an example" | `workers-examples.md` |
| Mentions specific Cloudflare product | `cloudflare-docs.md` → web_fetch specific page |

## Common Gotchas

1. **KV is eventually consistent** - Don't use for data that must be immediately consistent
2. **Workers have CPU time limits** - Offload heavy work to Queues or Durable Objects
3. **Durable Objects are single-threaded** - One instance handles requests serially
4. **Pages vs Workers** - Pages for static+API, Workers for pure compute
5. **D1 is regional** - Read replicas help but writes are single-region
6. **WebSockets require Durable Objects** - Can't handle WebSockets in regular Workers

## Quick Command Reference

See `references/wrangler-commands.md` for complete CLI reference. Key commands:
- `wrangler init` - Create Worker | `wrangler deploy` - Deploy | `wrangler dev` - Local dev
- `wrangler d1 create/execute/migrations` - D1 commands
- `wrangler kv namespace create` - KV | `wrangler r2 bucket create` - R2

## Summary

Cloudflare development is about:

1. **Thinking in primitives** - Not monolithic services, but composable building blocks
2. **Edge-first architecture** - Run code near users, minimize origin calls
3. **Bindings over network** - Direct connections between services
4. **Stateless by default** - Use Durable Objects when state is needed
5. **Global by default** - Automatic worldwide distribution

The mental shift from servers to Workers is the most important part. Once you understand the paradigm, the rest follows naturally.
