# Cloudflare Primitives Catalog

This catalog lists all Cloudflare developer primitives with their key characteristics, use cases, and composition patterns.

## Compute Layer

### Workers
**Category**: Compute  
**What it does**: Stateless JavaScript/TypeScript request handlers that run at the edge

**Use when**:
- Building APIs
- Proxying and routing requests
- Implementing middleware logic
- Edge-side logic and caching

**Binds with**: All other primitives (KV, D1, R2, DO, Queues, AI Gateway, etc.)

**Key limits**:
- CPU time: 50ms (Free), 30s (Paid)
- Memory: 128MB
- Subrequest limit varies by plan

**Pricing**: Pay per request + CPU time

**Links**:
- [Workers Docs](https://developers.cloudflare.com/workers/)
- [Get Started](https://developers.cloudflare.com/workers/get-started/)

---

### Durable Objects
**Category**: Compute + Storage  
**What it does**: Stateful coordinators with SQLite storage, single-threaded execution

**Use when**:
- WebSocket connections
- Real-time collaboration
- Coordination across requests
- Per-entity state (e.g., per-room, per-user)
- Rate limiting

**Binds with**: Workers, KV, R2, Queues

**Key limits**:
- Requests: Unlimited (single-threaded per instance)
- Storage: 1GB per object
- CPU time: Same as Workers

**Pricing**: Requests + duration + storage

**Links**:
- [Durable Objects Docs](https://developers.cloudflare.com/durable-objects/)
- [Get Started](https://developers.cloudflare.com/durable-objects/get-started/)

---

### Pages Functions
**Category**: Compute  
**What it does**: File-based routing Workers for Pages projects

**Use when**:
- Building full-stack applications
- Creating API routes alongside static frontend
- Need file-based routing

**Binds with**: Same as Workers (KV, D1, R2, DO, etc.)

**Key limits**: Same as Workers

**Pricing**: Included with Pages

**Links**:
- [Pages Functions Docs](https://developers.cloudflare.com/pages/functions/)
- [Get Started](https://developers.cloudflare.com/pages/functions/get-started/)

---

### Containers (Beta)
**Category**: Compute  
**What it does**: Long-running containerized processes

**Use when**:
- Need long-running processes
- Using existing Docker containers
- Stateful applications

**Binds with**: Workers can route to containers

**Key limits**: Beta - check current docs

**Pricing**: Beta pricing

**Links**:
- [Containers Docs](https://developers.cloudflare.com/containers/)

---

## Storage Layer

### KV (Workers KV)
**Category**: Storage  
**What it does**: Eventually consistent, globally replicated key-value store

**Use when**:
- Session storage
- Configuration and feature flags
- Caching API responses
- Global read performance matters
- Eventual consistency acceptable

**Binds with**: Workers, Pages Functions, Durable Objects

**Key limits**:
- Key size: 512 bytes
- Value size: 25MB
- Operations per day varies by plan
- Eventually consistent (writes take ~60s to propagate globally)

**Pricing**: Storage + operations (reads/writes)

**Links**:
- [KV Docs](https://developers.cloudflare.com/kv/)
- [Get Started](https://developers.cloudflare.com/kv/get-started/)

---

### D1
**Category**: Storage  
**What it does**: Regional SQLite database with read replication

**Use when**:
- Need SQL queries and relations
- Consistency required
- Structured relational data

**Binds with**: Workers, Pages Functions, Hyperdrive

**Key limits**:
- Database size: 10GB (Beta)
- Strong consistency within region
- Read replicas for global reads

**Pricing**: Storage + reads + writes

**Links**:
- [D1 Docs](https://developers.cloudflare.com/d1/)
- [Get Started](https://developers.cloudflare.com/d1/get-started/)

---

### R2
**Category**: Storage
**What it does**: S3-compatible object storage

**Use when**:
- Storing large files
- User uploads (images, videos, documents)
- Backups and archives
- Static assets

**Binds with**: Workers, Pages Functions, Images

**Key limits**:
- Object size: 5TB max
- S3-compatible API

**Pricing**: Storage + Class A/B operations (no egress fees)

**Links**:
- [R2 Docs](https://developers.cloudflare.com/r2/)
- [Get Started](https://developers.cloudflare.com/r2/get-started/)

---

### R2 Data Catalog
**Category**: Storage/Data
**What it does**: Apache Iceberg catalog for R2 object storage

**Use when**:
- Building data lakes on R2
- Managing large-scale analytics datasets
- Table versioning and schema evolution
- ACID transactions on object storage

**Key characteristics**:
- Apache Iceberg compatibility
- Table management and metadata
- Schema evolution
- Time travel queries
- Works with data processing frameworks (Spark, Flink, etc.)

**Binds with**: R2, Workers, external data processing tools

**Status**: New product

**Pricing**: Based on API operations

**Links**:
- [R2 Data Catalog Docs](https://developers.cloudflare.com/r2/data-catalog/)

---

### R2 SQL
**Category**: Storage/Data
**What it does**: Distributed SQL query engine over R2 data

**Use when**:
- Running SQL queries on data in R2
- Analytics and reporting on object storage
- Ad-hoc data analysis
- BI tool integration

**Key characteristics**:
- SQL interface to R2 data
- Supports Parquet, CSV, JSON formats
- Distributed query execution
- No data movement required
- Works with Apache Iceberg tables

**Binds with**: R2, R2 Data Catalog, Workers

**Status**: New product

**Pricing**: Query-based pricing

**Links**:
- [R2 SQL Docs](https://developers.cloudflare.com/r2/sql/)

---

### Pipelines
**Category**: Storage/Data
**What it does**: Real-time data streaming to R2

**Use when**:
- ETL pipelines to R2
- Real-time data ingestion
- Event streaming to storage
- Log aggregation to object storage

**Key characteristics**:
- Real-time streaming to R2
- Automatic batching and compression
- Schema management
- Exactly-once delivery semantics
- Parquet output format

**Binds with**: R2, Workers, Kafka, event streams

**Status**: New product

**Pricing**: Ingestion-based pricing

**Links**:
- [Pipelines Docs](https://developers.cloudflare.com/pipelines/)

---

### Durable Object Storage
**Category**: Storage  
**What it does**: Per-object SQLite storage tied to Durable Objects

**Use when**:
- State specific to a Durable Object
- Transactional consistency needed
- Coordination state

**Binds with**: Only accessible from Durable Objects

**Key limits**:
- 1GB per Durable Object
- SQLite API

**Pricing**: Included with Durable Objects

**Links**:
- [DO Storage API](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)

---

## Network Layer

### Hyperdrive
**Category**: Network  
**What it does**: Connection pooling and query caching for external databases

**Use when**:
- Connecting to existing PostgreSQL or MySQL
- Need connection pooling (avoid connection exhaustion)
- Want query caching
- Gradual migration from external DB

**Binds with**: Workers, Pages Functions

**Supported databases**: PostgreSQL, MySQL, CockroachDB, Timescale, Neon, Supabase, etc.

**Key limits**:
- Max connections per database
- Query cache TTL configurable

**Pricing**: Connections + queries

**Links**:
- [Hyperdrive Docs](https://developers.cloudflare.com/hyperdrive/)
- [Get Started](https://developers.cloudflare.com/hyperdrive/get-started/)

---

### Queues
**Category**: Network  
**What it does**: Asynchronous message queuing between Workers

**Use when**:
- Background processing
- Batch operations
- Decoupling producers from consumers
- Retry logic needed

**Binds with**: Workers (producers and consumers)

**Key limits**:
- Message size: 128KB
- Retention: 4 days
- Batch processing support

**Pricing**: Operations (enqueues + dequeues)

**Links**:
- [Queues Docs](https://developers.cloudflare.com/queues/)
- [Get Started](https://developers.cloudflare.com/queues/get-started/)

---

### Pub/Sub
**Category**: Network  
**What it does**: MQTT-based message streaming

**Use when**:
- Real-time messaging
- IoT applications
- Event streaming

**Binds with**: Workers can publish/subscribe

**Key limits**: Check current docs

**Pricing**: Messages

**Links**:
- [Pub/Sub Docs](https://developers.cloudflare.com/pub-sub/)

---

### Email Routing
**Category**: Network  
**What it does**: Route and process emails

**Use when**:
- Custom email handling
- Email forwarding
- Processing incoming emails

**Binds with**: Workers (Email Workers)

**Key limits**: Check current docs

**Pricing**: Free tier available

**Links**:
- [Email Routing Docs](https://developers.cloudflare.com/email-routing/)

---

## Intelligence Layer

### Workers AI
**Category**: Intelligence  
**What it does**: Run AI models at the edge (LLMs, embeddings, image gen)

**Use when**:
- Need on-edge AI inference
- Low-latency AI responses
- Using Cloudflare-hosted models

**Binds with**: Workers, Pages Functions

**Available models**: Llama, Mistral, SDXL, embeddings, etc.

**Key limits**:
- Model-specific limits
- Inference time limits

**Pricing**: Per inference

**Links**:
- [Workers AI Docs](https://developers.cloudflare.com/workers-ai/)

---

### AI Gateway
**Category**: Intelligence  
**What it does**: Proxy, cache, and monitor calls to external AI APIs

**Use when**:
- Using external AI providers (OpenAI, Anthropic, etc.)
- Want caching for AI responses
- Need observability and rate limiting
- Cost control for AI APIs

**Binds with**: Workers, Pages Functions

**Supported providers**: OpenAI, Anthropic, Cohere, HuggingFace, Azure OpenAI, etc.

**Key features**:
- Response caching
- Request/response logging
- Rate limiting
- Cost tracking

**Pricing**: Free (monitoring), pay for underlying AI provider

**Links**:
- [AI Gateway Docs](https://developers.cloudflare.com/ai-gateway/)
- [Get Started](https://developers.cloudflare.com/ai-gateway/get-started/)

---

### Agents
**Category**: Intelligence  
**What it does**: Build and deploy agentic workflows with tool calling

**Use when**:
- Building AI agents
- Implementing MCP servers
- Agentic workflows

**Binds with**: Workers, AI Gateway, KV, D1, R2

**Key features**:
- Tool calling
- MCP protocol support
- Human-in-the-loop patterns

**Pricing**: Based on underlying services

**Links**:
- [Agents Docs](https://developers.cloudflare.com/agents/)

---

### Browser Rendering
**Category**: Intelligence  
**What it does**: Puppeteer/Playwright API for headless browser automation

**Use when**:
- Taking screenshots
- Generating PDFs
- Web scraping
- Testing web applications

**Binds with**: Workers, Durable Objects (for session management)

**Key limits**:
- Session duration
- Concurrent sessions

**Pricing**: Per session

**Links**:
- [Browser Rendering Docs](https://developers.cloudflare.com/browser-rendering/)
- [Get Started](https://developers.cloudflare.com/browser-rendering/get-started/)

---

### Vectorize
**Category**: Intelligence
**What it does**: Vector database for embeddings

**Use when**:
- Semantic search
- RAG (Retrieval Augmented Generation)
- Similarity search

**Binds with**: Workers, Workers AI (for embeddings)

**Key limits**: Check current docs

**Pricing**: Storage + queries

**Links**:
- [Vectorize Docs](https://developers.cloudflare.com/vectorize/)

---

### AI Search
**Category**: Intelligence/Search
**What it does**: Managed search service that creates continuously updating indexes with natural language query support

**Use when**:
- Building enterprise search with natural language queries
- Creating AI-powered chat applications with your own data
- Implementing RAG patterns without infrastructure management
- Multi-tenant applications needing scoped search (folder-based filters)
- Continuous indexing of dynamic content (websites, documents)

**How it binds with Workers**:
- Yes - native Workers binding for search queries and AI-powered responses
- Typed access to search and query methods directly from Workers

**Key limits**:
- Max instances per account: 10
- Max files per instance: 100,000
- Individual file size: 4 MB max
- Max query results: 50
- Sync job cooldown: 3 minutes

**Key characteristics**:
- Automated indexing with continuous updates
- Similarity caching for improved latency
- Multitenancy support with folder-based filters
- Supports rich format files (PDF, etc.)

**Status**: Open Beta (formerly AutoRAG)

**Pricing**:
- Free to enable AI Search itself
- Underlying services billed separately: R2, Vectorize, Workers AI, AI Gateway, Browser Rendering
- Requires active R2 subscription

**Links**:
- [AI Search Docs](https://developers.cloudflare.com/ai-search/)

---

### Constellation
**Category**: Intelligence/Compute
**What it does**: DEPRECATED - Former platform for ML inference (replaced by Workers AI)

**Status**: ❌ **DEPRECATED** - Replaced by Workers AI

**Migration path**:
- Constellation was renamed and evolved into **Workers AI** (GA in April 2024)
- Workers AI provides the same ML inference functionality with improvements:
  - More powerful GPUs (faster inference, bigger models)
  - Expanded model catalog
  - Serverless GPU-powered execution
  - Better Time-to-First-Token (TTFT) performance

**What it was**:
- Platform for running pre-trained ML models (ONNX, TensorFlow) at the edge
- Model size limit: 50 MB
- Globally distributed inference

**Current alternative**: Use [Workers AI](https://developers.cloudflare.com/workers-ai/) instead

**Links**:
- [Workers AI Docs](https://developers.cloudflare.com/workers-ai/) (replacement product)

---

## Observability Layer

### Analytics Engine
**Category**: Observability  
**What it does**: Write custom analytics from Workers

**Use when**:
- Custom metrics and events
- Usage tracking
- Business intelligence

**Binds with**: Workers

**Key limits**: Write rate limits

**Pricing**: Writes + queries

**Links**:
- [Analytics Engine Docs](https://developers.cloudflare.com/analytics/analytics-engine/)

---

### Logpush
**Category**: Observability  
**What it does**: Stream logs to external services

**Use when**:
- Need logs in external system (S3, Splunk, etc.)
- Long-term log retention
- Compliance requirements

**Binds with**: Streams logs from Workers, R2, etc.

**Destinations**: S3, R2, Splunk, Datadog, etc.

**Pricing**: By volume

**Links**:
- [Logpush Docs](https://developers.cloudflare.com/logs/logpush/)

---

### Tail Workers
**Category**: Observability
**What it does**: Real-time log streaming from Workers

**Use when**:
- Real-time debugging
- Custom log processing
- Metrics collection

**Binds with**: Workers (receives logs from other Workers)

**Pricing**: Based on Workers pricing

**Links**:
- [Tail Workers Docs](https://developers.cloudflare.com/workers/observability/logging/tail-workers/)

---

### Workers Logs
**Category**: Observability
**What it does**: Built-in logging for all Workers (enabled by default)

**Use when**:
- Debugging Workers
- Monitoring production issues
- Analyzing request patterns
- Troubleshooting errors

**Key characteristics**:
- Enabled by default (no configuration needed)
- Stores logs for 24 hours
- console.log(), console.error(), console.warn() automatically captured
- View via dashboard or API
- Integrates with Logpush for long-term storage

**How to use**:
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    console.log('Request received:', request.url);
    console.error('Error occurred:', error);
    return new Response('OK');
  }
}
```

**Pricing**: Included with Workers

**Links**:
- [Workers Logs Docs](https://developers.cloudflare.com/workers/observability/logs/)

---

## Platform Features

### Service Bindings
**Category**: Platform
**What it does**: Worker-to-Worker RPC communication without public internet

**Use when**:
- Microservices architecture
- Splitting frontend and backend Workers
- Internal API calls between Workers
- Reducing latency between Workers

**Binds with**: Workers call other Workers directly

**Key characteristics**:
- No public internet egress
- Lower latency than HTTP
- Type-safe RPC
- Automatic load balancing

**Pricing**: No additional cost

**Links**:
- [Service Bindings Docs](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)

---

### Smart Placement
**Category**: Platform/Optimization
**What it does**: Automatically places workloads near backend infrastructure

**Use when**:
- Optimizing full-stack applications
- Separating frontend (edge) from backend (near database)
- Reducing database latency
- Multi-region deployments

**How it works**:
- Frontend Workers run at the edge (near users)
- Backend Workers run near data sources
- Cloudflare automatically routes requests optimally

**Pricing**: No additional cost

**Links**:
- [Smart Placement Docs](https://developers.cloudflare.com/workers/configuration/smart-placement/)

---

### Cloudflare for Platforms
**Category**: Platform
**What it does**: Multi-tenant SaaS infrastructure for running customer code and managing custom domains

**Use when**:
- Building domain-specific application platforms (e-commerce, mobile apps, no-code)
- Powering AI-assisted development platforms where users deploy code
- Enabling user-level code customization and extensibility
- Providing isolated data storage per customer/tenant
- Delivering branded customer subdomains or custom domains at scale

**Key components**:
- **Workers for Platforms**: Run untrusted customer code in isolated Workers using dispatch namespaces
- **Cloudflare for SaaS**: Manage custom domains and subdomains for customers

**How it binds with Workers**:
- Uses dispatch namespaces and dynamic dispatch to run customer Workers
- Platform Workers can wrap around user Workers for custom logic
- User Workers can access bindings (KV, D1, R2, DO) provisioned for them

**Key limits**:
- Max 30s CPU per invocation (15 min for Cron/Queue Consumers)
- Unlimited scripts for platform customers
- Cloudflare for SaaS: 5,000 hostname limit (Enterprise for more)
- First 100 custom hostnames free

**Pricing**:
- Workers for Platforms: $25/month base (20M requests, 60M CPU ms, 1,000 scripts)
- Overages: $0.30/1M requests, $0.02/1M CPU ms, $0.02/script
- Cloudflare for SaaS: First 100 hostnames free, then $0.10/hostname/month

**Links**:
- [Cloudflare for Platforms Docs](https://developers.cloudflare.com/cloudflare-for-platforms/)
- [Workers for Platforms](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/)
- [Cloudflare for SaaS](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/)

---

### Zaraz
**Category**: Platform/Observability
**What it does**: Third-party tool manager that offloads analytics, marketing pixels, and chatbots from browser to edge

**Use when**:
- Loading analytics tools (Google Analytics, Facebook Pixel, etc.)
- Managing advertising pixels and conversion tracking
- Adding chatbots to your website
- Implementing marketing automation tools
- Need consent management for GDPR/privacy compliance

**How it binds with Workers**:
- Integrates through Worker Variables - dynamic variables using Workers
- Zaraz forwards context object to Worker as JSON POST
- Worker response becomes the variable value
- Enables custom logic (cart calculations, User ID lookups, hashing)

**Key characteristics**:
- Server-side processing at edge (reduces client-side performance impact)
- Built-in consent management platform
- Supports custom Managed Components
- Available on all Cloudflare plans
- 1M free events/month per account

**Pricing**:
- Free tier: 1,000,000 events/month (all features)
- Paid: $5/month per additional 1,000,000 events
- One event = page view, zaraz.track event, or similar

**Links**:
- [Zaraz Docs](https://developers.cloudflare.com/zaraz/)

---

### Workers VPC
**Category**: Platform/Network
**What it does**: Connect Workers to private networks across clouds

**Use when**:
- Accessing private cloud resources
- Enterprise integration requirements
- Hybrid cloud architectures
- Connecting to VPCs in AWS, GCP, Azure

**Key characteristics**:
- Private network connectivity
- No public internet exposure
- Multi-cloud support
- Secure tunneling

**Status**: Open Beta

**Pricing**: Beta pricing

**Links**:
- [Workers VPC Docs](https://developers.cloudflare.com/workers/configuration/vpc/)

---

### Gradual Deployments
**Category**: Platform/DevOps
**What it does**: Deploy Worker changes gradually with traffic shifting

**Use when**:
- Reducing deployment risk
- Testing new versions with subset of traffic
- Canary deployments
- Blue-green deployments

**Key characteristics**:
- Traffic splitting by percentage
- Automatic rollback
- Version management
- Zero downtime deployments

**Pricing**: No additional cost

**Links**:
- [Gradual Deployments Docs](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments/)

---

## Security & Protection

### Secrets Store
**Category**: Security/Platform
**What it does**: Centralized secret management with RBAC and audit logging

**Use when**:
- Managing API keys and credentials
- Enterprise secret management
- Compliance requirements (audit logging)
- Rotating secrets across multiple Workers

**Key characteristics**:
- Centralized management
- Role-based access control
- Audit logging
- Secret rotation

**Status**: Beta

**Pricing**: Beta pricing

**Links**:
- [Secrets Store Docs](https://developers.cloudflare.com/workers/configuration/secrets/)

---

### Turnstile
**Category**: Security
**What it does**: CAPTCHA-free bot protection

**Use when**:
- Protecting forms from bots
- Login pages
- Account creation
- API endpoints needing bot protection

**Key characteristics**:
- No user friction (invisible verification)
- Privacy-focused (no tracking)
- Three modes: Managed, Non-Interactive, Invisible
- Server-side verification required

**Integration**: Widget + Workers verification

**Pricing**: Free tier available

**Links**:
- [Turnstile Docs](https://developers.cloudflare.com/turnstile/)

**See also**: `security-patterns.md` for complete integration example

---

### WAF (Web Application Firewall)
**Category**: Security
**What it does**: Protects applications from attacks and vulnerabilities

**Use when**:
- Production applications
- Protecting against OWASP Top 10
- IP-based blocking
- Rate limiting at edge

**How it integrates with Workers**:
```
WAF (runs first) → Filters malicious → Workers (business logic)
```

**Key capabilities**:
- Pre-configured rule sets
- Custom rules
- Rate limiting
- Geo-blocking
- IP reputation

**Pricing**: Pro plan and above

**Links**:
- [WAF Docs](https://developers.cloudflare.com/waf/)

**See also**: `security-patterns.md` for WAF + Workers integration patterns

---

### Cloudflare Tunnel
**Category**: Developer Tools/Security
**What it does**: Expose local/private services through Cloudflare without public IP

**Use when**:
- Local development with Workers
- Connecting to private resources
- Exposing internal services securely
- Development and testing workflows

**Key capabilities**:
- No open ports required
- Zero Trust access control
- Automatic HTTPS
- Works with Workers

**Pricing**: Free for developers

**Links**:
- [Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

---

### Privacy Gateway
**Category**: Security/Privacy
**What it does**: Managed OHTTP relay that hides client IP addresses from application backends

**Use when**:
- Building privacy-oriented applications (health apps, sensitive data platforms)
- Strong privacy requirements where client anonymity is critical
- Implementing "Mixnet"-like approaches to obfuscate source and destination
- Applications needing IP address anonymization
- Privacy-preserving analytics or data collection

**How it works**:
- Implements Oblivious HTTP (OHTTP) IETF standard
- Acts as trusted intermediary (relay) between clients and application backends
- End-to-end encryption: relay sees message length and target server, not application data
- Creates separation: Cloudflare knows source but not content, applications know content but not source

**How it binds with Workers**:
- No native Workers binding
- Requires client-side encryption and application gateway server (OHTTP gateway)
- Client and gateway must implement OHTTP protocol
- Not a simple API integration - requires architectural changes

**Key characteristics**:
- Relay observes: message length, target server, source IP
- Relay cannot see: application data (end-to-end encrypted)
- Client-side encryption required (key known only to client)
- Application data should not contain user-unique identifiers
- Recommend disabling crash reporting (may contain sensitive data)

**Status**: Closed Beta - Enterprise only

**Availability**:
- Available only to select privacy-oriented companies and partners
- Contact Cloudflare Privacy Edge program to apply

**Pricing**: Enterprise pricing (not publicly available)

**Links**:
- [Privacy Gateway Docs](https://developers.cloudflare.com/privacy-gateway/)

---

## Media Services

### Stream
**Category**: Media
**What it does**: Complete video platform for upload, storage, encoding, and delivery of live and on-demand video

**Use when**:
- Building video features in websites and native applications
- Creating video platforms or user-generated content sites
- Enabling creator uploads through one-time upload URLs
- Streaming to multiple platforms (web, iOS, Android, Apple TV)
- Implementing access-controlled video content with signed URLs

**How it binds with Workers**:
- No native Workers binding
- Integration via Stream API (REST) called from Workers using fetch()
- Workers can generate signed URLs, process upload webhooks, manage video metadata

**Key limits**:
- Max file size: 30 GB per upload
- Concurrent encoding: Up to 120 videos queued or encoding simultaneously
- Resolution: 360p to 1080p adaptive streaming
- Delivery codec: H.264 (industry standard)
- Videos removed if subscription lapses beyond 30 days

**Supported formats**:
- MP4, MKV, MOV, AVI, FLV, MPEG-2 TS, MPEG-2 PS, MXF, LXF, GXF, 3GP, WebM, MPG, QuickTime
- Recommended: MP4/H.264, AAC audio, ≤30 FPS

**Pricing**:
- Storage: $5 per 1,000 minutes stored (prepaid)
- Delivery: $1 per 1,000 minutes delivered (post-paid)
- Billed in 4-second intervals (precise billing)
- Pro/Business plans: 100 free minutes storage + 10,000 free minutes delivery/month
- Ingress and encoding: Always free

**Links**:
- [Stream Docs](https://developers.cloudflare.com/stream/)

---

### MoQ (Media over QUIC)
**Category**: Media/Network
**What it does**: Low-latency live media streaming protocol over QUIC transport

**Use when**:
- Building low-latency live streaming applications
- Real-time media delivery requirements (sub-second latency)
- Interactive live video applications
- Next-generation streaming protocols (beyond HLS/DASH)
- Testing emerging media standards

**How it binds with Workers**:
- Infrastructure-level integration (not application-level binding)
- MoQ relay network runs on every Cloudflare server globally

**Key characteristics**:
- Based on draft-07 of MoQ Transport specification (subset supported)
- Open protocol developed at IETF (not proprietary)
- Safari limitation: WebTransport support incomplete (Safari 18.4+ early testing)
- May require WebRTC or WebSocket fallbacks for Safari compatibility
- First MoQ relay network (330+ cities)

**Status**: Tech Preview (potential breaking changes)

**Pricing**:
- Tech Preview: Free at any scale
- Future GA pricing: $0.05/GB outbound (self-serve), no inbound cost
- Enterprise: Standard media delivery pricing

**Links**:
- [MoQ Docs](https://developers.cloudflare.com/moq/)

---

## Other Services

### Images
**Category**: Media
**What it does**: Image transformation and optimization

**Use when**:
- Serving images
- On-the-fly resizing/optimization
- Image transformations

**Binds with**: Workers, R2

**Pricing**: Transformations + storage

**Links**:
- [Images Docs](https://developers.cloudflare.com/images/)

---

### Pages
**Category**: Platform
**What it does**: Static site hosting with CI/CD

**Use when**:
- Deploying static sites
- JAMstack applications
- Full-stack apps (with Pages Functions)

**Binds with**: Pages Functions (Workers), Git providers

**Pricing**: Free tier, then per request

**Links**:
- [Pages Docs](https://developers.cloudflare.com/pages/)

---

## Composition Matrix

| Primary Product | Commonly Combined With | Pattern |
|-----------------|------------------------|---------|
| Workers | KV, D1, R2, DO, Queues, AI Gateway | All patterns |
| Pages | D1, R2, KV, Workers | Full-stack apps |
| Durable Objects | KV, R2, Queues | Stateful coordination |
| D1 | Workers, Pages, Hyperdrive | Database layer |
| Hyperdrive | Workers, D1 | External DB integration |
| AI Gateway | Workers, KV, D1, AI Search | AI applications |
| AI Search | Workers AI, Vectorize, D1, R2 | RAG, enterprise search |
| Browser Rendering | Workers, DO, R2 | Automation, scraping |
| Queues | Workers, R2, D1 | Background processing |
| Stream | Workers, R2, D1 | Video platforms |
| Zaraz | Workers (Worker Variables) | Analytics, marketing |
| Cloudflare for Platforms | Workers, KV, D1, R2, DO | Multi-tenant SaaS |

---

## Update History

- 2025-11-20 (Phase 2): Added 7 products to achieve 100% llms.txt coverage (Cloudflare for Platforms, Zaraz, Stream, MoQ, AI Search, Privacy Gateway, Constellation [deprecated])
- 2025-11-20 (Phase 1): Added 12 products (Service Bindings, Smart Placement, Workers VPC, Gradual Deployments, Secrets Store, Turnstile, WAF, Cloudflare Tunnel, Workers Logs, Pipelines, R2 SQL, R2 Data Catalog)
- 2025-11-18: Initial catalog created

**Coverage Status**: 27/27 products from llms.txt (100% coverage of Cloudflare Developer Platform) ✅
