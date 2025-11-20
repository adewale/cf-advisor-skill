# Migration Guide: Traditional Servers → Cloudflare Workers

This guide helps developers migrate from traditional server-based architectures to Cloudflare's edge platform.

## Migration Strategies

### Strategy 1: Lift and Optimize (Recommended for most)

**Timeline**: 2-4 weeks  
**Risk**: Low  
**Approach**: Incremental migration with minimal code changes

**Steps**:
1. Deploy static assets to Pages
2. Move API routes to Pages Functions or Workers
3. Keep existing database, add Hyperdrive for connection pooling
4. Migrate sessions to KV
5. Move file storage to R2
6. Add Queues for background jobs

**Example Migration Path**:
```
Traditional Stack:
nginx → Node.js/Express → PostgreSQL
↓
Cloudflare Stack:
Pages (static) → Pages Functions (API) → Hyperdrive → PostgreSQL
+ KV (sessions) + R2 (files) + Queues (jobs)
```

### Strategy 2: Greenfield Rebuild

**Timeline**: 4-12 weeks  
**Risk**: Medium  
**Approach**: Rebuild with Cloudflare-native patterns

**When to use**:
- Legacy codebase needs refactoring anyway
- Want to adopt edge-first architecture fully
- Have time for proper rewrite

### Strategy 3: Hybrid Approach

**Timeline**: 1-2 weeks  
**Risk**: Very Low  
**Approach**: Add Cloudflare in front of existing infrastructure

**Steps**:
1. Add Workers as proxy/middleware
2. Implement caching at edge
3. Gradually move logic to Workers
4. Keep existing backend as-is initially

## Common Migration Patterns

### Pattern 1: Express.js App → Pages Functions

**Before (Express)**:
```javascript
// server.js
const express = require('express');
const app = express();

app.get('/api/users', async (req, res) => {
  const users = await db.query('SELECT * FROM users');
  res.json(users);
});

app.listen(3000);
```

**After (Pages Functions)**:
```javascript
// functions/api/users.ts
export const onRequestGet: PagesFunction<Env> = async ({ env }) => {
  const users = await env.DB.prepare('SELECT * FROM users').all();
  return new Response(JSON.stringify(users.results), {
    headers: { 'Content-Type': 'application/json' }
  });
};
```

### Pattern 2: File Storage → R2

**Before (Local filesystem)**:
```javascript
const fs = require('fs');
const multer = require('multer');
const upload = multer({ dest: 'uploads/' });

app.post('/upload', upload.single('file'), (req, res) => {
  // File saved to uploads/ directory
  res.json({ path: req.file.path });
});
```

**After (R2)**:
```javascript
export const onRequestPost: PagesFunction<Env> = async ({ request, env }) => {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  
  const key = `uploads/${Date.now()}-${file.name}`;
  await env.UPLOADS.put(key, file.stream());
  
  return new Response(JSON.stringify({ key }));
};
```

### Pattern 3: Sessions → KV

**Before (In-memory sessions)**:
```javascript
const session = require('express-session');
app.use(session({
  secret: 'secret',
  resave: false,
  saveUninitialized: true
}));
```

**After (KV)**:
```javascript
// Store session
await env.SESSIONS.put(sessionId, JSON.stringify(sessionData), {
  expirationTtl: 86400 // 24 hours
});

// Retrieve session
const sessionData = await env.SESSIONS.get(sessionId, 'json');
```

### Pattern 4: Background Jobs → Queues

**Before (Node.js workers)**:
```javascript
const queue = require('bull');
const emailQueue = queue('emails');

emailQueue.process(async (job) => {
  await sendEmail(job.data);
});

// Add job
await emailQueue.add({ to: 'user@example.com', subject: '...' });
```

**After (Cloudflare Queues)**:
```javascript
// Producer (add to queue)
await env.EMAIL_QUEUE.send({
  to: 'user@example.com',
  subject: '...'
});

// Consumer Worker
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      await sendEmail(message.body);
      message.ack();
    }
  }
};
```

### Pattern 5: WebSockets → Durable Objects

**Before (Socket.io)**:
```javascript
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  socket.on('message', (msg) => {
    io.emit('message', msg); // Broadcast
  });
});
```

**After (Durable Objects)**:
```javascript
export class ChatRoom {
  sessions = new Set();
  
  async fetch(request) {
    const { 0: client, 1: server } = new WebSocketPair();
    server.accept();
    this.sessions.add(server);
    
    server.addEventListener('message', (event) => {
      // Broadcast to all connections
      for (const session of this.sessions) {
        session.send(event.data);
      }
    });
    
    return new Response(null, { status: 101, webSocket: client });
  }
}
```

## Database Migration Strategies

### Option 1: Keep Existing DB with Hyperdrive (Easiest)

**Pros**:
- No data migration needed
- Works with existing PostgreSQL/MySQL
- Connection pooling and caching built-in

**Cons**:
- Still tied to single database location
- Extra hop to database

**Setup**:
```bash
wrangler hyperdrive create my-db --connection-string="postgres://..."
```

### Option 2: Migrate to D1 (Best for new apps)

**Pros**:
- Serverless, no connection management
- Global read replicas
- Integrated with Workers platform

**Cons**:
- Requires data migration
- SQLite-based (some PostgreSQL features missing)

**Migration steps**:
1. Export data from existing DB
2. Transform schema to SQLite
3. Import to D1
4. Update queries

### Option 3: Hybrid Approach

**Use D1 for**:
- New features
- Read-heavy data
- App-specific data

**Use Hyperdrive + Existing DB for**:
- Existing features
- Complex queries
- Shared data with other systems

## Checklist: Pre-Migration

- [ ] Audit current architecture (inventory all services)
- [ ] Identify stateful vs stateless components
- [ ] Document database schema and queries
- [ ] List all external integrations
- [ ] Map file storage locations
- [ ] Document background job processes
- [ ] Identify WebSocket usage
- [ ] Review authentication/authorization flows

## Checklist: Post-Migration

- [ ] Monitor error rates and performance
- [ ] Test all critical user flows
- [ ] Verify database connections work
- [ ] Check file uploads/downloads
- [ ] Test background job processing
- [ ] Validate caching behavior
- [ ] Review cost optimization opportunities
- [ ] Document new architecture for team

## Common Gotchas & Solutions

### Gotcha 1: "My Workers keep timing out"
**Problem**: Trying to do too much in a single Worker request  
**Solution**: Use Queues for long-running tasks, keep Workers fast

### Gotcha 2: "KV isn't showing my latest writes"
**Problem**: KV is eventually consistent (60s propagation)  
**Solution**: Use D1 for data requiring immediate consistency

### Gotcha 3: "Can't install my favorite npm package"
**Problem**: Some Node.js packages don't work in Workers  
**Solution**: Use Workers-compatible alternatives or polyfills

### Gotcha 4: "Database connection pool exhausted"
**Problem**: Each Worker request creates new DB connection  
**Solution**: Use Hyperdrive for connection pooling

### Gotcha 5: "WebSocket connections dropping"
**Problem**: Trying to handle WebSockets in regular Workers  
**Solution**: Use Durable Objects for WebSocket connections

## Resources

- [Cloudflare Docs](https://developers.cloudflare.com)
- [Workers Examples](https://developers.cloudflare.com/workers/examples)
- [Discord Community](https://discord.gg/cloudflaredev)
- [Workers Playground](https://workers.cloudflare.com/playground)

## Need Help?

- Check primitives-catalog.md for product details
- Read composition-patterns.md for architecture examples
- Review SKILL.md for mental models
- Search cloudflare-docs.md for specific topics
