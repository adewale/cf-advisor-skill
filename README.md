# Cloudflare Developer Skill

Comprehensive skill for building applications on the Cloudflare Developer Platform.

## What This Skill Does

Teaches mental models, composition patterns, and code generation for:
- **Compute**: Workers, Durable Objects, Pages, Containers, Workflows
- **Storage**: KV, D1, R2, Vectorize, Hyperdrive
- **AI**: Workers AI, Agents, AI Gateway, Browser Rendering
- **Real-time**: WebSockets, Calls, Stream, Pub/Sub
- **Data**: Pipelines, R2 SQL, R2 Data Catalog, Analytics Engine
- **Platform**: Service Bindings, Smart Placement, Workers VPC, Secrets Store
- **Security**: WAF integration, Turnstile, rate limiting patterns

## When to Use This Skill

Invoke when the user asks about:
- Cloudflare products and architecture
- Migrating applications to Cloudflare
- Building edge applications and serverless functions
- Code generation for Workers
- Composition patterns and solution design
- Choosing the right Cloudflare primitives

## File Structure

```
.
├── README.md                              # This file
├── SKILL.md                               # Core mental models & decision trees
└── references/
    ├── primitives-catalog.md              # Product catalog & capabilities
    ├── composition-patterns.md            # Solution architectures
    ├── migration-guides.md                # Migration strategies
    ├── workers-best-practices.md          # Code standards & security
    ├── workers-integrations.md            # Service integration details
    ├── workers-examples.md                # Complete code examples (11 patterns)
    ├── workers-websockets-agents.md       # WebSocket & Agent patterns
    ├── security-patterns.md               # Security integration patterns
    ├── wrangler-commands.md               # CLI reference
    └── cloudflare-docs.md                 # Documentation links & sources
```

## How It Works

### 1. Mental Models First
The skill teaches **how to think** about edge computing, not just what to type. SKILL.md contains core paradigm shifts (servers → Workers) that enable solving novel problems.

### 2. Progressive Disclosure
Claude Code loads only relevant reference files based on the user's question:
- Conceptual questions → primitives-catalog.md, composition-patterns.md
- Code generation → workers-best-practices.md, workers-integrations.md, workers-examples.md
- WebSockets/Agents → workers-websockets-agents.md
- Security → security-patterns.md

### 3. Complete Examples
All code examples are production-ready, runnable, and include:
- Complete TypeScript code
- Full wrangler.jsonc configuration
- Dependencies and setup instructions
- Best practices and security patterns

## Sources of Truth

### Primary: llms.txt (Developer Platform)
- **URL**: https://developers.cloudflare.com/llms.txt
- **Coverage**: 36 developer platform products
- **Updated**: Auto-generated from documentation
- **Scope**: Products developers BUILD WITH (not infrastructure)

**Products included**:
Agents, AI Gateway, AI Search, Browser Rendering, Cloudflare for Platforms, Images, Constellation, Containers, D1, Durable Objects, Email Routing, Hyperdrive, KV, MoQ, Pages, Pipelines, Pub/Sub, Queues, R2, R2 Data Catalog, R2 SQL, Realtime, Stream, TURN Service, Vectorize, Workers, Workers AI, Workers Analytics Engine, Workers VPC, Workflows, Zaraz, and more.

### Secondary: GitHub Repository (Complete Catalog)
- **URL**: https://github.com/cloudflare/cloudflare-docs
- **Coverage**: 97 total product directories
- **Use**: Monitor for new products not yet in llms.txt
- **Path**: `src/content/docs/`

### Tertiary: Complete Documentation
- **llms-full.txt**: Complete docs (930K+ lines) - https://developers.cloudflare.com/llms-full.txt
- **markdown.zip**: Offline archive - https://developers.cloudflare.com/markdown.zip
- **/products/**: Human-friendly directory - https://developers.cloudflare.com/products/

### What's NOT Included
By design, this skill focuses on **developer platform products** and excludes:
- Infrastructure products (Magic Transit, Magic WAN, Spectrum)
- DNS management (DNS, Registrar, DMARC)
- Zero Trust suite (detailed configuration - interaction patterns only)
- Enterprise networking products

For these products, reference official Cloudflare documentation directly.

## Update Strategy

### Monitoring for New Products

**Monthly checks**:
```bash
# Check product count in llms.txt
curl -s https://developers.cloudflare.com/llms.txt | grep "^## " | wc -l

# List all products
curl -s https://developers.cloudflare.com/llms.txt | grep "^## " | sed 's/^## //'
```

**Watch for announcements**:
- Developer Week (April & November)
- Birthday Week (September)
- Blog: https://blog.cloudflare.com/tag/developer-platform/

### When New Product Detected

1. **Add to primitives-catalog.md**
   - Categorize (Compute/Storage/Network/Intelligence/Observability)
   - Document capabilities and use cases
   - Add to decision framework

2. **Add integration details to workers-integrations.md**
   - Binding configuration
   - Code examples
   - When to use

3. **Add composition patterns** (if applicable)
   - Show how it composes with other primitives
   - Architecture diagrams
   - Use case examples

4. **Add complete example to workers-examples.md**
   - Full working code
   - Configuration
   - Setup instructions

5. **Update decision trees in SKILL.md** (only if mental models change)
   - Most updates won't require SKILL.md changes
   - Keep SKILL.md focused on paradigms, not product details

### What NOT to Duplicate

- ❌ Don't copy llms.txt content verbatim (reference it)
- ❌ Don't duplicate code examples from docs (curate and add context)
- ❌ Don't include full documentation (link to it)
- ✅ DO add mental models, composition patterns, and curated knowledge

## Coverage Status

**Current focus**: Developer Platform (80-85% coverage of llms.txt products)

### Well Covered (Deep)
- ✅ Workers, Durable Objects, Pages
- ✅ KV, D1, R2, Hyperdrive
- ✅ Queues, Workflows
- ✅ Workers AI, Agents, AI Gateway
- ✅ WebSocket patterns, Hibernation API

### Recently Added
- ✅ Service Bindings (Platform feature)
- ✅ Smart Placement (Optimization)
- ✅ Secrets Store (Beta)
- ✅ Workers VPC (Private networking)
- ✅ Security patterns (WAF, Turnstile integration)
- ✅ R2 SQL, Pipelines, R2 Data Catalog (Data products)
- ✅ Workers Logs (Observability)

### Partial Coverage (Needs Expansion)
- ⚠️ Vectorize (needs complete RAG example)
- ⚠️ Images (needs transformation patterns)
- ⚠️ Pub/Sub (needs MQTT details)
- ⚠️ Realtime/Calls (needs real-time communication patterns)

### Out of Scope
- ❌ Infrastructure (Magic products, Spectrum)
- ❌ DNS management products
- ❌ Zero Trust detailed config (enterprise focus)
- ❌ Marketing/consumer products

## Design Principles

### 1. Mental Models Over Syntax
Teach the "why" not just the "what". Enable solving novel problems, not just recreating examples.

### 2. Complete, Runnable Examples
Every code example must be production-ready and self-contained. No placeholders or "TODO" comments.

### 3. Composition-First Thinking
Show how primitives compose into complete solutions. Emphasize patterns over individual products.

### 4. Progressive Disclosure
Keep SKILL.md focused on core concepts (<500 lines). Details go in reference files loaded on-demand.

### 5. LLM-Optimized Structure
- Clear decision trees for loading context
- Table of contents in large files
- One level deep (no nested references)
- Descriptive file names

## Contributing

When adding new products or patterns:

1. **Verify the source**: Check llms.txt or GitHub repo
2. **Maintain structure**: Follow existing file organization
3. **Add complete examples**: No partial code or placeholders
4. **Update decision trees**: Add triggers in SKILL.md for new content
5. **Keep SKILL.md lean**: Mental models only, details in references
6. **Test with questions**: Verify Claude Code loads the right context

## Quick Start for Claude Code

When a user asks about Cloudflare:

1. **Read SKILL.md** for mental models and decision trees
2. **Follow decision tree** to load appropriate reference files
3. **Generate code** following workers-best-practices.md patterns
4. **Reference examples** from workers-examples.md for similar use cases
5. **Validate** against best practices before presenting to user

## License

This skill documentation is provided as-is for use with Claude Code.

Cloudflare documentation and examples are subject to Cloudflare's terms of service.

---

**Last Updated**: 2025-11-20
**llms.txt Product Count**: 36
**Skill Coverage**: ~80-85% of developer platform products
