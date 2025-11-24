# Cloudflare Deployment Workflows & Production Best Practices

Complete guide to environment management, testing, deployment, and production operations for Cloudflare Workers.

## Table of Contents

1. [Environment Management](#environment-management)
2. [Secrets Management](#secrets-management)
3. [Testing Strategies](#testing-strategies)
4. [Deployment Strategies](#deployment-strategies)
5. [Observability & Monitoring](#observability--monitoring)
6. [Error Handling](#error-handling)
7. [Cost Optimization](#cost-optimization)
8. [Platform Limits & Constraints](#platform-limits--constraints)

---

## Environment Management

### Development vs Staging vs Production

**Source**: https://developers.cloudflare.com/workers/platform/environments/

Cloudflare environments create separate Worker deployments with the naming convention `<top-level-name>-<environment-name>`.

### Configuration Structure

```toml
# wrangler.toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2025-03-07"

# Development environment
[env.dev]
name = "my-worker-dev"
vars = { ENVIRONMENT = "development", LOG_LEVEL = "debug" }

# Staging environment
[env.staging]
name = "my-worker-staging"
vars = { ENVIRONMENT = "staging", LOG_LEVEL = "info" }

# Production environment
[env.production]
name = "my-worker"
vars = { ENVIRONMENT = "production", LOG_LEVEL = "error" }
```

### Key Configuration Behaviors

**Inheritable Elements** (apply to all environments unless overridden):
- Route configurations
- General settings
- Base variables

**Non-Inheritable Elements** (must specify per environment):
- Bindings (KV namespaces, D1 databases, R2 buckets, Durable Objects)
- Environment variables and secrets
- Service bindings
- Compatibility flags (sometimes)

**Important**: "Bindings and environment variables are non-inheritable, and must be specified per environment in your Wrangler configuration file."

### Environment-Specific Bindings

```toml
# Development - use dev databases
[env.dev]
name = "my-worker-dev"
[[env.dev.kv_namespaces]]
binding = "SESSIONS"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" # Dev KV namespace

[[env.dev.d1_databases]]
binding = "DB"
database_name = "my-db-dev"
database_id = "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"

# Production - use production databases
[env.production]
name = "my-worker"
[[env.production.kv_namespaces]]
binding = "SESSIONS"
id = "zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz" # Prod KV namespace

[[env.production.d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
```

### Deployment Commands

```bash
# Local development
npx wrangler dev                    # Default environment
npx wrangler dev --env dev          # Specific environment

# Deploy to staging
npx wrangler deploy --env staging

# Deploy to production
npx wrangler deploy --env production

# View deployed Workers
npx wrangler deployments list
```

---

## Secrets Management

### Official Recommendation: Secrets vs Environment Variables

**Source**: https://developers.cloudflare.com/workers/configuration/secrets/

**Critical Distinction**:
- **Secrets**: Encrypted at rest, not visible in dashboard/Wrangler after creation
- **Environment Variables**: Plaintext, visible in configuration

**Best Practice**: "Do not use plaintext environment variables to store sensitive information. Use secrets instead."

### Production Secrets Workflow

```bash
# Set secret for specific environment
npx wrangler secret put API_KEY --env production

# For gradual deployments
npx wrangler versions secret put API_KEY

# List secrets (shows names only, not values)
npx wrangler secret list --env production

# Delete secret
npx wrangler secret delete API_KEY --env production
```

### Local Development Secrets

**Use `.dev.vars` file** (dotenv format):

```bash
# .dev.vars
DATABASE_URL="postgresql://localhost:5432/mydb"
API_KEY="dev-key-123"
STRIPE_SECRET="sk_test_..."
```

**Environment-specific dev files**:
```bash
# .dev.vars.staging
DATABASE_URL="postgresql://staging.example.com:5432/mydb"

# .dev.vars.production
DATABASE_URL="postgresql://prod.example.com:5432/mydb"
```

**CRITICAL**: Add to `.gitignore`:
```gitignore
.dev.vars
.dev.vars.*
.env
.env.*
```

### Access Secrets in Code

```typescript
interface Env {
  API_KEY: string;           // From wrangler secret
  DATABASE_URL: string;      // From wrangler secret
  ENVIRONMENT: string;       // From wrangler.toml vars
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Access secrets same as environment variables
    const apiKey = env.API_KEY;
    const dbUrl = env.DATABASE_URL;

    // Also available via process.env in Node.js-compatible Workers
    // const apiKey = process.env.API_KEY;

    return new Response('OK');
  }
};
```

### Secrets Rotation Strategy

```bash
#!/bin/bash
# rotate-secrets.sh

# Update secret in staging first
echo "Rotating API_KEY in staging..."
echo "$NEW_API_KEY" | npx wrangler secret put API_KEY --env staging

# Test staging deployment
curl https://my-worker-staging.example.com/health

# If successful, rotate in production
echo "Rotating API_KEY in production..."
echo "$NEW_API_KEY" | npx wrangler secret put API_KEY --env production
```

---

## Testing Strategies

### Official Testing Approach: Vitest Integration

**Source**: https://developers.cloudflare.com/workers/testing/

**Recommended**: Use Vitest with Cloudflare's integration for full-featured testing inside the Workers runtime.

### Setup Vitest Testing

```bash
# Install dependencies
npm install --save-dev vitest @cloudflare/vitest-pool-workers
```

```typescript
// vitest.config.ts
import { defineWorkersConfig } from '@cloudflare/vitest-pool-workers/config';

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: './wrangler.toml' },
        miniflare: {
          bindings: {
            TEST_MODE: 'true'
          }
        }
      }
    }
  }
});
```

### Unit Test Example

```typescript
// src/index.test.ts
import { describe, it, expect } from 'vitest';
import { env, createExecutionContext, waitOnExecutionContext } from 'cloudflare:test';
import worker from './index';

describe('Worker', () => {
  it('responds with 200 for GET /', async () => {
    const request = new Request('https://example.com/');
    const ctx = createExecutionContext();

    const response = await worker.fetch(request, env, ctx);
    await waitOnExecutionContext(ctx);

    expect(response.status).toBe(200);
  });

  it('accesses KV binding', async () => {
    // KV binding automatically available from env
    await env.SESSIONS.put('test-key', 'test-value');
    const value = await env.SESSIONS.get('test-key');

    expect(value).toBe('test-value');
  });
});
```

### Integration Test Example

```typescript
// src/integration.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { env } from 'cloudflare:test';

describe('Database Integration', () => {
  beforeAll(async () => {
    // Seed test database
    await env.DB.exec(`
      CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY,
        email TEXT UNIQUE,
        name TEXT
      );
    `);
  });

  it('creates and retrieves user', async () => {
    // Insert user
    await env.DB.prepare(
      'INSERT INTO users (email, name) VALUES (?, ?)'
    ).bind('test@example.com', 'Test User').run();

    // Retrieve user
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE email = ?'
    ).bind('test@example.com').all();

    expect(results).toHaveLength(1);
    expect(results[0].email).toBe('test@example.com');
  });
});
```

### Local Development & Testing

```bash
# Local dev server (uses .dev.vars)
npx wrangler dev

# Local dev with remote bindings (uses actual KV, D1, etc.)
npx wrangler dev --remote

# Run tests
npm test

# Run tests in watch mode
npm test -- --watch
```

### Testing Best Practices

1. **Isolated storage per test**: Vitest integration provides clean storage for each test
2. **Mock external APIs**: Use `fetchMock` or similar for external requests
3. **Test with real bindings**: Vitest loads actual wrangler.toml configuration
4. **Test error paths**: Verify error handling and edge cases
5. **Test ctx.waitUntil()**: Use `waitOnExecutionContext()` to verify async tasks complete

---

## Deployment Strategies

### All-at-Once Deployment (Default)

**Source**: https://developers.cloudflare.com/workers/platform/deployments/

```bash
# Immediate deployment - traffic switches instantly
npx wrangler deploy --env production
```

**Behavior**: "Traffic is immediately shifted from one version to the newly deployed version automatically"

**Use when**:
- Low-risk changes
- Confident in testing
- Small user base
- Can tolerate brief downtime if issues occur

### Gradual Deployments (Recommended for Production)

**Source**: https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments/

**Capabilities**:
- Gradually shift traffic between multiple versions
- Monitor performance metrics and exceptions across versions
- Execute rollbacks to stable versions if issues arise

### Gradual Deployment Workflow

**Step 1: Upload version without deploying**

```bash
# Create new version but don't deploy yet
npx wrangler versions upload
# Output: Created version abc123 (not deployed)
```

**Step 2: Deploy gradually**

```bash
# Interactive deployment with traffic split
npx wrangler versions deploy
```

Dashboard allows configuring traffic percentages:
```
Version abc123 (new): 10%
Version xyz789 (current): 90%
```

**Step 3: Monitor metrics**

Watch for:
- Error rates per version
- CPU time per version
- Request success rates
- Custom exceptions

**Step 4: Increase traffic or rollback**

```
# If metrics look good, increase new version traffic:
Version abc123: 50%
Version xyz789: 50%

# Then 100% when confident:
Version abc123: 100%
```

### Traffic Distribution Strategies

**Per-Request Distribution**:
- Each request has probabilistic chance of routing to each version
- No session affinity by default

**Version Affinity** (maintain consistent version per user):

```typescript
// Client sends consistent key (e.g., user ID)
fetch('https://api.example.com/data', {
  headers: {
    'Cloudflare-Workers-Version-Key': userId
  }
});
```

Users with same key always route to same version during gradual rollout.

**Version Overrides** (test specific version before general rollout):

```typescript
// Force specific version for testing
fetch('https://api.example.com/data', {
  headers: {
    'Cloudflare-Workers-Version-Overrides': 'abc123:100'
  }
});
```

### Zero-Downtime Deployment Checklist

- [ ] Upload new version with `wrangler versions upload`
- [ ] Start with 5-10% traffic to new version
- [ ] Monitor error rates for 15-30 minutes
- [ ] Gradually increase to 50% if stable
- [ ] Monitor for another 30 minutes
- [ ] Increase to 100% when confident
- [ ] Keep previous version available for 24h for quick rollback

### Rollback Procedures

```bash
# View recent versions
npx wrangler versions list

# View current deployment
npx wrangler deployments list

# Rollback to previous deployment
npx wrangler rollback

# Or rollback to specific version
npx wrangler versions deploy <version-id>
```

**Rollback speed**: Instant (traffic shifts immediately to previous version)

---

## Observability & Monitoring

### Workers Logs (Default)

**Source**: https://developers.cloudflare.com/workers/observability/logging/

**Enabled by default** - 24-hour retention

**Captured automatically**:
- `console.log()` output
- `console.error()` output
- `console.warn()` output
- Unhandled exceptions

**View logs**:

```bash
# Real-time log streaming
npx wrangler tail

# Real-time logs for specific environment
npx wrangler tail --env production

# Filter logs
npx wrangler tail --status error    # Only errors
```

### Logging Best Practices

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const startTime = Date.now();

    try {
      // Structured logging with context
      console.log(JSON.stringify({
        event: 'request_start',
        method: request.method,
        url: request.url,
        timestamp: new Date().toISOString()
      }));

      const response = await handleRequest(request, env);

      // Log success metrics (use ctx.waitUntil to not block response)
      ctx.waitUntil(
        logMetrics({
          event: 'request_complete',
          status: response.status,
          duration: Date.now() - startTime,
          timestamp: new Date().toISOString()
        })
      );

      return response;

    } catch (error) {
      // Log errors with full context
      console.error(JSON.stringify({
        event: 'request_error',
        error: error instanceof Error ? error.message : 'Unknown error',
        stack: error instanceof Error ? error.stack : undefined,
        timestamp: new Date().toISOString()
      }));

      return new Response('Internal Server Error', { status: 500 });
    }
  }
};
```

### Workers Logpush

**Source**: https://developers.cloudflare.com/workers/observability/logging/logpush/

**Export logs to external destinations**:
- AWS S3
- Google Cloud Storage
- Azure Blob Storage
- Datadog
- Splunk
- Custom HTTP endpoints

**Data included**:
- Request/response metadata
- Console messages
- Exceptions
- Performance metrics
- Custom fields

### Tail Workers (Beta)

**Real-time log processing and filtering**:

```typescript
// tail-worker.ts
export default {
  async tail(events: TraceItem[]): Promise<void> {
    for (const event of events) {
      // Filter for errors only
      if (event.outcome === 'exception' || event.outcome === 'exceededCpu') {
        // Send to monitoring service
        await fetch('https://monitoring.example.com/alerts', {
          method: 'POST',
          body: JSON.stringify({
            type: 'worker_error',
            outcome: event.outcome,
            scriptName: event.scriptName,
            logs: event.logs,
            timestamp: event.eventTimestamp
          })
        });
      }
    }
  }
};
```

### Custom Metrics with Analytics Engine

```typescript
interface Env {
  ANALYTICS: AnalyticsEngineDataset;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const startTime = Date.now();
    const response = await handleRequest(request);
    const duration = Date.now() - startTime;

    // Write custom metrics (non-blocking)
    ctx.waitUntil(
      env.ANALYTICS.writeDataPoint({
        blobs: [request.url, request.method],
        doubles: [duration, response.status],
        indexes: [response.ok ? 'success' : 'failure']
      })
    );

    return response;
  }
};
```

---

## Error Handling

### Production Error Handling Pattern

**Source**: https://developers.cloudflare.com/workers/observability/errors/

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    try {
      // Validate request
      if (!isValidRequest(request)) {
        return new Response('Bad Request', { status: 400 });
      }

      // Handle request
      const response = await handleRequest(request, env);

      return response;

    } catch (error) {
      // Log error with context
      console.error('Request failed:', {
        error: error instanceof Error ? error.message : 'Unknown error',
        stack: error instanceof Error ? error.stack : undefined,
        url: request.url,
        method: request.method,
        timestamp: new Date().toISOString()
      });

      // Send error to monitoring service (non-blocking)
      ctx.waitUntil(
        reportError(error, request, env)
      );

      // Return graceful error response
      return new Response(
        JSON.stringify({
          error: 'Internal Server Error',
          requestId: crypto.randomUUID()
        }),
        {
          status: 500,
          headers: { 'Content-Type': 'application/json' }
        }
      );
    }
  }
};

async function reportError(error: unknown, request: Request, env: Env): Promise<void> {
  // Send to Sentry, Datadog, or custom monitoring
  await fetch('https://monitoring.example.com/errors', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: error instanceof Error ? error.message : 'Unknown error',
      stack: error instanceof Error ? error.stack : undefined,
      url: request.url,
      timestamp: Date.now()
    })
  });
}
```

### Graceful Degradation with Pass-Through

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Enable pass-through on exception (fail open)
    ctx.passThroughOnException();

    // If unhandled exception occurs, request passes to origin
    const data = await env.DB.prepare('SELECT * FROM users').all();

    return Response.json(data);
  }
};
```

**Use when**: You want requests to reach origin server if Worker fails

**Limitation**: Cannot use if request body already consumed

### Common Error Patterns to Avoid

**❌ Floating Promises** (unhandled rejections):
```typescript
// BAD - promise not awaited or caught
env.QUEUE.send(message); // Fails silently if error occurs
```

**✅ Fix**:
```typescript
// Use ctx.waitUntil() for non-blocking async work
ctx.waitUntil(env.QUEUE.send(message));

// Or await it
await env.QUEUE.send(message);
```

**❌ Caching Response objects**:
```typescript
// BAD - Response objects cannot be reused
const response = await fetch(url);
globalThis.cachedResponse = response; // ERROR
```

**✅ Fix**:
```typescript
// Clone response before caching
const response = await fetch(url);
const clone = response.clone();
await cache.put(url, clone);
return response;
```

---

## Cost Optimization

### Understanding Pricing

**Source**: https://developers.cloudflare.com/workers/platform/pricing/

**Request Charges**:
- Standard plan: 10M requests/month included
- $0.30 per additional million requests
- Static assets through Workers = no request charges

**CPU Time Charges**:
- 30M CPU milliseconds/month included
- $0.02 per million CPU ms after that

**Key Insight**: "Cloudflare Workers runs before the Cloudflare cache, the caching of a request still incurs costs"

### Optimization Strategies

**1. Set CPU Time Limits**

```toml
# wrangler.toml
[limits]
cpu_ms = 50  # Fail fast if Worker exceeds 50ms CPU time
```

Prevents runaway costs from inefficient code.

**2. Use Service Bindings (No Request Fees)**

```typescript
// ❌ Costs 2 requests
const response = await fetch('https://my-other-worker.example.com/api');

// ✅ Costs 1 request (combined CPU time)
const response = await env.OTHER_WORKER.fetch(request);
```

**Service Bindings charge**:
- One request charge total
- Combined CPU time of both Workers

**3. Optimize Database Queries**

```typescript
// ❌ Expensive - full table scan (charges per row read)
const { results } = await env.DB.prepare(
  'SELECT * FROM users WHERE name = ?'  // Missing index on 'name'
).bind('John').all();

// ✅ Optimized - uses index
await env.DB.exec('CREATE INDEX idx_users_name ON users(name)');
const { results } = await env.DB.prepare(
  'SELECT * FROM users WHERE name = ?'  // Uses index
).bind('John').all();
```

**D1 charges per rows read/written**, not per query count.

**4. Cache Aggressively**

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Check cache first (free)
    const cache = caches.default;
    let response = await cache.match(request);

    if (response) {
      return response; // Cached response = no backend costs
    }

    // Generate response
    response = await generateResponse(request, env);

    // Cache async (non-blocking)
    ctx.waitUntil(cache.put(request, response.clone()));

    return response;
  }
};
```

**5. Monitor Usage**

```bash
# View usage metrics
npx wrangler metrics

# Check current billing
# Dashboard: Workers & Pages > <Your Worker> > Metrics
```

---

## Platform Limits & Constraints

### CPU Time Limits

**Source**: https://developers.cloudflare.com/workers/platform/limits/

- **Free**: 10ms CPU time per request
- **Paid**: 5 minutes (300s) for HTTP requests
- **Cron Triggers (Paid)**: 15 minutes

**Important**: "Most Workers requests consume less than 1-2 milliseconds of CPU time"

**CPU Time vs Duration**:
- CPU time = actual processor activity
- Duration = wall-clock time (includes waiting on I/O)
- **No charge or limit on duration**

### Startup Time Limit

**Worker must parse and execute global scope within 1 second**

```typescript
// ❌ BAD - Expensive global scope initialization
const largeModel = await loadMLModel(); // Slow startup

export default {
  async fetch(request: Request): Promise<Response> {
    return new Response('OK');
  }
};

// ✅ GOOD - Lazy initialization
let modelCache: Model | null = null;

export default {
  async fetch(request: Request): Promise<Response> {
    if (!modelCache) {
      modelCache = await loadMLModel(); // Load on first request
    }
    return new Response('OK');
  }
};
```

### Memory Limit

**128 MB per isolate**

System gracefully handles overages:
- Allows in-flight requests to complete
- Creates new isolates for subsequent requests

### Request/Response Size Limits

**Requests**:
- URL: 16 KB maximum
- Headers: 128 KB combined total
- Body: 100 MB – 500 MB (depends on Cloudflare plan)

**Responses**:
- Headers: 128 KB combined total
- Body: No hard limit by Workers (CDN cache limits: 512 MB standard, 5 GB Enterprise)

### Subrequest Limits

- **Free**: 50 subrequests per request
- **Paid**: 1,000 subrequests per request

Includes:
- `fetch()` calls
- Cache API operations
- Service Binding calls

### Bundle Size Limits

- **Free**: 3 MB compressed
- **Paid**: 10 MB compressed
- **Uncompressed** (both): 64 MB

### Storage Limits

**KV**:
- Key size: 512 bytes max
- Value size: 25 MiB max
- Write rate: 1 write/second/key
- Consistency: Eventually consistent (~60s global propagation)

**D1**:
- Database size: 10 GB maximum (cannot be increased)
- Query timeout: 30 seconds
- Queries per invocation: 50 (Free) / 1,000 (Paid)
- Recommendation: Scale horizontally with multiple databases

**Durable Objects**:
- Storage per object: No hard limit
- Consistent storage API (strongly consistent)

### Known Issues

**Source**: https://developers.cloudflare.com/workers/platform/known-issues/

1. **Route Specificity**: Trailing wildcards (`/*`) don't work as expected with multiple similar patterns
2. **wrangler dev Remote Mode**: `cf-workers-preview-token` header causes requests to other Cloudflare zones to be discarded
3. **DNS Resolution**: All hostnames require dedicated DNS entry in Cloudflare's DNS
4. **IP Address Restrictions**: Cannot fetch directly to IP addresses (use DNS A/AAAA record instead)

---

## Summary: Production Deployment Checklist

### Pre-Deployment

- [ ] Environment-specific configuration in wrangler.toml
- [ ] Secrets configured for each environment
- [ ] Tests passing (unit + integration)
- [ ] Bundle size within limits
- [ ] CPU time optimized (< 50ms for most requests)
- [ ] Error handling implemented
- [ ] Logging configured

### Deployment

- [ ] Upload version with `wrangler versions upload`
- [ ] Start gradual rollout (10% traffic)
- [ ] Monitor logs and metrics for 15-30 min
- [ ] Increase to 50% if stable
- [ ] Monitor for another 30 min
- [ ] Deploy to 100% when confident
- [ ] Keep previous version available for rollback

### Post-Deployment

- [ ] Monitor error rates
- [ ] Check CPU time metrics
- [ ] Verify request success rates
- [ ] Review cost impact
- [ ] Document any issues
- [ ] Update runbooks if needed

---

## See Also

- **Security**: `security-patterns.md` - Authentication, validation, headers
- **Migration**: `migration-guides.md` - Server to Workers migration
- **Best Practices**: `workers-best-practices.md` - Code standards
- **Examples**: `workers-examples.md` - Complete implementations
- **Official Docs**: https://developers.cloudflare.com/workers/
