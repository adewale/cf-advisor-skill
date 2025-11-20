# Cloudflare Composition Patterns

Detailed solution architectures showing how to combine Cloudflare primitives into complete applications.

## Pattern 1: Content Site with Dynamic Features

### Architecture
```
Pages (Static Frontend - React/Vue/Astro)
├─→ Pages Functions (API Routes at /api/*)
│   ├─→ D1 (Comments, user data)
│   ├─→ R2 (User-uploaded media)
│   └─→ KV (Session storage, feature flags)
└─→ Direct serving of static assets
```

### Use Cases
- Blogs with comment systems
- Documentation sites with user feedback
- Marketing sites with contact forms
- Portfolio sites with dynamic content

### Implementation

**Project Structure:**
```
my-site/
├── public/               # Static assets
├── src/
│   ├── pages/           # Frontend pages
│   └── components/      # React/Vue components
├── functions/           # Pages Functions (API)
│   ├── api/
│   │   ├── comments.ts  # CRUD for comments
│   │   ├── upload.ts    # Handle file uploads
│   │   └── auth.ts      # Authentication
│   └── _middleware.ts   # Global middleware
├── wrangler.toml
└── package.json
```

**wrangler.toml:**
```toml
name = "my-content-site"
pages_build_output_dir = "dist"

[[d1_databases]]
binding = "DB"
database_name = "site-db"
database_id = "..."

[[r2_buckets]]
binding = "UPLOADS"
bucket_name = "user-uploads"

[[kv_namespaces]]
binding = "SESSIONS"
id = "..."
```

**Example API Function (functions/api/comments.ts):**
```typescript
interface Env {
  DB: D1Database;
  UPLOADS: R2Bucket;
  SESSIONS: KVNamespace;
}

export const onRequestPost: PagesFunction<Env> = async ({ request, env }) => {
  const { content, postId } = await request.json();
  
  // Validate session
  const sessionId = request.headers.get('X-Session-ID');
  const user = await env.SESSIONS.get(sessionId);
  if (!user) return new Response('Unauthorized', { status: 401 });
  
  // Insert comment
  await env.DB.prepare(
    'INSERT INTO comments (post_id, user_id, content) VALUES (?, ?, ?)'
  ).bind(postId, JSON.parse(user).id, content).run();
  
  return new Response(JSON.stringify({ success: true }));
};

export const onRequestGet: PagesFunction<Env> = async ({ request, env }) => {
  const url = new URL(request.url);
  const postId = url.searchParams.get('postId');
  
  const { results } = await env.DB.prepare(
    'SELECT * FROM comments WHERE post_id = ? ORDER BY created_at DESC'
  ).bind(postId).all();
  
  return new Response(JSON.stringify(results));
};
```

### Key Patterns
1. **Static + Dynamic**: Static frontend, dynamic API routes
2. **File-based routing**: `/functions/api/comments.ts` → `/api/comments`
3. **Binding access**: Bindings available as `env` parameter
4. **Session management**: Use KV for session storage

---

## Pattern 2: Real-Time Collaboration Application

### Architecture
```
Worker (HTTP Entry Point)
└─→ Durable Object (Per-room coordination)
    ├─→ WebSocket Connections (multiple clients)
    ├─→ SQLite Storage (room state, history)
    └─→ KV (User profiles, permissions)
```

### Use Cases
- Chat applications
- Collaborative editing (docs, whiteboards)
- Multiplayer games
- Live dashboards

### Implementation

**Worker Entry Point:**
```typescript
interface Env {
  ROOMS: DurableObjectNamespace;
  USER_PROFILES: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const roomId = url.searchParams.get('room');
    
    // Get or create room
    const id = env.ROOMS.idFromName(roomId);
    const room = env.ROOMS.get(id);
    
    // Forward request to Durable Object
    return room.fetch(request);
  }
};
```

**Durable Object (Room Coordinator):**
```typescript
export class ChatRoom {
  state: DurableObjectState;
  env: Env;
  sessions: Set<WebSocket>;
  
  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.env = env;
    this.sessions = new Set();
  }
  
  async fetch(request: Request): Promise<Response> {
    // Handle WebSocket upgrade
    if (request.headers.get('Upgrade') === 'websocket') {
      const { 0: client, 1: server } = new WebSocketPair();
      
      await this.handleSession(server);
      
      return new Response(null, {
        status: 101,
        webSocket: client
      });
    }
    
    return new Response('Expected WebSocket', { status: 400 });
  }
  
  async handleSession(ws: WebSocket) {
    ws.accept();
    this.sessions.add(ws);
    
    // Load room history from storage
    const history = await this.state.storage.get('messages') || [];
    ws.send(JSON.stringify({ type: 'history', messages: history }));
    
    ws.addEventListener('message', async (msg) => {
      const data = JSON.parse(msg.data);
      
      // Store message
      const messages = await this.state.storage.get('messages') || [];
      messages.push(data);
      await this.state.storage.put('messages', messages);
      
      // Broadcast to all connections
      this.broadcast(JSON.stringify(data));
    });
    
    ws.addEventListener('close', () => {
      this.sessions.delete(ws);
    });
  }
  
  broadcast(message: string) {
    for (const session of this.sessions) {
      session.send(message);
    }
  }
}
```

**wrangler.toml:**
```toml
name = "chat-app"

[[durable_objects.bindings]]
name = "ROOMS"
class_name = "ChatRoom"
script_name = "chat-app"

[[kv_namespaces]]
binding = "USER_PROFILES"
id = "..."

[[migrations]]
tag = "v1"
new_classes = ["ChatRoom"]
```

### Key Patterns
1. **One DO per room/session**: Isolated state and connections
2. **WebSocket in DO**: All connections to one room in one DO instance
3. **Persistent state**: Use DO storage for history
4. **Broadcast pattern**: Fan-out messages to all connections

---

## Pattern 3: Full-Stack SaaS Application

### Architecture
```
Pages (React/Vue Frontend)
├─→ Pages Functions (API Layer)
│   ├─→ D1 (Primary database - users, data)
│   ├─→ R2 (File storage)
│   ├─→ KV (Cache, sessions)
│   └─→ Queues (Background jobs)
└─→ Workers (Queue consumers for async tasks)
```

### Use Cases
- Project management tools
- CRM systems
- E-commerce platforms
- Admin dashboards

### Implementation

**Database Schema (migrations/001_init.sql):**
```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE projects (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  name TEXT NOT NULL,
  data TEXT,
  created_at INTEGER NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE files (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  r2_key TEXT NOT NULL,
  filename TEXT NOT NULL,
  size INTEGER,
  created_at INTEGER NOT NULL,
  FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

**API Function (functions/api/projects.ts):**
```typescript
export const onRequestGet: PagesFunction<Env> = async ({ request, env }) => {
  // Get user session
  const userId = await getUserFromSession(request, env);
  if (!userId) return new Response('Unauthorized', { status: 401 });
  
  // Try cache first
  const cacheKey = `projects:${userId}`;
  const cached = await env.CACHE.get(cacheKey);
  if (cached) return new Response(cached, {
    headers: { 'Content-Type': 'application/json' }
  });
  
  // Query database
  const { results } = await env.DB.prepare(
    'SELECT * FROM projects WHERE user_id = ?'
  ).bind(userId).all();
  
  const response = JSON.stringify(results);
  
  // Cache for 5 minutes
  await env.CACHE.put(cacheKey, response, { expirationTtl: 300 });
  
  return new Response(response, {
    headers: { 'Content-Type': 'application/json' }
  });
};

export const onRequestPost: PagesFunction<Env> = async ({ request, env }) => {
  const userId = await getUserFromSession(request, env);
  const { name, data } = await request.json();
  
  const projectId = crypto.randomUUID();
  
  await env.DB.prepare(
    'INSERT INTO projects (id, user_id, name, data, created_at) VALUES (?, ?, ?, ?, ?)'
  ).bind(projectId, userId, name, data, Date.now()).run();
  
  // Invalidate cache
  await env.CACHE.delete(`projects:${userId}`);
  
  // Queue background task
  await env.TASKS.send({
    type: 'project_created',
    projectId,
    userId
  });
  
  return new Response(JSON.stringify({ id: projectId }), { status: 201 });
};
```

**File Upload (functions/api/upload.ts):**
```typescript
export const onRequestPost: PagesFunction<Env> = async ({ request, env }) => {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  const projectId = formData.get('projectId') as string;
  
  // Upload to R2
  const key = `${projectId}/${crypto.randomUUID()}-${file.name}`;
  await env.UPLOADS.put(key, file.stream());
  
  // Save metadata to D1
  await env.DB.prepare(
    'INSERT INTO files (id, project_id, r2_key, filename, size, created_at) VALUES (?, ?, ?, ?, ?, ?)'
  ).bind(crypto.randomUUID(), projectId, key, file.name, file.size, Date.now()).run();
  
  return new Response(JSON.stringify({ key }), { status: 201 });
};
```

**Background Worker (src/queue-consumer.ts):**
```typescript
export default {
  async queue(batch: MessageBatch<QueueMessage>, env: Env): Promise<void> {
    for (const message of batch.messages) {
      switch (message.body.type) {
        case 'project_created':
          await handleProjectCreated(message.body, env);
          break;
        // Other task types...
      }
      message.ack();
    }
  }
};

async function handleProjectCreated(data: any, env: Env) {
  // Send welcome email, create default files, etc.
  // This runs async, doesn't block user request
}
```

**wrangler.toml:**
```toml
name = "saas-app"
pages_build_output_dir = "dist"

[[d1_databases]]
binding = "DB"
database_name = "saas-db"
database_id = "..."

[[r2_buckets]]
binding = "UPLOADS"
bucket_name = "project-files"

[[kv_namespaces]]
binding = "SESSIONS"
id = "..."

[[kv_namespaces]]
binding = "CACHE"
id = "..."

[[queues.producers]]
binding = "TASKS"
queue = "background-tasks"

# Separate worker for queue consumer
[workers]
name = "queue-consumer"
main = "src/queue-consumer.ts"

[[queues.consumers]]
queue = "background-tasks"
max_batch_size = 10
max_batch_timeout = 5
```

### Key Patterns
1. **Cache-aside**: Check KV cache before DB
2. **Cache invalidation**: Delete cache keys on writes
3. **Async processing**: Use Queues for non-blocking tasks
4. **File storage**: R2 for files, D1 for metadata
5. **Session management**: KV for fast session lookups

---

## Pattern 4: AI Application with RAG

### Architecture
```
Worker (Orchestrator)
├─→ AI Gateway (Proxy to OpenAI/Anthropic)
│   └─→ LLM (GPT-4, Claude, etc.)
├─→ Vectorize (Semantic search)
├─→ Workers AI (Embeddings generation)
├─→ D1 (Conversation history, documents)
└─→ KV (User preferences, system prompts)
```

### Use Cases
- Chatbots with knowledge base
- Document Q&A systems
- AI assistants
- Semantic search

### Implementation

**Document Ingestion Worker:**
```typescript
interface Env {
  AI: any; // Workers AI binding
  VECTORIZE: VectorizeIndex;
  DB: D1Database;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { content, documentId } = await request.json();
    
    // Split into chunks
    const chunks = splitIntoChunks(content, 500);
    
    // Generate embeddings
    const embeddings = await Promise.all(
      chunks.map(chunk => 
        env.AI.run('@cf/baai/bge-base-en-v1.5', {
          text: chunk
        })
      )
    );
    
    // Store in Vectorize
    const vectors = chunks.map((chunk, i) => ({
      id: `${documentId}-${i}`,
      values: embeddings[i].data[0],
      metadata: { documentId, chunk }
    }));
    
    await env.VECTORIZE.upsert(vectors);
    
    // Store document in D1
    await env.DB.prepare(
      'INSERT INTO documents (id, content, indexed_at) VALUES (?, ?, ?)'
    ).bind(documentId, content, Date.now()).run();
    
    return new Response(JSON.stringify({ success: true }));
  }
};
```

**Chat Worker with RAG:**
```typescript
interface Env {
  AI_GATEWAY: any;
  VECTORIZE: VectorizeIndex;
  DB: D1Database;
  AI: any;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { question, conversationId } = await request.json();
    
    // 1. Generate embedding for question
    const questionEmbedding = await env.AI.run('@cf/baai/bge-base-en-v1.5', {
      text: question
    });
    
    // 2. Search similar chunks
    const results = await env.VECTORIZE.query(questionEmbedding.data[0], {
      topK: 5
    });
    
    const context = results.matches
      .map(m => m.metadata.chunk)
      .join('\n\n');
    
    // 3. Get conversation history
    const { results: history } = await env.DB.prepare(
      'SELECT * FROM messages WHERE conversation_id = ? ORDER BY created_at DESC LIMIT 10'
    ).bind(conversationId).all();
    
    // 4. Build prompt
    const messages = [
      {
        role: 'system',
        content: `Answer based on this context:\n\n${context}`
      },
      ...history.reverse(),
      { role: 'user', content: question }
    ];
    
    // 5. Call LLM through AI Gateway
    const response = await fetch(
      `https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_name}/openai/chat/completions`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          model: 'gpt-4',
          messages
        })
      }
    );
    
    const { choices } = await response.json();
    const answer = choices[0].message.content;
    
    // 6. Store conversation
    await env.DB.batch([
      env.DB.prepare(
        'INSERT INTO messages (conversation_id, role, content, created_at) VALUES (?, ?, ?, ?)'
      ).bind(conversationId, 'user', question, Date.now()),
      env.DB.prepare(
        'INSERT INTO messages (conversation_id, role, content, created_at) VALUES (?, ?, ?, ?)'
      ).bind(conversationId, 'assistant', answer, Date.now())
    ]);
    
    return new Response(JSON.stringify({ answer }));
  }
};
```

**wrangler.toml:**
```toml
name = "ai-chat"

[[vectorize]]
binding = "VECTORIZE"
index_name = "knowledge-base"

[[d1_databases]]
binding = "DB"
database_name = "chat-db"
database_id = "..."

[ai]
binding = "AI"
```

### Key Patterns
1. **RAG Pipeline**: Embed → Search → Context → Generate
2. **AI Gateway**: Cache and monitor LLM calls
3. **Vectorize**: Semantic search over embeddings
4. **Workers AI**: Generate embeddings at edge
5. **Conversation storage**: D1 for history

---

## Pattern 5: External Database Integration

### Architecture
```
Worker (API)
└─→ Hyperdrive (Connection pool)
    └─→ PostgreSQL/MySQL (External/existing)
```

### Use Cases
- Connecting to existing databases
- Gradual migration to Cloudflare
- Enterprise integration
- Multi-cloud setups

### Implementation

**Worker with Hyperdrive:**
```typescript
interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Hyperdrive provides connection string
    const client = postgres(env.HYPERDRIVE.connectionString);
    
    try {
      const url = new URL(request.url);
      const userId = url.searchParams.get('userId');
      
      // Query through Hyperdrive
      const users = await client`
        SELECT * FROM users 
        WHERE id = ${userId}
      `;
      
      return new Response(JSON.stringify(users));
    } finally {
      await client.end();
    }
  }
};
```

**With Query Caching:**
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const cacheKey = new Request(request.url);
    const cache = caches.default;
    
    // Try cache first
    let response = await cache.match(cacheKey);
    if (response) return response;
    
    // Query through Hyperdrive
    const client = postgres(env.HYPERDRIVE.connectionString);
    const data = await client`SELECT * FROM products WHERE featured = true`;
    await client.end();
    
    response = new Response(JSON.stringify(data), {
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'max-age=300'
      }
    });
    
    await cache.put(cacheKey, response.clone());
    return response;
  }
};
```

**wrangler.toml:**
```toml
name = "hyperdrive-api"

[[hyperdrive]]
binding = "HYPERDRIVE"
id = "..." # Created via wrangler hyperdrive create
```

### Key Patterns
1. **Connection pooling**: Hyperdrive manages connections
2. **Query caching**: Hyperdrive caches queries automatically
3. **Existing DB**: No data migration required
4. **Gradual migration**: Move logic to Workers while keeping DB
5. **Multiple databases**: Can have multiple Hyperdrive bindings

---

## Pattern 6: Background Processing Pipeline

### Architecture
```
Worker (Trigger)
├─→ Queue (Async tasks)
│   └─→ Worker (Consumer)
│       ├─→ Browser Rendering (Screenshots, PDFs)
│       ├─→ R2 (Store results)
│       └─→ D1 (Status tracking)
└─→ Immediate response to user
```

### Use Cases
- Web scraping
- Report generation
- Batch processing
- ETL pipelines
- Screenshot services

### Implementation

**Producer Worker:**
```typescript
interface Env {
  TASKS: Queue;
  DB: D1Database;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { url, userId } = await request.json();
    
    const jobId = crypto.randomUUID();
    
    // Create job record
    await env.DB.prepare(
      'INSERT INTO jobs (id, user_id, status, created_at) VALUES (?, ?, ?, ?)'
    ).bind(jobId, userId, 'pending', Date.now()).run();
    
    // Queue the task
    await env.TASKS.send({
      jobId,
      url,
      userId
    });
    
    // Return immediately
    return new Response(JSON.stringify({ jobId }), { status: 202 });
  }
};
```

**Consumer Worker:**
```typescript
interface Env {
  BROWSER: Fetcher;
  RESULTS: R2Bucket;
  DB: D1Database;
}

export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      try {
        await processJob(message.body, env);
        message.ack();
      } catch (error) {
        message.retry();
      }
    }
  }
};

async function processJob(job: any, env: Env) {
  // Update status
  await env.DB.prepare(
    'UPDATE jobs SET status = ? WHERE id = ?'
  ).bind('processing', job.jobId).run();
  
  // Take screenshot
  const screenshot = await env.BROWSER.fetch(
    `https://api.cloudflare.com/puppeteer/screenshot?url=${job.url}`
  );
  
  const imageData = await screenshot.arrayBuffer();
  
  // Store in R2
  const key = `screenshots/${job.jobId}.png`;
  await env.RESULTS.put(key, imageData);
  
  // Update status
  await env.DB.prepare(
    'UPDATE jobs SET status = ?, result_key = ?, completed_at = ? WHERE id = ?'
  ).bind('completed', key, Date.now(), job.jobId).run();
}
```

**Status Check Worker:**
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const jobId = url.searchParams.get('jobId');
    
    const job = await env.DB.prepare(
      'SELECT * FROM jobs WHERE id = ?'
    ).bind(jobId).first();
    
    if (job.status === 'completed') {
      // Return signed URL for R2 object
      const signedUrl = await env.RESULTS.createSignedUrl(job.result_key, {
        expiresIn: 3600
      });
      
      return new Response(JSON.stringify({
        status: 'completed',
        downloadUrl: signedUrl
      }));
    }
    
    return new Response(JSON.stringify({ status: job.status }));
  }
};
```

**wrangler.toml:**
```toml
# Producer
name = "job-producer"
main = "src/producer.ts"

[[queues.producers]]
binding = "TASKS"
queue = "processing-queue"

[[d1_databases]]
binding = "DB"
database_name = "jobs-db"
database_id = "..."

# Consumer
[[workers]]
name = "job-consumer"
main = "src/consumer.ts"

[[queues.consumers]]
queue = "processing-queue"
max_batch_size = 10
max_batch_timeout = 5
max_retries = 3

[[browser]]
binding = "BROWSER"

[[r2_buckets]]
binding = "RESULTS"
bucket_name = "job-results"
```

### Key Patterns
1. **Decoupled architecture**: Producer and consumer are separate
2. **Immediate response**: Return 202 Accepted immediately
3. **Status tracking**: Use D1 to track job status
4. **Batch processing**: Consumer processes multiple messages
5. **Retry logic**: Automatic retries on failure
6. **Result delivery**: Store in R2, return signed URL

---

## Pattern 7: Media Processing Platform

### Architecture
```
Worker (Upload Handler)
├─→ R2 (Store original)
├─→ Images (Transform/optimize)
├─→ Browser Rendering (Complex transformations)
└─→ D1 (Metadata, access control)
```

### Use Cases
- User-generated content platforms
- Media galleries
- Image optimization services
- Thumbnail generation

### Implementation

**Upload Worker:**
```typescript
interface Env {
  ORIGINALS: R2Bucket;
  THUMBNAILS: R2Bucket;
  DB: D1Database;
  QUEUE: Queue;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const formData = await request.formData();
    const file = formData.get('file') as File;
    const userId = formData.get('userId') as string;
    
    const imageId = crypto.randomUUID();
    const originalKey = `originals/${imageId}`;
    
    // Store original in R2
    await env.ORIGINALS.put(originalKey, file.stream(), {
      httpMetadata: {
        contentType: file.type
      }
    });
    
    // Store metadata in D1
    await env.DB.prepare(`
      INSERT INTO images (id, user_id, original_key, filename, size, mime_type, uploaded_at)
      VALUES (?, ?, ?, ?, ?, ?, ?)
    `).bind(
      imageId,
      userId,
      originalKey,
      file.name,
      file.size,
      file.type,
      Date.now()
    ).run();
    
    // Queue thumbnail generation
    await env.QUEUE.send({
      type: 'generate_thumbnails',
      imageId,
      originalKey
    });
    
    return new Response(JSON.stringify({ imageId }), { status: 201 });
  }
};
```

**Serve Worker with Transformations:**
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const imageId = url.pathname.split('/')[2];
    const width = url.searchParams.get('w');
    const height = url.searchParams.get('h');
    
    // Get metadata
    const image = await env.DB.prepare(
      'SELECT * FROM images WHERE id = ?'
    ).bind(imageId).first();
    
    if (!image) return new Response('Not found', { status: 404 });
    
    // Fetch original from R2
    const original = await env.ORIGINALS.get(image.original_key);
    if (!original) return new Response('Not found', { status: 404 });
    
    // Transform using Cloudflare Images
    if (width || height) {
      const transformedUrl = new URL(`https://your-account.cloudflareimages.com/${imageId}`);
      if (width) transformedUrl.searchParams.set('width', width);
      if (height) transformedUrl.searchParams.set('height', height);
      
      const transformed = await fetch(transformedUrl);
      return new Response(transformed.body, {
        headers: {
          'Content-Type': image.mime_type,
          'Cache-Control': 'public, max-age=31536000'
        }
      });
    }
    
    // Return original
    return new Response(original.body, {
      headers: {
        'Content-Type': image.mime_type,
        'Cache-Control': 'public, max-age=31536000'
      }
    });
  }
};
```

**Thumbnail Generator (Queue Consumer):**
```typescript
export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      if (message.body.type === 'generate_thumbnails') {
        await generateThumbnails(message.body, env);
      }
      message.ack();
    }
  }
};

async function generateThumbnails(data: any, env: Env) {
  const sizes = [
    { name: 'small', width: 150 },
    { name: 'medium', width: 300 },
    { name: 'large', width: 600 }
  ];
  
  for (const size of sizes) {
    // Use Cloudflare Images to generate thumbnail
    const thumbUrl = `https://your-account.cloudflareimages.com/${data.imageId}?width=${size.width}`;
    const thumb = await fetch(thumbUrl);
    
    // Store thumbnail in R2
    const thumbKey = `thumbnails/${data.imageId}-${size.name}`;
    await env.THUMBNAILS.put(thumbKey, thumb.body);
    
    // Update metadata
    await env.DB.prepare(
      `UPDATE images SET ${size.name}_thumb = ? WHERE id = ?`
    ).bind(thumbKey, data.imageId).run();
  }
}
```

### Key Patterns
1. **Separation of concerns**: Original in one bucket, thumbnails in another
2. **Async processing**: Generate thumbnails in background
3. **On-demand transforms**: Transform on request with caching
4. **Metadata tracking**: D1 for searchability
5. **CDN caching**: Long cache times for immutable content

---

## Pattern 8: Microservices Architecture with Service Bindings

### Architecture
```
Worker (API Gateway)
├─→ Auth Service (Service Binding)
│   ├─→ D1 (User accounts)
│   └─→ KV (Session tokens)
├─→ Data Service (Service Binding)
│   ├─→ D1 (Application data)
│   └─→ R2 (File storage)
└─→ Analytics Service (Service Binding)
    └─→ Analytics Engine (Event tracking)
```

### Use Cases
- Splitting monolithic Workers into services
- Team boundaries (different teams own different services)
- Independent deployment of services
- Clear separation of concerns

### Implementation

**API Gateway Worker:**
```typescript
interface Env {
  AUTH_SERVICE: Fetcher;
  DATA_SERVICE: Fetcher;
  ANALYTICS_SERVICE: Fetcher;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // Route to appropriate service
    if (url.pathname.startsWith('/api/auth')) {
      return env.AUTH_SERVICE.fetch(request);
    }

    if (url.pathname.startsWith('/api/data')) {
      // Verify auth first
      const authCheck = await env.AUTH_SERVICE.fetch(
        new Request('https://fake-host/verify', {
          headers: { 'Authorization': request.headers.get('Authorization') }
        })
      );

      if (!authCheck.ok) {
        return new Response('Unauthorized', { status: 401 });
      }

      // Forward to data service
      const response = await env.DATA_SERVICE.fetch(request);

      // Track analytics (fire and forget)
      await env.ANALYTICS_SERVICE.fetch(
        new Request('https://fake-host/track', {
          method: 'POST',
          body: JSON.stringify({
            endpoint: url.pathname,
            userId: authCheck.headers.get('X-User-ID')
          })
        })
      );

      return response;
    }

    return new Response('Not found', { status: 404 });
  }
};
```

**Auth Service Worker:**
```typescript
interface Env {
  DB: D1Database;
  SESSIONS: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/login') {
      const { email, password } = await request.json();

      // Verify credentials
      const user = await env.DB.prepare(
        'SELECT id FROM users WHERE email = ? AND password_hash = ?'
      ).bind(email, hashPassword(password)).first();

      if (!user) return new Response('Invalid credentials', { status: 401 });

      // Create session
      const sessionId = crypto.randomUUID();
      await env.SESSIONS.put(sessionId, user.id, { expirationTtl: 86400 });

      return new Response(JSON.stringify({ sessionId }));
    }

    if (url.pathname === '/verify') {
      const authHeader = request.headers.get('Authorization');
      const sessionId = authHeader?.replace('Bearer ', '');

      const userId = await env.SESSIONS.get(sessionId);
      if (!userId) return new Response('Invalid session', { status: 401 });

      return new Response('OK', {
        headers: { 'X-User-ID': userId }
      });
    }

    return new Response('Not found', { status: 404 });
  }
};
```

**Data Service Worker:**
```typescript
interface Env {
  DB: D1Database;
  FILES: R2Bucket;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const userId = request.headers.get('X-User-ID'); // From auth service

    if (url.pathname === '/api/data/items') {
      if (request.method === 'GET') {
        const { results } = await env.DB.prepare(
          'SELECT * FROM items WHERE user_id = ?'
        ).bind(userId).all();

        return new Response(JSON.stringify(results));
      }

      if (request.method === 'POST') {
        const { name, data } = await request.json();
        const itemId = crypto.randomUUID();

        await env.DB.prepare(
          'INSERT INTO items (id, user_id, name, data) VALUES (?, ?, ?, ?)'
        ).bind(itemId, userId, name, data).run();

        return new Response(JSON.stringify({ id: itemId }), { status: 201 });
      }
    }

    return new Response('Not found', { status: 404 });
  }
};
```

**wrangler.toml (API Gateway):**
```toml
name = "api-gateway"
main = "src/gateway.ts"

[[services]]
binding = "AUTH_SERVICE"
service = "auth-service"
environment = "production"

[[services]]
binding = "DATA_SERVICE"
service = "data-service"
environment = "production"

[[services]]
binding = "ANALYTICS_SERVICE"
service = "analytics-service"
environment = "production"
```

**wrangler.toml (Auth Service):**
```toml
name = "auth-service"
main = "src/auth.ts"

[[d1_databases]]
binding = "DB"
database_name = "users-db"
database_id = "..."

[[kv_namespaces]]
binding = "SESSIONS"
id = "..."
```

**wrangler.toml (Data Service):**
```toml
name = "data-service"
main = "src/data.ts"

[[d1_databases]]
binding = "DB"
database_name = "app-db"
database_id = "..."

[[r2_buckets]]
binding = "FILES"
bucket_name = "user-files"
```

### Key Patterns
1. **Service Bindings**: Worker-to-Worker RPC without HTTP egress
2. **Gateway pattern**: Single entry point routes to services
3. **Auth propagation**: Verify once, pass user context to other services
4. **Independent deployment**: Each service has its own wrangler.toml
5. **Type safety**: Services share TypeScript interfaces
6. **No network latency**: Internal RPC is faster than HTTP

---

## Pattern 9: Secure Application Stack

### Architecture
```
User Request
├─→ WAF (First line of defense)
│   └─→ Blocks: SQL injection, XSS, malicious traffic
├─→ Worker (Application logic)
│   ├─→ Turnstile verification (bot protection)
│   ├─→ Rate limiting (Durable Object)
│   ├─→ Secrets Store (API keys, credentials)
│   └─→ D1 (Application data)
└─→ Response
```

### Use Cases
- Public-facing APIs
- E-commerce platforms
- User registration/login forms
- Payment processing
- High-security applications

### Implementation

**Worker with Security Layers:**
```typescript
interface Env {
  DB: D1Database;
  RATE_LIMITER: DurableObjectNamespace;
  TURNSTILE_SECRET_KEY: string; // From Secrets Store
  API_KEY: string; // From Secrets Store
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // 1. WAF runs before this (configured in dashboard)

    // 2. Verify Turnstile token (bot protection)
    if (request.method === 'POST' && url.pathname === '/api/signup') {
      const { email, password, turnstileToken } = await request.json();

      const turnstileValid = await verifyTurnstile(
        turnstileToken,
        env.TURNSTILE_SECRET_KEY,
        request.headers.get('CF-Connecting-IP')
      );

      if (!turnstileValid) {
        return new Response('Bot detection failed', { status: 403 });
      }

      // 3. Rate limiting
      const clientIP = request.headers.get('CF-Connecting-IP');
      const rateLimiterId = env.RATE_LIMITER.idFromName(clientIP);
      const rateLimiter = env.RATE_LIMITER.get(rateLimiterId);

      const allowed = await rateLimiter.fetch(
        new Request('https://fake-host/check')
      );

      if (!allowed.ok) {
        return new Response('Rate limit exceeded', { status: 429 });
      }

      // 4. Process signup
      const userId = crypto.randomUUID();
      await env.DB.prepare(
        'INSERT INTO users (id, email, password_hash) VALUES (?, ?, ?)'
      ).bind(userId, email, hashPassword(password)).run();

      return new Response(JSON.stringify({ userId }), { status: 201 });
    }

    // Protected API endpoint
    if (url.pathname.startsWith('/api/admin')) {
      const apiKey = request.headers.get('X-API-Key');

      if (apiKey !== env.API_KEY) {
        return new Response('Invalid API key', { status: 401 });
      }

      // Process admin request...
      return new Response(JSON.stringify({ status: 'ok' }));
    }

    return new Response('Not found', { status: 404 });
  }
};

async function verifyTurnstile(
  token: string,
  secret: string,
  ip: string
): Promise<boolean> {
  const formData = new URLSearchParams();
  formData.append('secret', secret);
  formData.append('response', token);
  formData.append('remoteip', ip);

  const response = await fetch(
    'https://challenges.cloudflare.com/turnstile/v0/siteverify',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: formData
    }
  );

  const data = await response.json<{ success: boolean }>();
  return data.success;
}

function hashPassword(password: string): string {
  // Use proper password hashing (bcrypt, scrypt, etc.)
  // This is just a placeholder
  return password; // Replace with actual hashing
}
```

**Rate Limiter Durable Object:**
```typescript
export class RateLimiter {
  state: DurableObjectState;

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const now = Date.now();
    const window = 60000; // 1 minute
    const limit = 10; // 10 requests per minute

    // Get request timestamps
    const timestamps = await this.state.storage.get<number[]>('timestamps') || [];

    // Remove old timestamps
    const recent = timestamps.filter(t => now - t < window);

    // Check limit
    if (recent.length >= limit) {
      return new Response('Rate limit exceeded', { status: 429 });
    }

    // Add current timestamp
    recent.push(now);
    await this.state.storage.put('timestamps', recent);

    return new Response('OK');
  }
}
```

**Frontend (Turnstile widget):**
```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
</head>
<body>
  <form id="signup-form">
    <input type="email" name="email" required />
    <input type="password" name="password" required />

    <!-- Turnstile widget -->
    <div class="cf-turnstile"
         data-sitekey="YOUR_SITE_KEY"
         data-callback="onTurnstileSuccess"></div>

    <button type="submit">Sign Up</button>
  </form>

  <script>
    let turnstileToken = null;

    function onTurnstileSuccess(token) {
      turnstileToken = token;
    }

    document.getElementById('signup-form').addEventListener('submit', async (e) => {
      e.preventDefault();

      const formData = new FormData(e.target);

      const response = await fetch('/api/signup', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email: formData.get('email'),
          password: formData.get('password'),
          turnstileToken
        })
      });

      if (response.ok) {
        alert('Signup successful!');
      }
    });
  </script>
</body>
</html>
```

**wrangler.toml:**
```toml
name = "secure-app"
main = "src/index.ts"

[[d1_databases]]
binding = "DB"
database_name = "secure-db"
database_id = "..."

[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"
script_name = "secure-app"

[[migrations]]
tag = "v1"
new_classes = ["RateLimiter"]

# Secrets (set via: wrangler secret put TURNSTILE_SECRET_KEY)
# TURNSTILE_SECRET_KEY
# API_KEY
```

**WAF Configuration (Dashboard):**
```
1. Enable WAF Managed Rules
   - OWASP Core Ruleset
   - Cloudflare Managed Ruleset

2. Create custom rules:
   - Block requests with SQL injection patterns
   - Block requests with XSS patterns
   - Block requests from known bad IPs

3. Enable Bot Fight Mode or Super Bot Fight Mode
```

### Key Patterns
1. **Defense in depth**: Multiple security layers (WAF → Turnstile → Rate limiting → Auth)
2. **Bot protection**: Turnstile for CAPTCHA-free verification
3. **Rate limiting**: Per-IP limits using Durable Objects
4. **Secrets management**: Store sensitive keys in Secrets Store (not in code)
5. **WAF first**: WAF blocks attacks before they reach Workers
6. **IP-based controls**: Use `CF-Connecting-IP` header for rate limiting

---

## Pattern 10: Enterprise Integration with Workers VPC

### Architecture
```
Worker (Edge)
└─→ Workers VPC (Private connectivity)
    ├─→ AWS RDS (PostgreSQL)
    ├─→ Azure SQL Database
    ├─→ Internal APIs (private network)
    └─→ Legacy systems (on-premise)
```

### Use Cases
- Enterprise cloud migration
- Hybrid cloud architectures
- Connecting to private databases
- Legacy system integration
- Multi-cloud setups

### Implementation

**Worker with VPC Connectivity:**
```typescript
interface Env {
  PRIVATE_DB: Fetcher; // VPC endpoint
  INTERNAL_API: Fetcher; // VPC endpoint
  CACHE: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // Public API that proxies to private systems
    if (url.pathname === '/api/customers') {
      const customerId = url.searchParams.get('id');

      // Try cache first
      const cacheKey = `customer:${customerId}`;
      const cached = await env.CACHE.get(cacheKey);
      if (cached) {
        return new Response(cached, {
          headers: { 'Content-Type': 'application/json' }
        });
      }

      // Query private database via VPC
      const dbRequest = new Request('https://private-db/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          query: 'SELECT * FROM customers WHERE id = $1',
          params: [customerId]
        })
      });

      const dbResponse = await env.PRIVATE_DB.fetch(dbRequest);
      const customer = await dbResponse.json();

      // Enrich with data from internal API
      const enrichRequest = new Request(`https://internal-api/enrich/${customerId}`);
      const enrichResponse = await env.INTERNAL_API.fetch(enrichRequest);
      const enrichment = await enrichResponse.json();

      const result = {
        ...customer,
        ...enrichment
      };

      const resultJson = JSON.stringify(result);

      // Cache for 5 minutes
      await env.CACHE.put(cacheKey, resultJson, { expirationTtl: 300 });

      return new Response(resultJson, {
        headers: { 'Content-Type': 'application/json' }
      });
    }

    // Write operation to private database
    if (url.pathname === '/api/orders' && request.method === 'POST') {
      const order = await request.json();

      // Insert into private database
      const dbRequest = new Request('https://private-db/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          query: 'INSERT INTO orders (id, customer_id, total, items) VALUES ($1, $2, $3, $4)',
          params: [
            crypto.randomUUID(),
            order.customerId,
            order.total,
            JSON.stringify(order.items)
          ]
        })
      });

      const dbResponse = await env.PRIVATE_DB.fetch(dbRequest);

      if (!dbResponse.ok) {
        return new Response('Failed to create order', { status: 500 });
      }

      // Invalidate cache
      await env.CACHE.delete(`customer:${order.customerId}`);

      // Notify internal systems
      await env.INTERNAL_API.fetch(
        new Request('https://internal-api/notify/order-created', {
          method: 'POST',
          body: JSON.stringify({ orderId: order.id })
        })
      );

      return new Response(JSON.stringify({ success: true }), { status: 201 });
    }

    return new Response('Not found', { status: 404 });
  }
};
```

**VPC Connector (Proxy Worker in VPC):**
```typescript
// This worker runs inside the VPC and has access to private resources
import { Client } from 'pg';

interface Env {
  DB_HOST: string;
  DB_USER: string;
  DB_PASSWORD: string;
  DB_NAME: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/query' && request.method === 'POST') {
      const { query, params } = await request.json();

      // Connect to private PostgreSQL
      const client = new Client({
        host: env.DB_HOST,
        user: env.DB_USER,
        password: env.DB_PASSWORD,
        database: env.DB_NAME,
        ssl: false // Internal network
      });

      await client.connect();

      try {
        const result = await client.query(query, params);
        return new Response(JSON.stringify(result.rows));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), {
          status: 500
        });
      } finally {
        await client.end();
      }
    }

    return new Response('Not found', { status: 404 });
  }
};
```

**wrangler.toml (Edge Worker):**
```toml
name = "enterprise-api"
main = "src/index.ts"

# VPC Service Bindings
[[services]]
binding = "PRIVATE_DB"
service = "vpc-db-connector"
environment = "production"

[[services]]
binding = "INTERNAL_API"
service = "vpc-api-connector"
environment = "production"

[[kv_namespaces]]
binding = "CACHE"
id = "..."
```

**wrangler.toml (VPC Connector):**
```toml
name = "vpc-db-connector"
main = "src/vpc-connector.ts"

# This worker runs in Workers VPC
[vpc]
enabled = true
network_id = "your-vpc-network-id"

# Secrets for database credentials
# DB_HOST, DB_USER, DB_PASSWORD, DB_NAME
```

### Key Patterns
1. **VPC isolation**: Connectors run in private network, edge workers don't
2. **Service Bindings**: Edge workers communicate with VPC workers via bindings
3. **Caching**: Cache private data at the edge to reduce VPC calls
4. **Security**: Database credentials only in VPC workers, never exposed
5. **Proxy pattern**: VPC workers act as proxies to private resources
6. **Hybrid architecture**: Public edge + private backend
7. **Multi-cloud**: Connect to AWS, Azure, GCP, on-premise from one Worker

---

## Choosing the Right Pattern

| Your App | Recommended Pattern |
|----------|---------------------|
| Blog with comments | Pattern 1: Content Site |
| Chat application | Pattern 2: Real-Time Collaboration |
| SaaS product | Pattern 3: Full-Stack Application |
| AI chatbot | Pattern 4: AI with RAG |
| Existing DB | Pattern 5: External DB Integration |
| Report generator | Pattern 6: Background Processing |
| Photo sharing | Pattern 7: Media Processing |
| Microservices architecture | Pattern 8: Service Bindings |
| Public-facing API with security | Pattern 9: Secure Application Stack |
| Enterprise cloud integration | Pattern 10: Workers VPC |

Mix and match these patterns as needed. Most real applications combine multiple patterns.

### Common Pattern Combinations

- **Secure SaaS**: Pattern 3 (Full-Stack) + Pattern 9 (Security)
- **Enterprise API**: Pattern 8 (Microservices) + Pattern 10 (VPC) + Pattern 9 (Security)
- **Real-time with AI**: Pattern 2 (Real-Time) + Pattern 4 (AI with RAG)
- **Background processing with security**: Pattern 6 (Background) + Pattern 9 (Security)
- **Multi-service architecture**: Pattern 8 (Microservices) + Pattern 5 (External DB)
