# Cloudflare Workers: WebSockets & Agents

This document contains specialized guidelines for implementing WebSocket connections and AI Agents in Cloudflare Workers.

## WebSocket Guidelines

### Hibernatable WebSocket API (Preferred)

When providing WebSocket handling code within a Durable Object, **you SHALL use the Durable Objects WebSocket Hibernation API**.

**Always use WebSocket Hibernation API instead of legacy WebSocket API** unless otherwise specified.

### Core WebSocket Principles

1. **Use `this.ctx.acceptWebSocket(server)`** to accept the WebSocket connection
   - **DO NOT use** the `server.accept()` method (legacy approach)

2. **Define handler methods** for WebSocket events:
   - `async webSocketMessage()` - Invoked when a message is received from the client
   - `async webSocketClose()` - Invoked when the WebSocket connection is closed
   - `async webSocketError()` - Invoked when a WebSocket error occurs

3. **DO NOT use the `addEventListener` pattern** to handle WebSocket events inside a Durable Object
   - You MUST use the handler methods instead

4. **Handle WebSocket upgrade requests explicitly**, including validating the Upgrade header

5. **Don't over-use the "Hibernation" term** in code or in bindings
   - It is an implementation detail, not a user-facing concept

### WebSocket Handler Signatures

```typescript
class MyDurableObject extends DurableObject {
  // Accept WebSocket connection
  async fetch(request: Request) {
    const webSocketPair = new WebSocketPair();
    const [client, server] = Object.values(webSocketPair);

    // Use this.ctx.acceptWebSocket, NOT server.accept()
    this.ctx.acceptWebSocket(server);

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }

  // Handle incoming messages
  async webSocketMessage(
    ws: WebSocket,
    message: string | ArrayBuffer
  ): Promise<void> {
    // Process message and respond
    ws.send(`Received: ${message}`);
  }

  // Handle connection closure
  async webSocketClose(
    ws: WebSocket,
    code: number,
    reason: string,
    wasClean: boolean
  ): Promise<void> {
    // Cleanup logic
    ws.close(code, "Closing connection");
  }

  // Handle errors
  async webSocketError(
    ws: WebSocket,
    error: unknown
  ): Promise<void> {
    console.error("WebSocket error:", error);
    ws.close(1011, "WebSocket error");
  }
}
```

### WebSocket Hibernation Benefits

The Hibernation API enables:
- **Memory efficiency**: Durable Object can be evicted from memory during inactivity
- **Connection persistence**: WebSocket connections remain open even when the DO is not in memory
- **Automatic recreation**: Runtime recreates the Durable Object when a message arrives
- **Scalability**: Supports many concurrent connections without pinning objects to memory

### WebSocket Connection Management

```typescript
// Get all active WebSocket connections
const connections = this.ctx.getWebSockets();
console.log(`Active connections: ${connections.length}`);

// Broadcast to all connections
for (const ws of connections) {
  ws.send("Broadcast message");
}

// Close specific connection
ws.close(1000, "Normal closure");
```

### WebSocket Best Practices

1. **Validate upgrade requests**
   ```typescript
   const upgradeHeader = request.headers.get("Upgrade");
   if (upgradeHeader !== "websocket") {
     return new Response("Expected WebSocket", { status: 426 });
   }
   ```

2. **Handle disconnections gracefully**
   - Clean up state in `webSocketClose()`
   - Track connection state

3. **Implement heartbeat/ping-pong**
   - Keep connections alive
   - Detect dead connections

4. **Use proper close codes**
   - 1000: Normal closure
   - 1011: Internal error
   - 1008: Policy violation

5. **Broadcast efficiently**
   - Use `this.ctx.getWebSockets()` to get all connections
   - Filter connections as needed before broadcasting

### Complete WebSocket Example Pattern

```typescript
export class ChatRoom extends DurableObject {
  private sessions: Map<WebSocket, { name: string; joined: number }>;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.sessions = new Map();
  }

  async fetch(request: Request) {
    // Validate WebSocket upgrade
    if (request.headers.get("Upgrade") !== "websocket") {
      return new Response("Expected WebSocket", { status: 426 });
    }

    const pair = new WebSocketPair();
    const [client, server] = Object.values(pair);

    // Accept using Hibernation API
    this.ctx.acceptWebSocket(server);

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }

  async webSocketMessage(ws: WebSocket, message: string | ArrayBuffer) {
    // Parse and validate message
    const data = JSON.parse(message.toString());

    // Handle different message types
    if (data.type === "join") {
      this.sessions.set(ws, {
        name: data.name,
        joined: Date.now()
      });
      this.broadcast({ type: "user_joined", name: data.name });
    } else if (data.type === "message") {
      const session = this.sessions.get(ws);
      this.broadcast({
        type: "chat",
        from: session?.name,
        text: data.text
      });
    }
  }

  async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) {
    const session = this.sessions.get(ws);
    this.sessions.delete(ws);

    if (session) {
      this.broadcast({ type: "user_left", name: session.name });
    }
  }

  async webSocketError(ws: WebSocket, error: unknown) {
    console.error("WebSocket error:", error);
    ws.close(1011, "Internal error");
  }

  private broadcast(message: any) {
    const data = JSON.stringify(message);
    for (const ws of this.ctx.getWebSockets()) {
      try {
        ws.send(data);
      } catch (err) {
        console.error("Broadcast error:", err);
      }
    }
  }
}
```

---

## Agents Framework

### Overview

**Strongly prefer the `agents` framework** when building AI Agents.

Agents extend AI capabilities with:
- **State management** via `this.setState()`
- **SQL database access** through `this.sql` API
- **Scheduled task execution** via `this.schedule()`
- **WebSocket and HTTP request handling**
- **React integration** through `useAgent` hook

### Core Agent Principles

1. **Extend the Agent class** with proper type parameters
   ```typescript
   class AIAgent extends Agent<Env, MyState> { }
   ```

2. **Use streaming responses** from AI SDKs
   - OpenAI SDK
   - Workers AI bindings
   - Anthropic client SDK

3. **Follow user direction** on AI provider choice
   - User requests Claude → Use Anthropic SDK
   - User requests OpenAI → Use OpenAI SDK
   - Default → Workers AI

4. **State management**
   - Prefer `this.setState` API for state management
   - Use `this.sql` for direct SQLite access when beneficial

5. **Client interface**
   - Use `useAgent` React hook from `agents/react` library as the preferred approach

6. **Configuration**
   - Include valid Durable Object bindings in wrangler.jsonc
   - MUST set `migrations[].new_sqlite_classes` to the Agent class name

### Agent Class Structure

```typescript
import { Agent, Connection } from 'agents';

interface Env {
  AIAgent: AgentNamespace<Agent>;
  OPENAI_API_KEY: string;
}

interface MyState {
  insights: string[];
  understanding: number;
}

export class AIAgent extends Agent<Env, MyState> {
  // HTTP request handler
  async onRequest(request: Request) {
    // Handle HTTP requests
    return new Response("OK");
  }

  // WebSocket connection handler
  async onConnect(connection: Connection) {
    // Initialize connection
    connection.accept();
  }

  // WebSocket message handler
  async onMessage(connection: Connection, message: any) {
    // Handle incoming messages
  }

  // State update handler
  onStateUpdate(state: MyState, source: string) {
    console.log("State updated:", state, "from:", source);
  }
}
```

### State Management

#### Using setState()

```typescript
async evolve(newInsight: string) {
  this.setState({
    ...this.state,
    insights: [...(this.state.insights || []), newInsight],
    understanding: this.state.understanding + 1,
  });
}
```

**Key Points:**
- Automatically syncs state across connections
- Triggers `onStateUpdate()` handler
- Persists to embedded SQLite database

#### Using SQL Directly

```typescript
async useSql() {
  // Direct SQLite access for complex queries
  const results = await this.sql.exec(`
    SELECT * FROM data
    WHERE timestamp > datetime('now', '-1 day')
  `);
  return results;
}
```

**When to use SQL:**
- Complex queries with joins
- Performance-critical operations
- Direct database control needed

### Scheduling Tasks

Agents can schedule tasks to run in the future using `this.schedule(when, callback, data)`.

```typescript
async scheduleExamples() {
  // Schedule a task to run in 10 seconds
  let task = await this.schedule(10, "processTask", { message: "hello" });

  // Schedule a task to run at a specific date
  await this.schedule(new Date("2025-01-01"), "yearlyReport", {});

  // Schedule a task to run every 10 seconds (cron)
  let { id } = await this.schedule("*/10 * * * *", "heartbeat", {});

  // Schedule a task to run every Monday at midnight
  await this.schedule("0 0 * * 1", "weeklyDigest", {});

  // Cancel a scheduled task
  this.cancelSchedule(id);

  // Get a specific schedule by ID
  const schedule = await this.getSchedule(id);
}

// The scheduled task handler
async processTask(data: { message: string }) {
  console.log("Scheduled task running:", data.message);
  // Can do anything: make requests, query databases, send emails, read/write state
}
```

**Scheduling Parameters:**
- `when`: Delay in seconds, Date object, or cron string
- `callback`: Function name to call (as string)
- `data`: Object of data to pass to the function

**Cron Format:**
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
* * * * *
```

### AI Integration Patterns

#### Using Workers AI

```typescript
async processWithWorkersAI(prompt: string) {
  const response = await this.env.AI.run(
    '@cf/meta/llama-3-8b-instruct',
    {
      messages: [
        { role: 'system', content: 'You are a helpful assistant.' },
        { role: 'user', content: prompt }
      ],
      stream: true
    }
  );

  return response;
}
```

#### Using OpenAI SDK

```typescript
import { OpenAI } from 'openai';

async processWithOpenAI(prompt: string) {
  const ai = new OpenAI({
    apiKey: this.env.OPENAI_API_KEY,
  });

  const stream = await ai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: prompt }],
    stream: true
  });

  return stream;
}
```

#### Using Anthropic SDK

```typescript
import Anthropic from '@anthropic-ai/sdk';

async processWithClaude(prompt: string) {
  const anthropic = new Anthropic({
    apiKey: this.env.ANTHROPIC_API_KEY,
  });

  const stream = await anthropic.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 1024,
    messages: [{ role: "user", content: prompt }],
    stream: true
  });

  return stream;
}
```

### React Integration

#### Using the useAgent Hook

```typescript
// Client-side React component
import { useAgent } from 'agents/react';

function ChatInterface() {
  const { state, setState, sendMessage } = useAgent({
    agentUrl: 'https://my-agent.workers.dev',
    agentId: 'my-agent-instance'
  });

  const handleSend = (message: string) => {
    sendMessage({ type: 'chat', text: message });
  };

  return (
    <div>
      <div>Understanding level: {state.understanding}</div>
      <input onSubmit={handleSend} />
    </div>
  );
}
```

**Key Features:**
- Automatic WebSocket connection management
- Real-time state synchronization
- Optimistic updates

### Agent Configuration

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
      "new_sqlite_classes": ["AIAgent"]  // REQUIRED for Agents
    }
  ],
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  }
}
```

**Critical Configuration:**
- Must set `new_sqlite_classes` (not `new_classes`) for Agents
- Agent class name must match exactly

### Agent Best Practices

1. **Type your Agent properly**
   ```typescript
   class AIAgent extends Agent<Env, StateType> { }
   ```

2. **Use streaming for AI responses**
   - Better user experience
   - Lower latency
   - Progressive rendering

3. **Handle errors gracefully**
   - Wrap AI calls in try-catch
   - Provide fallback responses
   - Log errors for debugging

4. **Manage state efficiently**
   - Use `setState` for simple state
   - Use `this.sql` for complex queries
   - Don't over-sync state

5. **Schedule cleanup tasks**
   - Use cron for periodic maintenance
   - Clean up old data
   - Optimize database

6. **Test thoroughly**
   - Test state synchronization
   - Test scheduled tasks
   - Test WebSocket connections

### Complete Agent Example

```typescript
import { Agent, Connection } from 'agents';
import { OpenAI } from 'openai';

interface Env {
  MyAgent: AgentNamespace<Agent>;
  OPENAI_API_KEY: string;
}

interface ConversationState {
  messages: Array<{ role: string; content: string }>;
  userId: string;
  createdAt: number;
}

export class MyAgent extends Agent<Env, ConversationState> {
  async onRequest(request: Request) {
    const prompt = await request.text();
    const response = await this.processPrompt(prompt);
    return new Response(response);
  }

  async onConnect(connection: Connection) {
    // Initialize state for new connections
    if (!this.state.messages) {
      this.setState({
        messages: [],
        userId: crypto.randomUUID(),
        createdAt: Date.now()
      });
    }
    connection.accept();
  }

  async onMessage(connection: Connection, message: any) {
    const { text } = JSON.parse(message);

    // Add user message to history
    const updatedMessages = [
      ...this.state.messages,
      { role: 'user', content: text }
    ];

    this.setState({
      ...this.state,
      messages: updatedMessages
    });

    // Process with AI
    const response = await this.processPrompt(text);

    // Add assistant response
    this.setState({
      ...this.state,
      messages: [
        ...this.state.messages,
        { role: 'assistant', content: response }
      ]
    });

    // Send response
    connection.send(JSON.stringify({ type: 'response', text: response }));
  }

  async processPrompt(prompt: string): Promise<string> {
    const ai = new OpenAI({ apiKey: this.env.OPENAI_API_KEY });

    const completion = await ai.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "system", content: "You are a helpful assistant." },
        ...this.state.messages,
        { role: "user", content: prompt }
      ]
    });

    return completion.choices[0].message.content || "";
  }

  onStateUpdate(state: ConversationState, source: string) {
    console.log(`State updated from ${source}:`, {
      messageCount: state.messages.length,
      userId: state.userId
    });
  }

  // Schedule daily cleanup
  async init() {
    await this.schedule("0 0 * * *", "dailyCleanup", {});
  }

  async dailyCleanup() {
    // Keep only last 100 messages
    if (this.state.messages.length > 100) {
      this.setState({
        ...this.state,
        messages: this.state.messages.slice(-100)
      });
    }
  }
}
```

## Summary

### WebSocket Guidelines
- ✅ Use `this.ctx.acceptWebSocket(server)`
- ✅ Implement `webSocketMessage()`, `webSocketClose()`, `webSocketError()` handlers
- ✅ Don't use `addEventListener` pattern in Durable Objects
- ✅ Handle upgrades explicitly
- ✅ Avoid over-using "hibernation" terminology

### Agent Guidelines
- ✅ Extend `Agent<Env, State>` with proper types
- ✅ Use `this.setState()` for state management
- ✅ Use `this.sql` for complex database operations
- ✅ Use streaming responses from AI SDKs
- ✅ Set `new_sqlite_classes` in migrations
- ✅ Use `useAgent` hook for React integration
- ✅ Schedule tasks with `this.schedule()`
- ✅ Follow user's AI provider preference

Both WebSockets and Agents leverage Durable Objects to provide strongly consistent, stateful coordination at the edge.
