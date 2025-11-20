# Cloudflare Workers: Service Integrations

This document provides detailed guidance on when and how to integrate various Cloudflare services with Workers.

## Service Integration Overview

When data storage or additional capabilities are needed, integrate with appropriate Cloudflare services. Always include necessary bindings in both code and wrangler.jsonc, and add appropriate environment variable definitions.

## Storage Services

### Workers KV

**Use KV for:**
- Key-value storage
- Configuration data
- User profiles and sessions
- A/B testing flags
- Cache-like data that can tolerate eventual consistency

**Binding Example:**

```jsonc
{
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "preview_id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  ]
}
```

**Code Usage:**

```typescript
interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request, env) {
    // Write
    await env.MY_KV.put('key', 'value');

    // Read
    const value = await env.MY_KV.get('key');

    // Delete
    await env.MY_KV.delete('key');

    return new Response(value);
  }
}
```

**Key Characteristics:**
- Eventually consistent
- Global distribution
- Low-latency reads
- Ideal for configuration and session data

### Durable Objects

**Use Durable Objects for:**
- Strongly consistent state management
- Per-object storage (SQLite)
- Multiplayer coordination
- Real-time collaboration
- WebSocket connection handling
- Agent use-cases
- Rate limiting and counters

**Binding Example:**

```jsonc
{
  "durable_objects": {
    "bindings": [
      {
        "name": "COUNTER",
        "class_name": "Counter"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_classes": ["Counter"]
    }
  ]
}
```

**Code Usage:**

```typescript
export class Counter extends DurableObject {
  async fetch(request) {
    let value = (await this.ctx.storage.get("value")) || 0;
    value++;
    await this.ctx.storage.put("value", value);
    return new Response(value.toString());
  }
}
```

**Key Characteristics:**
- Strong consistency per object
- Single-threaded execution per instance
- Built-in SQLite storage
- WebSocket support
- Global coordination point

### D1 (SQL Database)

**Use D1 for:**
- Relational data
- SQL queries and joins
- Structured data with relationships
- Transactional consistency requirements

**Binding Example:**

```jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database",
      "database_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  ]
}
```

**Code Usage:**

```typescript
interface Env {
  DB: D1Database;
}

export default {
  async fetch(request, env) {
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE id = ?'
    ).bind(1).all();

    return Response.json(results);
  }
}
```

**Key Characteristics:**
- SQL interface (SQLite dialect)
- Regional deployment
- Read replicas available
- ACID transactions

### R2 (Object Storage)

**Use R2 for:**
- Object storage
- Storing structured data (JSON, etc.)
- AI assets and models
- Image and media assets
- User-facing uploads
- Backups and archives

**Binding Example:**

```jsonc
{
  "r2_buckets": [
    {
      "binding": "MY_BUCKET",
      "bucket_name": "my-bucket"
    }
  ]
}
```

**Code Usage:**

```typescript
interface Env {
  MY_BUCKET: R2Bucket;
}

export default {
  async fetch(request, env) {
    // Upload
    await env.MY_BUCKET.put('file.txt', 'Hello, World!');

    // Download
    const object = await env.MY_BUCKET.get('file.txt');
    const text = await object?.text();

    // Delete
    await env.MY_BUCKET.delete('file.txt');

    return new Response(text);
  }
}
```

**Key Characteristics:**
- S3-compatible API
- No egress fees
- Automatic global distribution
- Large object support

## Database Connectivity

### Hyperdrive

**Use Hyperdrive for:**
- Connecting to existing PostgreSQL databases
- Connection pooling to external databases
- Query caching
- Reducing connection overhead

**Binding Example:**

```jsonc
{
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<YOUR_DATABASE_ID>"
    }
  ]
}
```

**Code Usage:**

```typescript
import postgres from "postgres";

interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request, env) {
    const sql = postgres(env.HYPERDRIVE.connectionString);
    const results = await sql`SELECT * FROM users`;
    return Response.json(results);
  }
}
```

**Key Characteristics:**
- Connection pooling
- Query caching
- Supports PostgreSQL and MySQL
- Reduces database connection overhead

## Async Processing

### Queues

**Use Queues for:**
- Asynchronous processing
- Background tasks
- Batch operations
- Retry logic
- Decoupling producers from consumers

**Binding Example:**

```jsonc
{
  "queues": {
    "producers": [
      {
        "name": "my-queue",
        "binding": "MY_QUEUE"
      }
    ],
    "consumers": [
      {
        "name": "my-queue",
        "dead_letter_queue": "my-queue-dlq",
        "retry_delay": 300
      }
    ]
  }
}
```

**Code Usage:**

```typescript
interface Env {
  MY_QUEUE: Queue;
}

export default {
  // Producer
  async fetch(request, env) {
    await env.MY_QUEUE.send({ task: 'process', data: 'value' });
    return new Response('Queued');
  },

  // Consumer
  async queue(batch: MessageBatch, env) {
    for (const message of batch.messages) {
      console.log('Processing:', message.body);
      // Process message
    }
  }
}
```

**Key Characteristics:**
- At-least-once delivery
- Automatic retries
- Dead letter queues
- Batch processing support

## AI & Intelligence

### Workers AI

**Use Workers AI for:**
- AI inference at the edge
- LLM requests (default AI API)
- Image generation
- Text embeddings
- Translation

**Binding Example:**

```jsonc
{
  "ai": {
    "binding": "AI"
  }
}
```

**Code Usage:**

```typescript
interface Env {
  AI: Ai;
}

export default {
  async fetch(request, env) {
    const response = await env.AI.run(
      '@cf/meta/llama-2-7b-chat-int8',
      {
        messages: [
          { role: 'user', content: 'Hello!' }
        ]
      }
    );
    return Response.json(response);
  }
}
```

**External AI SDKs:**
- If user requests **Claude**, use the official **Anthropic SDK**
- If user requests **OpenAI**, use the official **OpenAI SDK**
- Use streaming responses when appropriate

### Vectorize

**Use Vectorize for:**
- Storing embeddings
- Vector search and similarity matching
- Often used in combination with Workers AI
- RAG (Retrieval Augmented Generation) systems

**Binding Example:**

```jsonc
{
  "vectorize": [
    {
      "binding": "VECTORIZE",
      "index_name": "my-index"
    }
  ]
}
```

**Key Characteristics:**
- Vector similarity search
- Integrates with Workers AI for embeddings
- Supports multiple distance metrics

### Browser Rendering

**Use Browser Rendering for:**
- Remote browser capabilities
- Web scraping
- Searching the web
- Screenshot generation
- Puppeteer APIs
- Headless automation

**Binding Example:**

```jsonc
{
  "browser": [
    {
      "binding": "BROWSER"
    }
  ]
}
```

**Code Usage:**

```typescript
import puppeteer from "@cloudflare/puppeteer";

interface Env {
  BROWSER: Fetcher;
}

export default {
  async fetch(request, env) {
    const browser = await puppeteer.launch(env.BROWSER);
    const page = await browser.newPage();
    await page.goto('https://example.com');
    const content = await page.content();
    await browser.close();
    return new Response(content);
  }
}
```

**Dependencies:**

```bash
npm install @cloudflare/puppeteer --save-dev
```

## Analytics & Observability

### Workers Analytics Engine

**Use Analytics Engine for:**
- Tracking user events
- Billing metrics
- High-cardinality analytics
- Custom metrics and dimensions

**Binding Example:**

```jsonc
{
  "analytics_engine_datasets": [
    {
      "binding": "ANALYTICS",
      "dataset": "my_dataset"
    }
  ]
}
```

**Code Usage:**

```typescript
interface Env {
  ANALYTICS: AnalyticsEngineDataset;
}

export default {
  async fetch(request, env) {
    // Write datapoint (non-blocking, don't await)
    env.ANALYTICS.writeDataPoint({
      doubles: [123.45],  // numeric metrics
      blobs: ['page_view'],  // text labels
      indexes: ['user_123']  // grouping key
    });

    return new Response('Logged');
  }
}
```

**Key Characteristics:**
- Do NOT `await` calls to `writeDataPoint` (non-blocking)
- Define an index as the key representing app/customer/tenant
- Query via GraphQL or SQL APIs
- High-cardinality support

### Querying Analytics Engine

```bash
# Query via SQL API
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <API_TOKEN>" \
  --data "SELECT * FROM my_dataset WHERE timestamp > NOW() - INTERVAL '1' DAY"
```

## Workflows

**Use Workflows for:**
- Durable execution
- Multi-step async tasks
- Human-in-the-loop workflows
- Retry logic with backoff
- Long-running processes

**Binding Example:**

```jsonc
{
  "workflows": [
    {
      "name": "my-workflow",
      "binding": "MY_WORKFLOW",
      "class_name": "MyWorkflow"
    }
  ]
}
```

**Key Characteristics:**
- Durable execution across steps
- Built-in retry logic
- State persistence
- Sleep/delay support

## Static Assets

**Use Static Assets for:**
- Hosting frontend applications
- Serving static files
- Building Workers with frontend frameworks (React, Vue, Svelte)
- Single Page Applications (SPAs)

**Binding Example:**

```jsonc
{
  "assets": {
    "directory": "./public/",
    "not_found_handling": "single-page-application",
    "binding": "ASSETS"
  }
}
```

**Code Usage:**

```typescript
interface Env {
  ASSETS: Fetcher;
}

export default {
  fetch(request, env) {
    const url = new URL(request.url);

    // API routes
    if (url.pathname.startsWith("/api/")) {
      return Response.json({ message: "API" });
    }

    // Serve static assets
    return env.ASSETS.fetch(request);
  }
}
```

**Key Characteristics:**
- SPA support (404 handling)
- Framework integration
- API and static assets in one Worker

## Platform Features

### Service Bindings

**Use for:** Worker-to-Worker RPC communication without public internet

**Binding Example:**

```jsonc
{
  "services": [
    {
      "binding": "BACKEND_SERVICE",
      "service": "backend-worker",
      "environment": "production"
    }
  ]
}
```

**Code Usage (Frontend Worker):**

```typescript
interface Env {
  BACKEND_SERVICE: Fetcher;
}

export default {
  async fetch(request: Request, env: Env) {
    // Call backend Worker directly (no public internet)
    const backendRequest = new Request('https://fake-host/api/data', {
      method: 'POST',
      body: JSON.stringify({ userId: '123' })
    });

    const response = await env.BACKEND_SERVICE.fetch(backendRequest);
    const data = await response.json();

    return Response.json(data);
  }
}
```

**Code Usage (Backend Worker):**

```typescript
export default {
  async fetch(request: Request, env: Env) {
    // This Worker can be called via Service Binding
    // AND can access database bindings
    const data = await env.DB.prepare('SELECT * FROM users').all();
    return Response.json(data);
  }
}
```

**Key Characteristics:**
- No egress to public internet
- Lower latency than HTTP
- Type-safe RPC
- Automatic load balancing
- Can pass request context

**When to use:**
- Microservices architecture
- Splitting frontend (edge) and backend (near DB)
- Internal APIs between Workers
- Reducing latency

**Pattern:**
```
User → Frontend Worker (edge)
         ↓ Service Binding (internal)
       Backend Worker (near database)
         ↓ Binding
       Database (D1/Hyperdrive)
```

---

### Smart Placement

**Use for:** Automatically placing workloads near backend infrastructure

**Configuration:**

```jsonc
{
  "name": "backend-worker",
  "main": "src/index.ts",
  "placement": {
    "mode": "smart"
  }
}
```

**How it works:**
- **Frontend Workers**: Run at edge (near users)
- **Backend Workers**: Automatically run near data sources
- **Cloudflare chooses**: Optimal location based on your bindings

**Code Usage:**

```typescript
// Backend Worker - will run near D1/Hyperdrive automatically
export default {
  async fetch(request: Request, env: Env) {
    // This Worker runs near your database
    const results = await env.DB.prepare('SELECT * FROM products').all();
    return Response.json(results);
  }
}
```

**When to use:**
- Full-stack applications
- Workers that primarily access databases
- Optimizing database query latency
- Multi-region deployments

**Pattern:**
```
Frontend Worker (mode: default, runs at edge)
  ↓ Service Binding
Backend Worker (mode: smart, runs near DB)
  ↓ Binding
Database (D1/Hyperdrive in specific region)
```

**Key Benefits:**
- Automatic optimization (no manual configuration)
- Reduced database latency
- Better performance for data-heavy Workers
- Works with D1 and Hyperdrive

---

### Secrets Store

**Use for:** Centralized secret management with RBAC

**Status:** Beta

**Configuration:**

```jsonc
{
  "secrets_store": {
    "binding": "SECRETS"
  }
}
```

**Code Usage:**

```typescript
interface Env {
  SECRETS: Fetcher;
}

export default {
  async fetch(request: Request, env: Env) {
    // Fetch secret from Secrets Store
    const apiKeyResponse = await env.SECRETS.fetch(
      'https://secrets/API_KEY'
    );
    const apiKey = await apiKeyResponse.text();

    // Use secret in external API call
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    });

    return response;
  }
}
```

**Key Characteristics:**
- Centralized secret management
- Role-based access control (RBAC)
- Audit logging
- Secret rotation support
- Share secrets across multiple Workers

**Migration from Environment Variables:**

```typescript
// OLD: Wrangler secrets (per-Worker)
interface Env {
  API_KEY: string; // Set via: wrangler secret put API_KEY
}

// NEW: Secrets Store (centralized)
interface Env {
  SECRETS: Fetcher; // Access any secret from central store
}
```

**When to use:**
- Enterprise secret management
- Multiple Workers sharing secrets
- Compliance requirements (audit logging)
- Secret rotation workflows
- RBAC for secret access

---

### Workers VPC

**Use for:** Connect Workers to private networks across clouds

**Status:** Open Beta

**Configuration:**

```jsonc
{
  "vpc": {
    "binding": "MY_VPC",
    "vpc_id": "vpc-xxxxx"
  }
}
```

**Code Usage:**

```typescript
interface Env {
  MY_VPC: Fetcher;
}

export default {
  async fetch(request: Request, env: Env) {
    // Call private resource in VPC
    const response = await env.MY_VPC.fetch(
      'http://internal-api.private:8080/data'
    );

    const data = await response.json();
    return Response.json(data);
  }
}
```

**Key Characteristics:**
- Private network connectivity
- No public internet exposure
- Multi-cloud support (AWS, GCP, Azure)
- Secure tunneling to VPCs
- Works with on-premise networks

**When to use:**
- Accessing private cloud resources
- Enterprise integration
- Hybrid cloud architectures
- Connecting to internal APIs
- Private database access

**Pattern:**
```
Worker (public edge)
  ↓ VPC Binding (secure tunnel)
Private Cloud Resources (AWS/GCP/Azure VPC)
  ↓
Internal APIs, Databases, Services
```

**Security:**
- Encrypted tunnels
- No public IPs required
- Firewall rules apply
- Zero Trust integration available

---

## Service Selection Guide

### Quick Decision Tree

**Need storage?**
- Key-value, eventually consistent → **KV**
- Relational data, SQL queries → **D1**
- Files, media, large objects → **R2**
- Strong consistency, coordination → **Durable Objects**
- Existing PostgreSQL database → **Hyperdrive**

**Need async processing?**
- Background tasks, retries → **Queues**
- Durable multi-step workflows → **Workflows**

**Need AI capabilities?**
- Edge inference → **Workers AI**
- External LLMs (Claude/OpenAI) → **Use official SDKs**
- Vector search → **Vectorize**
- Web scraping, automation → **Browser Rendering**

**Need observability?**
- Custom analytics, metrics → **Analytics Engine**
- Standard logging → **Built-in observability**

**Need frontend hosting?**
- Static sites, SPAs → **Static Assets**

## Integration Best Practices

1. **Only include bindings you use** in wrangler.jsonc
2. **Use TypeScript interfaces** to type your Env
3. **Follow least privilege principle** for binding permissions
4. **Don't hardcode IDs** - use environment variables when appropriate
5. **Consider consistency requirements** when choosing storage
6. **Use official SDKs** when available
7. **Test locally** with `wrangler dev`

## Summary

Cloudflare provides a rich ecosystem of services that compose well together:
- **Storage**: KV, D1, R2, Durable Objects
- **Database connectivity**: Hyperdrive
- **Async**: Queues, Workflows
- **AI**: Workers AI, Vectorize, Browser Rendering
- **Analytics**: Analytics Engine
- **Frontend**: Static Assets

Choose the right primitive for your use case, and they'll work seamlessly together through Workers bindings.
