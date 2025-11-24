# Cloudflare Workers Development Lifecycle

Complete end-to-end guide to developing, testing, deploying, and operating Cloudflare Workers applications.

## Overview

The Cloudflare Workers development lifecycle consists of four main phases:

1. **Development** - Local iteration with `wrangler dev`
2. **Testing** - Vitest integration with Workers runtime
3. **Deployment** - Environment-based or gradual rollouts
4. **Operations** - Monitoring, incident response, maintenance

**Quick Workflow:**
```
Create → Develop Locally → Test → Deploy to Staging → Deploy to Production → Monitor
```

---

## Development Phase

### Project Initialization

```bash
# Interactive setup (recommended)
npm create cloudflare@latest -- my-worker

# Manual initialization
npx wrangler init my-worker
cd my-worker
```

### Local Development

```bash
# Start development server
npx wrangler dev                    # Local mode (isolated)
npx wrangler dev --remote           # Remote mode (actual bindings)
npx wrangler dev --env staging      # Specific environment

# Server runs at http://localhost:8787
# Hot reload on file save
```

**When to use each mode:**
- **Local mode**: Fast iteration, pure code changes, no costs
- **Remote mode**: Testing with actual data, production-like behavior

### Creating Resources

```bash
# D1 database
npx wrangler d1 create my-database

# KV namespace
npx wrangler kv namespace create SESSIONS

# R2 bucket
npx wrangler r2 bucket create uploads

# Queue
npx wrangler queues create my-queue
```

### Database Migrations

```bash
# Create migration
npx wrangler d1 migrations create my-db create_users_table

# Apply locally for testing
npx wrangler d1 migrations apply my-db --local

# Apply to production
npx wrangler d1 migrations apply my-db --remote
```

---

## Testing Phase

### Setup Vitest

```bash
npm install --save-dev vitest@~3.2.0 @cloudflare/vitest-pool-workers
```

```typescript
// vitest.config.ts
import { defineWorkersConfig } from '@cloudflare/vitest-pool-workers/config';

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: './wrangler.jsonc' }
      }
    }
  }
});
```

### Unit Tests

```typescript
import { describe, it, expect } from 'vitest';
import { env, createExecutionContext, waitOnExecutionContext } from 'cloudflare:test';
import worker from '../src/index';

describe('Worker', () => {
  it('responds with 200', async () => {
    const request = new Request('https://example.com/');
    const ctx = createExecutionContext();

    const response = await worker.fetch(request, env, ctx);
    await waitOnExecutionContext(ctx);

    expect(response.status).toBe(200);
  });

  it('accesses KV binding', async () => {
    await env.SESSIONS.put('key', 'value');
    const value = await env.SESSIONS.get('key');
    expect(value).toBe('value');
  });
});
```

### Running Tests

```bash
npm test                    # Run all tests
npm test -- --watch         # Watch mode
npm test -- --coverage      # With coverage
```

---

## Deployment Phase

### Environment-Based Deployment

```bash
# Deploy to specific environment
npx wrangler deploy --env dev
npx wrangler deploy --env staging
npx wrangler deploy --env production
```

### Gradual Deployment (Production)

```bash
# Step 1: Upload version without deploying
npx wrangler versions upload --env production

# Step 2: Deploy with traffic split
npx wrangler versions deploy --env production
# Configure: 10% new version, 90% old version

# Step 3: Monitor and gradually increase
# 10% → 25% → 50% → 100%

# Rollback if needed
npx wrangler rollback --env production
```

### Deployment Workflow

```
1. Apply database migrations first
2. Set secrets if changed
3. Deploy code
4. Smoke test
5. Monitor logs and metrics
6. Rollback if issues detected
```

---

## Production Operations

### Monitoring

```bash
# Real-time log streaming
npx wrangler tail --env production

# Filter errors only
npx wrangler tail --env production --status error

# JSON format for parsing
npx wrangler tail --env production --format json
```

### Incident Response

```
1. Detect issue (alerts, logs, reports)
2. Assess severity (P0/P1/P2)
3. Mitigate:
   - P0/P1: Immediate rollback
   - P2: Reduce traffic to new version
4. Investigate root cause
5. Fix and redeploy
```

### Maintenance

```bash
# Rotate secrets
echo "$NEW_SECRET" | npx wrangler secret put API_KEY --env production

# Update dependencies
npm update
npm test
npx wrangler deploy --env staging  # Test first

# Database migrations
npx wrangler d1 migrations create my-db add_column
npx wrangler d1 migrations apply my-db --remote
```

---

## Workflows by Team Size

### Solo Developer (2 environments)

```
Local Development → Production
├─ wrangler dev (local testing)
├─ npm test
├─ git commit
└─ wrangler deploy
```

**Characteristics:**
- Simple, fast iteration
- Git for version control and rollback
- Focus on critical path testing

### Team (4 environments)

```
Local → Dev → Staging → Production
├─ Feature branches
├─ Code reviews
├─ CI/CD automation
└─ Gradual production rollouts
```

**CI/CD Pipeline:**
- Tests on every commit
- Auto-deploy to dev on merge
- Manual promotion to staging
- Gradual rollout to production

### Enterprise (5+ environments)

```
Local → Dev → QA → Staging → Production (+ DR)
├─ Formal change management
├─ Security scanning
├─ Compliance gates
├─ Multi-team coordination
└─ 24-48h gradual rollouts
```

---

## Command Quick Reference

**Development:**
```bash
npm create cloudflare@latest           # Create project
npx wrangler dev                       # Local development
npx wrangler dev --remote              # Remote development
npx wrangler d1 create my-db           # Create D1 database
npx wrangler kv namespace create KV    # Create KV namespace
```

**Testing:**
```bash
npm test                               # Run tests
npm test -- --watch                    # Watch mode
npm test -- --coverage                 # With coverage
```

**Deployment:**
```bash
npx wrangler deploy                    # Traditional deployment
npx wrangler deploy --env staging      # Environment-specific
npx wrangler versions upload           # Create version
npx wrangler versions deploy           # Gradual deployment
npx wrangler rollback                  # Rollback to previous
```

**Operations:**
```bash
npx wrangler tail                      # Stream logs
npx wrangler tail --status error       # Filter errors
npx wrangler secret put SECRET_NAME    # Add secret
npx wrangler deployments list          # View deployments
npx wrangler versions list             # View versions
```

---

## Decision Matrix

| Decision | Option A | Option B |
|----------|----------|----------|
| **Development Mode** | Local (fast, isolated) | Remote (real data) |
| **Testing** | Unit tests (business logic) | Integration tests (E2E) |
| **Deployment** | Traditional (low-risk) | Gradual (production) |
| **Environments** | 2 (local, prod) - Solo | 4+ (local, dev, staging, prod) - Team |
| **Monitoring** | wrangler tail (debugging) | Full observability (production) |

---

## Best Practices

**Development:**
- Use TypeScript for type safety
- Keep bundle size < 1 MB
- Test locally first, then remote
- Lazy-load expensive resources

**Testing:**
- Test critical business logic
- Use real bindings in tests
- Run tests in CI/CD
- Aim for >80% coverage on critical paths

**Deployment:**
- Apply migrations before code
- Use gradual rollouts for production
- Monitor each traffic increase
- Keep previous versions for rollback

**Operations:**
- Structure logs as JSON
- Set up alerts for error rates
- Document incident response
- Practice rollback procedures

---

## Common Patterns

### Structured Logging

```typescript
// ✅ Good
console.log(JSON.stringify({
  event: 'request',
  method: request.method,
  status: response.status,
  timestamp: Date.now()
}));

// ❌ Bad
console.log('Request: ' + request.method);
```

### Non-Blocking Async Work

```typescript
// ✅ Good - Don't block response
ctx.waitUntil(env.ANALYTICS.writeDataPoint(metrics));
return response;

// ❌ Bad - Blocks response
await env.ANALYTICS.writeDataPoint(metrics);
return response;
```

### Avoid Global State

```typescript
// ❌ Bad - Unreliable across requests
let counter = 0;
export default {
  fetch() { counter++; }
};

// ✅ Good - Use Durable Objects
const id = env.COUNTER.idFromName('global');
const stub = env.COUNTER.get(id);
await stub.increment();
```

---

## CI/CD Integration Example

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          command: deploy --env production
```

---

## When to Evolve

**Solo → Team** when:
- Adding second developer
- Customer revenue at stake
- Need formal change tracking

**Team → Enterprise** when:
- 10+ developers
- Critical SLA requirements
- Regulatory compliance (SOC2, HIPAA)
- Multi-service architecture

---

## See Also

- **Deployment Workflows**: `deployment-workflows.md` - Environment management, gradual deployments
- **Security Patterns**: `security-patterns.md` - Authentication, encryption, CORS
- **Migration Guides**: `migration-guides.md` - Server to Workers migration
- **Best Practices**: `workers-best-practices.md` - Code standards
- **Examples**: `workers-examples.md` - Complete implementations
- **Official Docs**: https://developers.cloudflare.com/workers/
