# Cloudflare Workers: Complete Code Examples

This document contains complete, production-ready code examples for common Cloudflare Workers patterns.

## Table of Contents

1. [Durable Objects: WebSocket with Hibernation API](#1-durable-objects-websocket-with-hibernation-api)
2. [Durable Objects: Alarm API](#2-durable-objects-alarm-api)
3. [Workers KV: Session Authentication](#3-workers-kv-session-authentication)
4. [Queues: Producer and Consumer](#4-queues-producer-and-consumer)
5. [Hyperdrive: PostgreSQL Connection](#5-hyperdrive-postgresql-connection)
6. [Workflows: Durable Execution](#6-workflows-durable-execution)
7. [Analytics Engine: Event Tracking](#7-analytics-engine-event-tracking)
8. [Browser Rendering: Puppeteer Automation](#8-browser-rendering-puppeteer-automation)
9. [Static Assets: SPA Hosting](#9-static-assets-spa-hosting)
10. [Agents: AI Agent with State Management](#10-agents-ai-agent-with-state-management)
11. [Workers AI: Structured JSON Outputs](#11-workers-ai-structured-json-outputs)

---

## 1. Durable Objects: WebSocket with Hibernation API

**Use Case:** Real-time communication, chat rooms, multiplayer games, live collaboration

### Code

```typescript
import { DurableObject } from "cloudflare:workers";

interface Env {
  WEBSOCKET_HIBERNATION_SERVER: DurableObjectNamespace;
}

// Durable Object
export class WebSocketHibernationServer extends DurableObject {
  async fetch(request) {
    // Creates two ends of a WebSocket connection.
    const webSocketPair = new WebSocketPair();
    const [client, server] = Object.values(webSocketPair);

    // Calling `acceptWebSocket()` informs the runtime that this WebSocket is to begin terminating
    // request within the Durable Object. It has the effect of "accepting" the connection,
    // and allowing the WebSocket to send and receive messages.
    // Unlike `ws.accept()`, `state.acceptWebSocket(ws)` informs the Workers Runtime that the WebSocket
    // is "hibernatable", so the runtime does not need to pin this Durable Object to memory while
    // the connection is open. During periods of inactivity, the Durable Object can be evicted
    // from memory, but the WebSocket connection will remain open. If at some later point the
    // WebSocket receives a message, the runtime will recreate the Durable Object
    // (run the `constructor`) and deliver the message to the appropriate handler.
    this.ctx.acceptWebSocket(server);

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }

  async webSocketMessage(ws: WebSocket, message: string | ArrayBuffer): Promise<void> {
    // Upon receiving a message from the client, reply with the same message,
    // but will prefix the message with "[Durable Object]: " and return the
    // total number of connections.
    ws.send(
      `[Durable Object] message: ${message}, connections: ${this.ctx.getWebSockets().length}`,
    );
  }

  async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean): Promise<void> {
    // If the client closes the connection, the runtime will invoke the webSocketClose() handler.
    ws.close(code, "Durable Object is closing WebSocket");
  }

  async webSocketError(ws: WebSocket, error: unknown): Promise<void> {
    console.error("WebSocket error:", error);
    ws.close(1011, "WebSocket error");
  }
}

export default {
  async fetch(request: Request, env: Env) {
    const id = env.WEBSOCKET_HIBERNATION_SERVER.idFromName("chat-room");
    const stub = env.WEBSOCKET_HIBERNATION_SERVER.get(id);
    return stub.fetch(request);
  }
}
```

### Configuration

```jsonc
{
  "name": "websocket-hibernation-server",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "durable_objects": {
    "bindings": [
      {
        "name": "WEBSOCKET_HIBERNATION_SERVER",
        "class_name": "WebSocketHibernationServer"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_classes": ["WebSocketHibernationServer"]
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Key Points

- Uses the WebSocket Hibernation API instead of the legacy WebSocket API
- Calls `this.ctx.acceptWebSocket(server)` to accept the WebSocket connection
- Has a `webSocketMessage()` handler that is invoked when a message is received from the client
- Has a `webSocketClose()` handler that is invoked when the WebSocket connection is closed
- Does NOT use the `server.addEventListener` API
- Don't over-use the "Hibernation" term in code or in bindings. It is an implementation detail.

---

## 2. Durable Objects: Alarm API

**Use Case:** Scheduled tasks, periodic cleanup, recurring notifications

### Code

```typescript
import { DurableObject } from "cloudflare:workers";

interface Env {
  ALARM_EXAMPLE: DurableObjectNamespace;
}

export default {
  async fetch(request, env) {
    let url = new URL(request.url);
    let userId = url.searchParams.get("userId") || crypto.randomUUID();
    const id = env.ALARM_EXAMPLE.idFromName(userId);
    return env.ALARM_EXAMPLE.get(id).fetch(request);
  },
};

const SECONDS = 1000;

export class AlarmExample extends DurableObject {
  constructor(ctx, env) {
    super(ctx, env);
    this.storage = ctx.storage;
  }

  async fetch(request) {
    // If there is no alarm currently set, set one for 10 seconds from now
    let currentAlarm = await this.storage.getAlarm();
    if (currentAlarm == null) {
      await this.storage.setAlarm(Date.now() + 10 * SECONDS);
    }
    return new Response("Alarm set");
  }

  async alarm() {
    // The alarm handler will be invoked whenever an alarm fires.
    // You can use this to do work, read from the Storage API, make HTTP calls
    // and set future alarms to run using this.storage.setAlarm() from within this handler.
    console.log("Alarm triggered!");

    // Set a new alarm for 10 seconds from now before exiting the handler
    await this.storage.setAlarm(Date.now() + 10 * SECONDS);
  }
}
```

### Configuration

```jsonc
{
  "name": "durable-object-alarm",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "durable_objects": {
    "bindings": [
      {
        "name": "ALARM_EXAMPLE",
        "class_name": "AlarmExample"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_classes": ["AlarmExample"]
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Key Points

- Uses the Durable Object Alarm API to trigger an alarm
- Has an `alarm()` handler that is invoked when the alarm is triggered
- Sets a new alarm for 10 seconds from now before exiting the handler

---

## 3. Workers KV: Session Authentication

**Use Case:** Session management, token authentication, user auth

### Code

```typescript
// src/index.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'

interface Env {
  AUTH_TOKENS: KVNamespace;
}

const app = new Hono<{ Bindings: Env }>()

// Add CORS middleware
app.use('*', cors())

app.get('/', async (c) => {
  try {
    // Get token from header or cookie
    const token = c.req.header('Authorization')?.slice(7) ||
      c.req.header('Cookie')?.match(/auth_token=([^;]+)/)?.[1];

    if (!token) {
      return c.json({
        authenticated: false,
        message: 'No authentication token provided'
      }, 403)
    }

    // Check token in KV
    const userData = await c.env.AUTH_TOKENS.get(token)

    if (!userData) {
      return c.json({
        authenticated: false,
        message: 'Invalid or expired token'
      }, 403)
    }

    return c.json({
      authenticated: true,
      message: 'Authentication successful',
      data: JSON.parse(userData)
    })
  } catch (error) {
    console.error('Authentication error:', error)
    return c.json({
      authenticated: false,
      message: 'Internal server error'
    }, 500)
  }
})

export default app
```

### Configuration

```jsonc
{
  "name": "auth-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "kv_namespaces": [
    {
      "binding": "AUTH_TOKENS",
      "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "preview_id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Dependencies

```bash
npm install hono
```

### Key Points

- Uses Hono as the router and middleware
- Uses Workers KV to store session data
- Uses the Authorization header or Cookie to get the token
- Checks the token in Workers KV
- Returns a 403 if the token is invalid or expired

---

## 4. Queues: Producer and Consumer

**Use Case:** Background processing, async tasks, batch operations

### Code

```typescript
// src/index.ts
interface Env {
  REQUEST_QUEUE: Queue;
  UPSTREAM_API_URL: string;
  UPSTREAM_API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env) {
    const info = {
      timestamp: new Date().toISOString(),
      method: request.method,
      url: request.url,
      headers: Object.fromEntries(request.headers),
    };
    await env.REQUEST_QUEUE.send(info);

    return Response.json({
      message: 'Request logged',
      requestId: crypto.randomUUID()
    });
  },

  async queue(batch: MessageBatch<any>, env: Env) {
    const requests = batch.messages.map(msg => msg.body);

    const response = await fetch(env.UPSTREAM_API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${env.UPSTREAM_API_KEY}`
      },
      body: JSON.stringify({
        timestamp: new Date().toISOString(),
        batchSize: requests.length,
        requests
      })
    });

    if (!response.ok) {
      throw new Error(`Upstream API error: ${response.status}`);
    }
  }
};
```

### Configuration

```jsonc
{
  "name": "request-logger-consumer",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "queues": {
    "producers": [
      {
        "name": "request-queue",
        "binding": "REQUEST_QUEUE"
      }
    ],
    "consumers": [
      {
        "name": "request-queue",
        "dead_letter_queue": "request-queue-dlq",
        "retry_delay": 300
      }
    ]
  },
  "vars": {
    "UPSTREAM_API_URL": "https://api.example.com/batch-logs"
  },
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Key Points

- Defines both a producer and consumer for the queue
- Uses a dead letter queue for failed messages
- Uses a retry delay of 300 seconds to delay the re-delivery of failed messages
- Shows how to batch requests to an upstream API

---

## 5. Hyperdrive: PostgreSQL Connection

**Use Case:** Connecting to external PostgreSQL databases with connection pooling

### Code

```typescript
// Postgres.js 3.4.5 or later is recommended
import postgres from "postgres";

export interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request, env, ctx): Promise<Response> {
    // Create a database client that connects to your database via Hyperdrive.
    const sql = postgres(env.HYPERDRIVE.connectionString);

    try {
      // Test query
      const results = await sql`SELECT * FROM pg_tables`;

      // Return result rows as JSON
      return Response.json(results);
    } catch (e) {
      console.error(e);
      return Response.json(
        { error: e instanceof Error ? e.message : e },
        { status: 500 },
      );
    }
  },
} satisfies ExportedHandler<Env>;
```

### Configuration

```jsonc
{
  "name": "hyperdrive-postgres",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<YOUR_DATABASE_ID>"
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Setup

```bash
# Install Postgres.js
npm install postgres

# Create a Hyperdrive configuration
npx wrangler hyperdrive create <YOUR_CONFIG_NAME> \
  --connection-string="postgres://user:password@HOSTNAME_OR_IP_ADDRESS:PORT/database_name"
```

### Key Points

- Installs and uses Postgres.js as the database client/driver
- Creates a Hyperdrive configuration using wrangler and the database connection string
- Uses the Hyperdrive connection string to connect to the database
- Calling `sql.end()` is optional, as Hyperdrive will handle the connection pooling

---

## 6. Workflows: Durable Execution

**Use Case:** Multi-step async tasks, retry logic, human-in-the-loop workflows

### Code

```typescript
import { WorkflowEntrypoint, WorkflowStep, WorkflowEvent } from 'cloudflare:workers';

type Env = {
  MY_WORKFLOW: Workflow;
};

type Params = {
  email: string;
  metadata: Record<string, string>;
};

export class MyWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    const files = await step.do('fetch files', async () => {
      return {
        files: [
          'doc_7392_rev3.pdf',
          'report_x29_final.pdf',
          'memo_2024_05_12.pdf',
        ],
      };
    });

    const apiResponse = await step.do('call API', async () => {
      let resp = await fetch('https://api.cloudflare.com/client/v4/ips');
      return await resp.json<any>();
    });

    await step.sleep('wait on something', '1 minute');

    await step.do(
      'make a call that could fail',
      {
        retries: {
          limit: 5,
          delay: '5 second',
          backoff: 'exponential',
        },
        timeout: '15 minutes',
      },
      async () => {
        if (Math.random() > 0.5) {
          throw new Error('API call failed');
        }
      },
    );
  }
}

export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    let url = new URL(req.url);

    // Get the status of an existing instance
    let id = url.searchParams.get('instanceId');
    if (id) {
      let instance = await env.MY_WORKFLOW.get(id);
      return Response.json({
        status: await instance.status(),
      });
    }

    const data = await req.json();

    // Spawn a new instance
    let instance = await env.MY_WORKFLOW.create({
      id: crypto.randomUUID(),
      params: data,
    });

    return Response.json({
      id: instance.id,
      details: await instance.status(),
    });
  },
};
```

### Configuration

```jsonc
{
  "name": "workflows-starter",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "workflows": [
    {
      "name": "workflows-starter",
      "binding": "MY_WORKFLOW",
      "class_name": "MyWorkflow"
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Key Points

- Defines a Workflow by extending the WorkflowEntrypoint class
- Defines a run method that is invoked when the Workflow is started
- Ensures that `await` is used before calling `step.do` or `step.sleep`
- Passes a payload (event) to the Workflow from a Worker
- Defines a payload type and uses TypeScript type arguments

---

## 7. Analytics Engine: Event Tracking

**Use Case:** Custom analytics, metrics, high-cardinality data

### Code

```typescript
interface Env {
  USER_EVENTS: AnalyticsEngineDataset;
}

export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    let url = new URL(req.url);
    let path = url.pathname;
    let userId = url.searchParams.get("userId");

    // Write a datapoint for this visit
    env.USER_EVENTS.writeDataPoint({
      doubles: [],  // numeric metrics
      blobs: [path],  // text labels
      indexes: [userId],  // grouping index
    });

    return Response.json({
      hello: "world",
    });
  },
};
```

### Configuration

```jsonc
{
  "name": "analytics-engine-example",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "analytics_engine_datasets": [
    {
      "binding": "USER_EVENTS",
      "dataset": "user_events"
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Querying Data

```bash
# Query via SQL API
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <API_TOKEN>" \
  --data "SELECT timestamp, blob1 AS path, index1 AS user_id FROM user_events WHERE timestamp > NOW() - INTERVAL '1' DAY"
```

### Key Points

- Binds an Analytics Engine dataset to the Worker
- Does NOT `await` calls to `writeDataPoint`, as it is non-blocking
- Defines an index as the key representing an app, customer, merchant or tenant
- Developers can use the GraphQL or SQL APIs to query data

---

## 8. Browser Rendering: Puppeteer Automation

**Use Case:** Web scraping, screenshots, headless browser automation

### Code

```typescript
import puppeteer from "@cloudflare/puppeteer";

interface Env {
  BROWSER: Fetcher;
}

export default {
  async fetch(request, env): Promise<Response> {
    const { searchParams } = new URL(request.url);
    let url = searchParams.get("url");

    if (url) {
      url = new URL(url).toString(); // normalize
      const browser = await puppeteer.launch(env.BROWSER);
      const page = await browser.newPage();
      await page.goto(url);

      // Parse the page content
      const content = await page.content();

      // Find text within the page content
      const text = await page.$eval("body", (el) => el.textContent);

      console.log(text);

      // Ensure we close the browser session
      await browser.close();

      return Response.json({
        bodyText: text,
      });
    } else {
      return Response.json({
        error: "Please add an ?url=https://example.com/ parameter"
      }, { status: 400 });
    }
  },
} satisfies ExportedHandler<Env>;
```

### Configuration

```jsonc
{
  "name": "browser-rendering-example",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "browser": [
    {
      "binding": "BROWSER"
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Dependencies

```bash
npm install @cloudflare/puppeteer --save-dev
```

### Key Points

- Configures a BROWSER binding
- Passes the binding to Puppeteer
- Uses the Puppeteer APIs to navigate to a URL and render the page
- Parses the DOM and returns context
- Correctly creates and closes the browser instance

---

## 9. Static Assets: SPA Hosting

**Use Case:** Hosting SPAs, frontend frameworks (React, Vue, Svelte)

### Code

```typescript
// src/index.ts
interface Env {
  ASSETS: Fetcher;
}

export default {
  fetch(request, env) {
    const url = new URL(request.url);

    if (url.pathname.startsWith("/api/")) {
      return Response.json({
        name: "Cloudflare",
      });
    }

    return env.ASSETS.fetch(request);
  },
} satisfies ExportedHandler<Env>;
```

### Configuration

```jsonc
{
  "name": "my-app",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "assets": {
    "directory": "./public/",
    "not_found_handling": "single-page-application",
    "binding": "ASSETS"
  },
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Key Points

- Configures an ASSETS binding
- Uses /public/ as the directory for build output
- The Worker handles API routes while serving static assets
- If the application is an SPA, HTTP 404 requests redirect to the SPA

---

## 10. Agents: AI Agent with State Management

**Use Case:** Building AI agents with state management, WebSockets, and scheduling

### Code

```typescript
// src/index.ts
import { Agent, AgentNamespace, Connection } from 'agents';
import { OpenAI } from "openai";

interface Env {
  AIAgent: AgentNamespace<Agent>;
  OPENAI_API_KEY: string;
}

export class AIAgent extends Agent<Env> {
  // Handle HTTP requests
  async onRequest(request) {
    const ai = new OpenAI({
      apiKey: this.env.OPENAI_API_KEY,
    });

    const response = await ai.chat.completions.create({
      model: "gpt-4",
      messages: [{ role: "user", content: await request.text() }],
    });

    return new Response(response.choices[0].message.content);
  }

  // Handle WebSockets
  async onConnect(connection: Connection) {
    await this.initiate(connection);
    connection.accept();
  }

  async onMessage(connection, message) {
    const understanding = await this.comprehend(message);
    await this.respond(connection, understanding);
  }

  // State management
  async evolve(newInsight) {
    this.setState({
      ...this.state,
      insights: [...(this.state.insights || []), newInsight],
      understanding: this.state.understanding + 1,
    });
  }

  onStateUpdate(state, source) {
    console.log("Understanding deepened:", {
      newState: state,
      origin: source,
    });
  }

  // Scheduling examples
  async scheduleExamples() {
    // Schedule a task to run in 10 seconds
    let task = await this.schedule(10, "someTask", { message: "hello" });

    // Schedule a task to run at a specific date
    await this.schedule(new Date("2025-01-01"), "someTask", {});

    // Schedule a task to run every 10 seconds (cron)
    let { id } = await this.schedule("*/10 * * * *", "someTask", { message: "hello" });

    // Cancel a scheduled task
    this.cancelSchedule(id);
  }

  // SQL database access
  async useSql() {
    // Use this.sql to interact with the Agent's embedded SQLite database
    const results = await this.sql.exec("SELECT * FROM data");
    return results;
  }
}

export default {
  async fetch(request: Request, env: Env) {
    const id = env.AIAgent.idFromName("my-agent");
    const agent = env.AIAgent.get(id);
    return agent.fetch(request);
  }
};
```

### Configuration

```jsonc
{
  "name": "ai-agent",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "durable_objects": {
    "bindings": [
      {
        "name": "AIAgent",
        "class_name": "AIAgent"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["AIAgent"]
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Dependencies

```bash
npm install agents openai
```

### Key Points

- Extends the Agent class with proper type parameters
- Uses `this.setState` for state management
- Uses `this.sql` for direct SQLite access
- Implements `onRequest`, `onConnect`, and `onMessage` handlers
- Must set `migrations[].new_sqlite_classes` to the Agent class name in wrangler.jsonc
- Use streaming responses from AI SDKs when appropriate

---

## 11. Workers AI: Structured JSON Outputs

**Use Case:** Getting structured JSON responses from AI models

### Code

```typescript
import { OpenAI } from "openai";

interface Env {
  AI: Ai;
}

export default {
  async fetch(request: Request, env: Env) {
    const ai = new OpenAI({
      apiKey: "empty",
      baseURL: "https://api.cloudflare.com/client/v4/accounts/{account_id}/ai/v1",
    });

    const response = await ai.chat.completions.create({
      model: "@cf/meta/llama-3-8b-instruct",
      messages: [
        {
          role: "system",
          content: "You are a helpful assistant that outputs JSON.",
        },
        {
          role: "user",
          content: "Extract the name and age from: John is 30 years old",
        },
      ],
      response_format: {
        type: "json_object",
      },
    });

    return Response.json(response.choices[0].message.content);
  },
} satisfies ExportedHandler<Env>;
```

### Configuration

```jsonc
{
  "name": "workers-ai-json",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],
  "ai": {
    "binding": "AI"
  },
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

### Dependencies

```bash
npm install openai
```

### Key Points

- Uses OpenAI SDK with Workers AI
- Sets `response_format: { type: "json_object" }` for structured outputs
- Works with compatible models that support JSON mode

---

## Summary

These examples demonstrate:

- **Durable Objects** for stateful coordination and WebSockets
- **Workers KV** for session management
- **Queues** for async processing
- **Hyperdrive** for external database connections
- **Workflows** for durable, multi-step execution
- **Analytics Engine** for custom metrics
- **Browser Rendering** for web automation
- **Static Assets** for frontend hosting
- **Agents** for AI-powered applications
- **Workers AI** for edge inference

All examples follow best practices:
- TypeScript with proper types
- ES modules format
- Complete wrangler.jsonc configurations
- Proper error handling
- Observability enabled
