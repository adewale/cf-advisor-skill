# Cloudflare Developer Skill

**Comprehensive Cloudflare Developer Platform knowledge for Claude Code**

- 27 products, 100% llms.txt coverage
- Mental models for edge computing
- Production-ready code examples
- Progressive disclosure (efficient token usage)

*This is an unofficial tool for use with Claude Code.*

---

## Installation

### Option 1: Personal Installation (Available in All Projects)

1. **Create the skills directory:**
   ```bash
   mkdir -p ~/.claude/skills
   ```

2. **Copy this skill:**
   ```bash
   cd ~/.claude/skills
   # Option A: Clone from git
   git clone https://github.com/adewale/cf-advisor-skill cloudflare

   # Option B: Copy existing directory
   cp -r /path/to/cf-advisor-skill cloudflare
   ```

3. **Verify structure:**
   ```bash
   ls ~/.claude/skills/cloudflare/SKILL.md
   # Should show: /Users/<you>/.claude/skills/cloudflare/SKILL.md
   ```

4. **Restart Claude Code** (if running)

### Option 2: Project Installation (This Project Only)

1. **In your project root:**
   ```bash
   mkdir -p .claude/skills
   cd .claude/skills

   # Clone or copy the skill here
   git clone https://github.com/adewale/cf-advisor-skill cloudflare
   ```

2. **Commit to version control** (team members get it automatically):
   ```bash
   git add .claude/skills/cloudflare
   git commit -m "Add Cloudflare Developer Skill"
   git push
   ```

3. **Team members:** Just `git pull` to get the skill automatically

**Which option to choose?**
- Use **Option 1** if you work on Cloudflare projects frequently
- Use **Option 2** if you want the skill versioned with your project

---

## Quick Start

### Verify Installation

Ask Claude Code these questions to verify the skill is working:

#### Test 1: Basic Discovery
```
You: "What is Cloudflare Workers?"

✓ Expected: Claude explains Workers with mental models
            (request-scoped execution at edge, stateless vs Durable Objects)

✓ Token usage: ~3,000 tokens (SKILL.md loaded)
```

#### Test 2: Decision Tree Routing
```
You: "How do I build a real-time chat app on Cloudflare?"

✓ Expected: Claude loads WebSocket patterns and suggests
            Durable Objects + WebSocket Hibernation API pattern

✓ Token usage: ~10,000 tokens (SKILL.md + workers-websockets-agents.md)
```

#### Test 3: Code Generation
```
You: "Generate a Worker that stores data in KV"

✓ Expected: Complete TypeScript code with:
            - Full Worker implementation
            - wrangler.jsonc configuration
            - Error handling and best practices
            - Security patterns

✓ Token usage: ~8,000 tokens (SKILL.md + workers-best-practices.md + workers-integrations.md)
```

### Common Tasks

**Architecture Guidance:**
- "How do I choose between KV, D1, and R2 for my use case?"
- "Show me patterns for composing Workers with databases"
- "What's the best architecture for a full-stack SaaS app?"

**Code Generation:**
- "Build a REST API with D1 database and authentication"
- "Create a Worker with rate limiting using Durable Objects"
- "Generate a WebSocket chat server with Durable Objects"

**Migration:**
- "How do I migrate from Express.js to Cloudflare Workers?"
- "Convert this Node.js app to run on Cloudflare"
- "What's the equivalent of Redis on Cloudflare?"

**Troubleshooting:**
- "Why is my Worker timing out?"
- "How do I debug KV eventually consistent issues?"
- "Best practices for error handling in Workers"

---

## What This Skill Does

Teaches mental models, composition patterns, and code generation for:

**Compute:**
- Workers, Durable Objects, Pages, Containers, Workflows

**Storage:**
- KV, D1, R2, Vectorize, Hyperdrive
- R2 SQL, R2 Data Catalog, Pipelines

**AI:**
- Workers AI, Agents, AI Gateway, AI Search, Browser Rendering

**Real-time & Media:**
- WebSockets, Durable Objects, Pub/Sub, Stream, MoQ

**Platform:**
- Service Bindings, Smart Placement, Workers VPC
- Cloudflare for Platforms, Secrets Store, Gradual Deployments

**Security:**
- WAF integration patterns, Turnstile, rate limiting
- Secrets management, API security

**Observability:**
- Analytics Engine, Workers Logs, Logpush, Tail Workers

### When to Use This Skill

Claude Code automatically activates this skill when you ask about:
- Cloudflare products and architecture
- Building edge applications and serverless functions
- Migrating applications to Cloudflare
- Code generation for Workers
- Composition patterns and solution design
- Choosing the right Cloudflare primitives
- WebSocket and real-time patterns
- Security and best practices

---

## How It Works

### 1. Mental Models First

The skill teaches **how to think** about edge computing, not just what to type:

- **Paradigm shift:** From long-running servers to request-scoped execution
- **No filesystem:** Use bindings instead (KV, D1, R2)
- **No in-memory state:** Use Durable Objects for coordination
- **Edge-first architecture:** Run code near users, not in data centers
- **Composition:** Primitives compose into complete solutions

**Result:** Enable solving novel problems, not just recreating examples.

### 2. Progressive Disclosure

Claude Code loads only relevant reference files based on your question:

```
Question type               → Files loaded
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"What is X?"               → SKILL.md + primitives-catalog.md
"How do I build X?"        → SKILL.md + composition-patterns.md
"Generate code for X"      → SKILL.md + workers-best-practices.md
                             + workers-integrations.md
"WebSocket/real-time"      → SKILL.md + workers-websockets-agents.md
"Security/rate limiting"   → SKILL.md + security-patterns.md
"Show me an example of X"  → SKILL.md + workers-examples.md
```

**Token efficiency:**
- Startup: ~80 tokens (metadata only)
- Typical usage: ~7,500 tokens (SKILL.md + 1-2 references)
- Worst case: ~19,000 tokens (all files loaded)
- Context remaining: 181,000+ tokens for conversation (90% available)

### 3. Complete, Runnable Examples

All code examples are production-ready and include:
- Complete TypeScript code
- Full `wrangler.jsonc` configuration
- Dependencies and setup instructions
- Best practices and security patterns
- Error handling and observability

**No placeholders or "TODO" comments.**

### 4. How Claude Uses This Skill

When you ask about Cloudflare, Claude Code:

1. **Discovers** the skill from its description (automatic)
2. **Loads** SKILL.md for mental models and decision trees
3. **Follows** decision tree to determine which reference files to load
4. **Progressively loads** only relevant reference files (efficient)
5. **Generates** code following best practices from workers-best-practices.md
6. **Validates** against patterns from workers-examples.md

This happens automatically—you just ask questions naturally.

---

## Coverage & Sources

### Coverage Status

**Current:** 27/27 products from llms.txt = **100% Developer Platform coverage** ✅

#### Well Covered (Production-Ready Guidance)
- ✅ Workers, Durable Objects, Pages, Containers, Workflows
- ✅ KV, D1, R2, Hyperdrive
- ✅ Queues, Pub/Sub
- ✅ Workers AI, Agents, AI Gateway, AI Search
- ✅ WebSocket patterns, Hibernation API
- ✅ Service Bindings, Smart Placement, Workers VPC
- ✅ Security: WAF, Turnstile, Secrets Store, rate limiting
- ✅ Data: Pipelines, R2 SQL, R2 Data Catalog
- ✅ Observability: Workers Logs, Analytics Engine

#### Recently Added (2025-11-20)
- ✅ Cloudflare for Platforms (multi-tenant SaaS)
- ✅ Stream (video platform)
- ✅ Zaraz (third-party tool manager)
- ✅ AI Search (managed search with RAG)
- ✅ MoQ (Media over QUIC)
- ✅ Privacy Gateway (OHTTP relay)
- ✅ Constellation (deprecated, replaced by Workers AI)

### Sources of Truth

#### Primary: llms.txt (Developer Platform)
- **URL:** https://developers.cloudflare.com/llms.txt
- **Coverage:** 27 developer platform products
- **Updated:** Auto-generated from documentation
- **Scope:** Products developers BUILD WITH

#### Secondary: GitHub Repository
- **URL:** https://github.com/cloudflare/cloudflare-docs
- **Coverage:** 97 total product directories
- **Use:** Monitor for new products not yet in llms.txt
- **Path:** `src/content/docs/`

#### Tertiary: Complete Documentation
- **llms-full.txt:** Complete docs (930K+ lines) - https://developers.cloudflare.com/llms-full.txt
- **markdown.zip:** Offline archive - https://developers.cloudflare.com/markdown.zip
- **/products/:** Human-friendly directory - https://developers.cloudflare.com/products/

### What's NOT Included

By design, this skill focuses on **developer platform products** and excludes:
- ❌ Infrastructure products (Magic Transit, Magic WAN, Spectrum)
- ❌ DNS management (DNS, Registrar, DMARC)
- ❌ Zero Trust suite (detailed configuration—interaction patterns only)
- ❌ Enterprise networking products

For these products, reference official Cloudflare documentation directly.

---

## File Structure

```
cf-advisor-skill/
├── README.md                           # This file (for humans)
├── SKILL.md                            # Core mental models & decision trees
│
├── references/                         # Loaded on-demand by SKILL.md
│   ├── primitives-catalog.md          # 27 products, capabilities, limits
│   ├── composition-patterns.md        # 10 solution architectures
│   ├── migration-guides.md            # Server → Workers strategies
│   ├── workers-best-practices.md      # Code standards, security patterns
│   ├── workers-integrations.md        # Service integration code examples
│   ├── workers-examples.md            # 11 complete working examples
│   ├── workers-websockets-agents.md   # WebSocket & Agent patterns
│   ├── security-patterns.md           # WAF, Turnstile, rate limiting
│   ├── wrangler-commands.md           # CLI reference
│   └── cloudflare-docs.md             # Documentation index & sources
│
└── docs/                               # Meta-documentation (not loaded by Claude)
    └── why_skills.md                  # Skills vs MCP servers analysis
```

**Key insight:** Claude Code only loads SKILL.md and files it references. The `docs/` folder contains documentation ABOUT the skill for human readers.

---

## For Maintainers

### Update Strategy

#### Monthly Monitoring

Check for new products:
```bash
# Check product count in llms.txt
curl -s https://developers.cloudflare.com/llms.txt | grep "^## " | wc -l
# Current: 27

# List all products
curl -s https://developers.cloudflare.com/llms.txt | grep "^## " | sed 's/^## //'
```

Watch for announcements:
- **Developer Week** (April & November)
- **Birthday Week** (September)
- **Blog:** https://blog.cloudflare.com/tag/developer-platform/

#### When New Product Detected

1. **Research** product docs and llms.txt entry (30-60 min)

2. **Update primitives-catalog.md:**
   - Add to appropriate category
   - Document capabilities, use cases, limits
   - Add to decision framework ("Use when...")

3. **Update workers-integrations.md:**
   - Binding configuration
   - Code examples
   - When to use

4. **Update composition-patterns.md** (if major product):
   - Show how it composes with other primitives
   - Architecture diagrams
   - Complete pattern examples

5. **Add complete example to workers-examples.md** (if major product):
   - Full working code
   - Configuration
   - Setup instructions

6. **Update SKILL.md decision trees** (only if mental models change):
   - Most updates won't require SKILL.md changes
   - Keep SKILL.md focused on paradigms, not product details

**Time estimate:** ~2 hours per new product

### Contributing Guidelines

When adding new products or patterns:

1. ✅ **Verify the source:** Check llms.txt or GitHub repo
2. ✅ **Maintain structure:** Follow existing file organization
3. ✅ **Add complete examples:** No partial code or placeholders
4. ✅ **Update decision trees:** Add triggers in SKILL.md for new content
5. ✅ **Keep SKILL.md lean:** Mental models only (<500 lines), details in references
6. ✅ **Test with questions:** Verify Claude Code loads the right context

### Design Principles

#### 1. Mental Models Over Syntax
Teach the "why" not just the "what". Enable solving novel problems, not just recreating examples.

#### 2. Complete, Runnable Examples
Every code example must be production-ready and self-contained. No placeholders or "TODO" comments.

#### 3. Composition-First Thinking
Show how primitives compose into complete solutions. Emphasize patterns over individual products.

#### 4. Progressive Disclosure
Keep SKILL.md focused on core concepts (<500 lines). Details go in reference files loaded on-demand.

#### 5. LLM-Optimized Structure
- Clear decision trees for loading context
- Table of contents in large files
- One level deep (no nested references)
- Descriptive file names

### What NOT to Duplicate

- ❌ Don't copy llms.txt content verbatim (reference it instead)
- ❌ Don't duplicate code examples from docs without adding context
- ❌ Don't include full documentation (link to it)
- ✅ DO add mental models, composition patterns, and curated knowledge

---

## Troubleshooting

### Skill Not Activating?

**Check installation path:**
```bash
# Personal installation
ls ~/.claude/skills/cloudflare/SKILL.md

# Project installation
ls .claude/skills/cloudflare/SKILL.md
```

**Both should show the SKILL.md file path.**

**If missing:**
- Re-run installation steps above
- Check you're in the correct directory
- Verify folder name is exactly `cloudflare` (lowercase)

**If exists but not activating:**
- Restart Claude Code
- Check SKILL.md has valid YAML frontmatter
- Try asking a more specific question (e.g., "Explain Cloudflare Workers" instead of "Tell me about Cloudflare")

### Wrong or Outdated Answers?

**Check skill version:**
```bash
# Check last updated date
grep "Last Updated" ~/.claude/skills/cloudflare/README.md
```

**If outdated:**
```bash
cd ~/.claude/skills/cloudflare
git pull  # If installed via git
# or re-copy the updated skill
```

**If current but wrong:**
- The skill may not cover that specific topic yet
- Check "Coverage Status" section above
- Report issue with your question and expected answer

### Token Usage Concerns?

**Normal behavior:**
- Startup: ~80 tokens (just metadata)
- Typical conversation: ~7,500 tokens
- Complex questions: ~15,000-19,000 tokens

**This is expected** due to progressive disclosure. The skill loads comprehensive knowledge only when needed.

**Check current usage:**
```
/context
```
(Claude Code command to see token usage)

**If consistently high:**
- This is by design for comprehensive knowledge
- Alternative: Use smaller, focused skills
- Or: Only ask specific questions (loads fewer files)

### Getting Better Answers

**Be specific:**
- ❌ "How do I use Cloudflare?"
- ✅ "How do I build a REST API with D1 database on Cloudflare Workers?"

**Provide context:**
- "I'm migrating from Express.js to Cloudflare Workers"
- "I need real-time WebSocket communication for a chat app"
- "I want to store user sessions—KV or D1?"

**Ask follow-ups:**
- "Show me the complete code for that"
- "How do I deploy this?"
- "What are the security considerations?"

---

## License

This skill documentation is provided as-is for use with Claude Code.

Cloudflare documentation and examples are subject to Cloudflare's terms of service.

---

## Metadata

**Last Updated:** 2025-11-20
**llms.txt Product Count:** 27 (verified via WebFetch)
**Skill Coverage:** 27/27 products = **100% Developer Platform coverage** ✅
**Total Skill Lines:** ~8,181 (SKILL.md + references/)
**Architecture:** Skills-based (optimal for documentation—see docs/why_skills.md)

**Maintainer Notes:**
- See `docs/why_skills.md` for architectural rationale (Skills vs MCP servers)
- Monthly monitoring: Check llms.txt for new products
- Update frequency: ~5-8 new products/year during Cloudflare events
