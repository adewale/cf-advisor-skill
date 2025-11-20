# Wrangler CLI Reference

Quick reference for common wrangler commands and workflows for Cloudflare Workers development.

## Table of Contents

1. [Project Initialization](#project-initialization)
2. [Local Development](#local-development)
3. [Deployment](#deployment)
4. [Workers Commands](#workers-commands)
5. [Pages Commands](#pages-commands)
6. [D1 Commands](#d1-commands)
7. [KV Commands](#kv-commands)
8. [R2 Commands](#r2-commands)
9. [Durable Objects](#durable-objects)
10. [Queues Commands](#queues-commands)
11. [Secrets Management](#secrets-management)
12. [Logs and Debugging](#logs-and-debugging)
13. [Configuration](#configuration)

---

## Project Initialization

### Create New Worker

```bash
# Interactive initialization
npx wrangler init

# Create Worker with specific name
npx wrangler init my-worker

# Create Worker with TypeScript
npx wrangler init my-worker --type typescript
```

### Create New Pages Project

```bash
# Initialize Pages project
npx wrangler pages project create my-pages-project
```

---

## Local Development

### Run Worker Locally

```bash
# Start local development server
npx wrangler dev

# Dev with specific port
npx wrangler dev --port 8787

# Dev with local bindings
npx wrangler dev --local

# Dev with remote bindings
npx wrangler dev --remote
```

### Run Pages Locally

```bash
# Run Pages site locally
npx wrangler pages dev

# Run from specific directory
npx wrangler pages dev ./dist

# Run with specific port
npx wrangler pages dev ./dist --port 8788
```

---

## Deployment

### Deploy Worker

```bash
# Deploy to production
npx wrangler deploy

# Deploy specific file
npx wrangler deploy src/index.ts

# Deploy with environment
npx wrangler deploy --env staging

# Dry run (validate without deploying)
npx wrangler deploy --dry-run
```

### Deploy Pages

```bash
# Deploy Pages site
npx wrangler pages deploy ./dist

# Deploy with specific project name
npx wrangler pages deploy ./dist --project-name my-site

# Deploy to specific branch
npx wrangler pages deploy ./dist --branch preview
```

---

## Workers Commands

### Worker Management

```bash
# List all Workers
npx wrangler list

# Delete Worker
npx wrangler delete my-worker

# Get Worker details
npx wrangler status
```

---

## Pages Commands

### Pages Project Management

```bash
# List Pages projects
npx wrangler pages project list

# Create new project
npx wrangler pages project create my-project

# Delete project
npx wrangler pages project delete my-project
```

### Pages Deployments

```bash
# List deployments
npx wrangler pages deployment list

# Get deployment details
npx wrangler pages deployment list --project-name my-project
```

---

## D1 Commands

### Database Management

```bash
# Create new D1 database
npx wrangler d1 create my-database

# List databases
npx wrangler d1 list

# Delete database
npx wrangler d1 delete my-database
```

### Execute SQL

```bash
# Run SQL command (remote)
npx wrangler d1 execute my-db --command "SELECT * FROM users"

# Run SQL command (local)
npx wrangler d1 execute my-db --local --command "SELECT * FROM users"

# Run SQL from file
npx wrangler d1 execute my-db --file ./schema.sql

# Run with JSON output
npx wrangler d1 execute my-db --command "SELECT * FROM users" --json
```

### Migrations

```bash
# Create migration
npx wrangler d1 migrations create my-db add_users_table

# List migrations
npx wrangler d1 migrations list my-db

# Apply migrations (local)
npx wrangler d1 migrations apply my-db --local

# Apply migrations (remote)
npx wrangler d1 migrations apply my-db --remote
```

---

## KV Commands

### Namespace Management

```bash
# Create KV namespace
npx wrangler kv namespace create MY_KV

# Create preview namespace
npx wrangler kv namespace create MY_KV --preview

# List namespaces
npx wrangler kv namespace list

# Delete namespace
npx wrangler kv namespace delete --namespace-id=xxxxx
```

### Key Operations

```bash
# Write key
npx wrangler kv key put --binding=MY_KV "my-key" "my-value"

# Write key from file
npx wrangler kv key put --binding=MY_KV "my-key" --path ./value.txt

# Read key
npx wrangler kv key get --binding=MY_KV "my-key"

# Delete key
npx wrangler kv key delete --binding=MY_KV "my-key"

# List keys
npx wrangler kv key list --binding=MY_KV

# List keys with prefix
npx wrangler kv key list --binding=MY_KV --prefix "user:"
```

### Bulk Operations

```bash
# Bulk upload from JSON
npx wrangler kv bulk put --binding=MY_KV ./data.json

# Bulk delete from JSON
npx wrangler kv bulk delete --binding=MY_KV ./keys.json
```

---

## R2 Commands

### Bucket Management

```bash
# Create R2 bucket
npx wrangler r2 bucket create my-bucket

# List buckets
npx wrangler r2 bucket list

# Delete bucket
npx wrangler r2 bucket delete my-bucket
```

### Object Operations

```bash
# Upload object
npx wrangler r2 object put my-bucket/file.txt --file ./local-file.txt

# Download object
npx wrangler r2 object get my-bucket/file.txt --file ./downloaded-file.txt

# Delete object
npx wrangler r2 object delete my-bucket/file.txt

# List objects
npx wrangler r2 object list my-bucket

# List objects with prefix
npx wrangler r2 object list my-bucket --prefix "uploads/"
```

---

## Durable Objects

Durable Objects are configured in wrangler.jsonc/toml - no separate CLI commands needed.

### Configuration Example

```jsonc
// wrangler.jsonc
{
  "durable_objects": {
    "bindings": [
      {
        "name": "COUNTER",
        "class_name": "Counter",
        "script_name": "my-worker"
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

---

## Queues Commands

### Queue Management

```bash
# Create queue
npx wrangler queues create my-queue

# List queues
npx wrangler queues list

# Delete queue
npx wrangler queues delete my-queue
```

### Queue Operations

```bash
# Send message to queue (for testing)
npx wrangler queues send my-queue --body '{"test": "message"}'
```

---

## Secrets Management

### Secret Operations

```bash
# Set secret
npx wrangler secret put SECRET_NAME

# Set secret from file
echo "my-secret-value" | npx wrangler secret put SECRET_NAME

# List secrets (shows names only, not values)
npx wrangler secret list

# Delete secret
npx wrangler secret delete SECRET_NAME
```

### Environment-Specific Secrets

```bash
# Set secret for specific environment
npx wrangler secret put SECRET_NAME --env production
```

---

## Logs and Debugging

### Tail Logs

```bash
# Tail production logs
npx wrangler tail

# Tail with specific Worker
npx wrangler tail my-worker

# Tail with environment
npx wrangler tail --env production

# Tail with filters
npx wrangler tail --status error

# Tail with output format
npx wrangler tail --format json
```

### Inspect Logs

```bash
# View recent logs (if Workers Logs enabled)
npx wrangler logs my-worker

# View logs with time range
npx wrangler logs my-worker --start "2024-01-01" --end "2024-01-02"
```

---

## Configuration

### Configuration File Example

```jsonc
// wrangler.jsonc
{
  "name": "my-app",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-07",
  "compatibility_flags": ["nodejs_compat"],

  // Observability
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1
  },

  // KV Namespaces
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "preview_id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  ],

  // D1 Databases
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database",
      "database_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  ],

  // R2 Buckets
  "r2_buckets": [
    {
      "binding": "MY_BUCKET",
      "bucket_name": "my-bucket"
    }
  ],

  // Durable Objects
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
  ],

  // Queues
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
  },

  // Environment Variables
  "vars": {
    "API_URL": "https://api.example.com"
  }
}
```

### Environment Configuration

```jsonc
// wrangler.jsonc with environments
{
  "name": "my-app",
  "main": "src/index.ts",

  "env": {
    "staging": {
      "name": "my-app-staging",
      "vars": {
        "API_URL": "https://staging.api.example.com"
      }
    },
    "production": {
      "name": "my-app-production",
      "vars": {
        "API_URL": "https://api.example.com"
      }
    }
  }
}
```

---

## Common Workflows

### Complete Setup Workflow

```bash
# 1. Initialize project
npx wrangler init my-app

# 2. Create D1 database
npx wrangler d1 create my-database

# 3. Create KV namespace
npx wrangler kv namespace create MY_KV

# 4. Create R2 bucket
npx wrangler r2 bucket create my-bucket

# 5. Update wrangler.jsonc with bindings

# 6. Test locally
npx wrangler dev --local

# 7. Run migrations
npx wrangler d1 migrations apply my-database --remote

# 8. Deploy
npx wrangler deploy
```

### Development Workflow

```bash
# 1. Start local dev
npx wrangler dev

# 2. Make changes (auto-reloads)

# 3. Test with local data
npx wrangler d1 execute my-db --local --command "SELECT * FROM users"

# 4. Deploy when ready
npx wrangler deploy

# 5. Tail production logs
npx wrangler tail
```

---

## Tips and Best Practices

### Global Installation (Optional)

```bash
# Install globally for shorter commands
npm install -g wrangler

# Then use without npx
wrangler dev
wrangler deploy
```

### Using with Package.json Scripts

```json
{
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "deploy:staging": "wrangler deploy --env staging",
    "tail": "wrangler tail",
    "db:migrate": "wrangler d1 migrations apply my-database --remote"
  }
}
```

Then run:
```bash
npm run dev
npm run deploy
npm run tail
```

### Configuration Tips

1. **Always set compatibility_date** to latest or current date
2. **Enable observability** for production debugging
3. **Use environment variables** for configuration, not secrets
4. **Use secrets** for API keys and credentials
5. **Keep wrangler.jsonc** in version control (without IDs if sensitive)
6. **Use .dev.vars** for local development secrets (gitignored)

---

## Getting Help

```bash
# Get general help
npx wrangler --help

# Get command-specific help
npx wrangler deploy --help
npx wrangler d1 --help
npx wrangler kv --help

# Check wrangler version
npx wrangler --version
```

---

## See Also

- **Best Practices**: Read `workers-best-practices.md` for code standards
- **Integration Details**: Read `workers-integrations.md` for binding usage
- **Complete Examples**: Read `workers-examples.md` for full implementations
- **Official Docs**: https://developers.cloudflare.com/workers/wrangler/
