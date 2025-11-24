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
- Query caching for frequent queries

**Cons**:
- Still tied to single database location
- Extra hop to database
- External database hosting costs

**Setup**:

```bash
# Create Hyperdrive configuration
wrangler hyperdrive create my-db \
  --connection-string="postgres://user:password@host:5432/dbname"

# Add to wrangler.toml
```

```toml
# wrangler.toml
[[hyperdrive]]
binding = "HYPERDRIVE"
id = "your-hyperdrive-id"
```

**Use in Worker**:

```typescript
import { Client } from 'pg'; // or 'mysql2'

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Hyperdrive provides optimized connection string
    const client = new Client({ connectionString: env.HYPERDRIVE.connectionString });
    await client.connect();

    const result = await client.query('SELECT * FROM users');
    await client.end();

    return Response.json(result.rows);
  }
};
```

**Supported Databases**:
- PostgreSQL (AWS RDS, Google Cloud SQL, Azure, Neon, Supabase, etc.)
- MySQL (AWS RDS, Google Cloud SQL, PlanetScale, etc.)
- CockroachDB
- Timescale

---

### Option 2: Migrate to D1 (Best for new apps)

**Source**: https://developers.cloudflare.com/d1/build-with-d1/import-export-data/

**Pros**:
- Serverless, no connection management
- Global read replicas
- Integrated with Workers platform
- No database hosting costs

**Cons**:
- Requires data migration
- SQLite-based (some PostgreSQL features missing)
- 10 GB database size limit (scale horizontally with multiple databases)

#### Complete D1 Migration Process

**Step 1: Create D1 database**

```bash
# Create database
npx wrangler d1 create my-database

# Output shows database ID - add to wrangler.toml
```

```toml
# wrangler.toml
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

**Step 2: Export data from existing database**

```bash
# From PostgreSQL
pg_dump mydb > export.sql

# From MySQL
mysqldump -u user -p mydb > export.sql

# From SQLite
sqlite3 mydb.sqlite .dump > export.sql
```

**Step 3: Transform schema for SQLite compatibility**

```sql
-- PostgreSQL → SQLite transformations needed:

-- 1. Remove SERIAL → Use INTEGER PRIMARY KEY AUTOINCREMENT
-- Before:
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE
);

-- After:
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE
);

-- 2. Change VARCHAR/TEXT types → TEXT
-- SQLite uses TEXT for all string types

-- 3. Remove unsupported features:
--    - ENUM types → Use TEXT with CHECK constraints
--    - Complex DEFAULT expressions → Simplify or handle in application
--    - Some PostgreSQL-specific functions

-- 4. Handle foreign keys carefully
--    - Ensure parent tables created before child tables
--    - D1 supports foreign keys but must be explicitly enabled
```

**Step 4: Prepare import file**

**CRITICAL**: D1 import requirements:
- Remove `BEGIN TRANSACTION` and `COMMIT` statements
- Remove `_cf_KV` system table creation if present
- Maximum file size: 5 GiB
- Split large multi-row INSERT statements
- Ensure proper SQL statement order (parent tables first)

```bash
# Clean up SQL file
cat export.sql \
  | grep -v "BEGIN TRANSACTION" \
  | grep -v "COMMIT" \
  | grep -v "_cf_KV" \
  > cleaned.sql
```

**Step 5: Import to D1**

```bash
# Import SQL file
npx wrangler d1 execute my-database --remote --file=cleaned.sql

# Or run migrations
npx wrangler d1 migrations create my-database create_users_table
# Edit migrations/0001_create_users_table.sql
npx wrangler d1 migrations apply my-database --remote
```

**Step 6: Verify data**

```bash
# Query D1 to verify import
npx wrangler d1 execute my-database --remote --command="SELECT COUNT(*) FROM users"
```

**Step 7: Update Worker code**

```typescript
// Before (PostgreSQL with Hyperdrive)
import { Client } from 'pg';
const client = new Client({ connectionString: env.HYPERDRIVE.connectionString });
await client.connect();
const result = await client.query('SELECT * FROM users WHERE id = $1', [userId]);

// After (D1)
const { results } = await env.DB.prepare(
  'SELECT * FROM users WHERE id = ?'
).bind(userId).all();
```

#### D1 Limits & Considerations

**Source**: https://developers.cloudflare.com/d1/platform/limits/

- **Database size**: 10 GB maximum (design for horizontal scale-out)
- **Query timeout**: 30 seconds
- **Queries per invocation**: 50 (Free) / 1,000 (Paid)
- **Rows read**: Unlimited (but charged per row)
- **Single-threaded**: Query processing is single-threaded per database

**Horizontal Scaling Pattern**:

```typescript
// Instead of one 50GB database, use multiple 10GB databases
interface Env {
  DB_SHARD_1: D1Database;
  DB_SHARD_2: D1Database;
  DB_SHARD_3: D1Database;
  DB_SHARD_4: D1Database;
  DB_SHARD_5: D1Database;
}

function getShardForUser(userId: string): number {
  // Hash user ID to determine shard
  const hash = userId.split('').reduce((acc, char) => acc + char.charCodeAt(0), 0);
  return (hash % 5) + 1;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const userId = new URL(request.url).searchParams.get('userId');
    const shardNum = getShardForUser(userId!);
    const db = env[`DB_SHARD_${shardNum}` as keyof Env] as D1Database;

    const { results } = await db.prepare(
      'SELECT * FROM users WHERE id = ?'
    ).bind(userId).all();

    return Response.json(results);
  }
};
```

#### D1 Export Procedures

```bash
# Export full database
npx wrangler d1 export my-database --remote --output=./backup.sql

# Export single table
npx wrangler d1 export my-database --remote --output=./users.sql --table=users

# Export schema only (no data)
npx wrangler d1 export my-database --remote --output=./schema.sql --no-data
```

---

### Option 3: Hybrid Approach

**Use D1 for**:
- New features
- Read-heavy data
- App-specific data
- Data under 10GB per logical partition

**Use Hyperdrive + Existing DB for**:
- Existing features during migration
- Complex queries not supported in SQLite
- Shared data with other systems
- Gradual migration approach

**Pattern**:

```typescript
interface Env {
  DB: D1Database;           // New features
  HYPERDRIVE: Hyperdrive;   // Legacy data
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // New feature: Use D1
    const newUserData = await env.DB.prepare(
      'SELECT * FROM user_preferences WHERE user_id = ?'
    ).bind(userId).all();

    // Legacy feature: Use Hyperdrive
    const client = new Client({ connectionString: env.HYPERDRIVE.connectionString });
    await client.connect();
    const legacyOrders = await client.query(
      'SELECT * FROM orders WHERE user_id = $1',
      [userId]
    );
    await client.end();

    return Response.json({
      preferences: newUserData.results,
      orders: legacyOrders.rows
    });
  }
};
```

---

## R2 Migration (from S3 or local storage)

**Source**: https://developers.cloudflare.com/r2/api/workers/workers-api-reference/

### R2 Consistency Guarantees

- **Strong consistency** for writes and deletes
- **Important**: "Once the Promise resolves, all subsequent read operations will see this key value pair globally"

No eventual consistency issues like KV!

### S3 to R2 Migration

**R2 is S3-compatible**, so minimal code changes needed:

```typescript
// Before (AWS S3)
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({
  region: 'us-east-1',
  credentials: { accessKeyId, secretAccessKey }
});

await s3.send(new PutObjectCommand({
  Bucket: 'my-bucket',
  Key: 'file.txt',
  Body: fileData
}));

// After (R2 with Workers API)
interface Env {
  UPLOADS: R2Bucket;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const file = await request.arrayBuffer();

    await env.UPLOADS.put('file.txt', file);

    return new Response('Uploaded', { status: 200 });
  }
};
```

### R2 Migration Strategies

**Option 1: Copy existing files to R2**

```bash
# Using rclone or AWS CLI with R2 endpoint
rclone copy s3:my-bucket r2:my-bucket --progress

# Or use Cloudflare's Super Slurper (via dashboard)
# Dashboard > R2 > Create bucket > Import data
```

**Option 2: Gradual migration with dual-write**

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const file = await request.arrayBuffer();
    const key = 'uploads/file.txt';

    // Write to both S3 and R2 during migration
    await Promise.all([
      env.R2_BUCKET.put(key, file),
      uploadToS3(key, file, env)
    ]);

    // Read from R2 first, fallback to S3 if not found
    let object = await env.R2_BUCKET.get(key);

    if (!object) {
      // Backfill from S3 to R2
      const s3Data = await fetchFromS3(key, env);
      await env.R2_BUCKET.put(key, s3Data);
      return new Response(s3Data);
    }

    return new Response(object.body);
  }
};
```

### R2 Best Practices

```typescript
// Use storage classes for cost optimization
await env.BUCKET.put('archive/old-data.txt', data, {
  storageClass: 'InfrequentAccess' // Cheaper storage for rarely accessed data
});

// Use multipart uploads for large files (> 100MB)
const upload = await env.BUCKET.createMultipartUpload('large-file.bin');
// ... upload parts ...
await env.BUCKET.completeMultipartUpload('large-file.bin', upload.uploadId, parts);

// Use conditional operations for optimistic concurrency
await env.BUCKET.put('counter.txt', '1', {
  onlyIf: { etagDoesNotMatch: existingEtag } // Only write if not changed
});
```

---

## Traffic Migration Strategies

**Source**: https://developers.cloudflare.com/workers/configuration/routing/

### Routing Options

1. **Custom Domains** - Worker serves as origin server
2. **Routes** - Worker intercepts traffic to existing origin
3. **workers.dev** - Development/testing subdomain (not for production)

### Zero-Downtime Traffic Migration

**Phase 1: Deploy Worker in shadow mode (no traffic)**

```bash
# Deploy Worker to workers.dev
npx wrangler deploy
# Output: https://my-worker.username.workers.dev
```

Test thoroughly on workers.dev subdomain.

**Phase 2: Route small percentage of traffic**

```toml
# wrangler.toml - Add route
[[routes]]
pattern = "example.com/api/*"
zone_name = "example.com"
```

Use gradual deployments to control traffic percentage.

**Phase 3: Monitor and increase traffic**

```bash
# Monitor with real-time logs
npx wrangler tail --env production

# Check error rates in dashboard
# Workers & Pages > my-worker > Metrics
```

Gradually increase from 10% → 50% → 100%

**Phase 4: Full cutover**

Once 100% traffic on Worker with no issues:
- Update DNS if using custom domains
- Decommission old infrastructure

### Rollback Procedures

```bash
# View recent versions
npx wrangler versions list

# View current deployment
npx wrangler deployments list

# Instant rollback to previous version
npx wrangler rollback

# Or rollback to specific version
npx wrangler versions deploy <version-id>
```

**Rollback speed**: Instant (traffic shifts immediately)

**Best practice**: Keep previous version deployed for 24 hours for quick rollback if needed.

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
