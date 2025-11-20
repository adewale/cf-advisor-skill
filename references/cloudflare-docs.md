# Cloudflare Documentation Sources

This document tracks the authoritative sources for Cloudflare Developer Platform documentation.

## Sources of Truth

### Primary: llms.txt (Developer Platform Products)

**URL**: https://developers.cloudflare.com/llms.txt
**Purpose**: LLM-optimized documentation index for developer platform products
**Format**: Markdown with hierarchical structure
**Coverage**: 36 developer-focused products
**Update frequency**: Auto-generated from documentation builds

**Products included**:
- Compute: Workers, Durable Objects, Pages, Containers, Workflows
- Storage: KV, D1, R2, R2 Data Catalog, R2 SQL, Hyperdrive, Vectorize
- AI: Workers AI, AI Gateway, Agents, Browser Rendering
- Data: Pipelines, Analytics Engine
- Messaging: Queues, Pub/Sub, Email Routing
- Real-time: Realtime, RealtimeKit, SFU, TURN Service, Stream
- Media: Images, MoQ
- Platform: Cloudflare for Platforms, Sandbox SDK, Workers VPC, Zaraz
- Other: Constellation, Privacy Gateway, Developer Spotlight

**Usage**: Primary reference for this skill - covers products developers BUILD WITH

---

### Secondary: GitHub Repository (Complete Catalog)

**URL**: https://github.com/cloudflare/cloudflare-docs
**Path**: `src/content/docs/`
**Purpose**: Source code for all Cloudflare documentation
**Coverage**: 97 product directories (all documented products)
**License**: MIT (code), CC BY 4.0 (docs)

**Additional products not in llms.txt** (infrastructure/security focus):
- Security: WAF, DDoS Protection, Bots, Page Shield, API Shield, Turnstile, Challenges
- Zero Trust: Cloudflare One, Access, Gateway, Browser Isolation, Tunnel, WARP
- Networking: Magic Transit, Magic WAN, Magic Firewall, Spectrum, Load Balancing
- DNS: 1.1.1.1, DNS, Registrar, DMARC Management
- Platform: Cache, Speed, Rules, Ruleset Engine, Analytics, SSL/TLS
- Tools: Terraform, Pulumi, Learning Paths, Reference Architecture

**Usage**: Monitor for new products and features

---

### Tertiary: Complete Documentation

**llms-full.txt**: https://developers.cloudflare.com/llms-full.txt
- Complete documentation export (930,080+ lines)
- All products, all pages with metadata
- Too large for regular use, fetch specific products as needed

**Per-product llms-full.txt**:
- `https://developers.cloudflare.com/{product}/llms-full.txt`
- Examples: `/workers/llms-full.txt`, `/d1/llms-full.txt`
- Use for deep dives into specific products

**markdown.zip**: https://developers.cloudflare.com/markdown.zip
- Offline archive of all documentation
- Downloadable for offline reference

---

### Web Resources

**Products Directory**: https://developers.cloudflare.com/products/
- Human-friendly alphabetical product listing
- All documented products with descriptions
- Good for discovery

**Developer Portal**: https://developers.cloudflare.com/
- Main documentation hub
- Getting started guides
- Featured products and use cases

**Blog**: https://blog.cloudflare.com/tag/developer-platform/
- Product announcements
- Developer Week posts
- New feature releases

---

## llms.txt Standard

**Specification**: https://llmstxt.org/
**Created by**: Jeremy Howard (Answer.AI)
**Purpose**: Standardized format for LLM-friendly documentation

### Format

```markdown
# Project Name

> Short summary

## Product/Section
- [Page Title](URL to .md file)
```

### Cloudflare Implementation

- `/llms.txt` - Developer platform index (curated, 36 products)
- `/llms-full.txt` - Complete documentation
- `/[product]/llms-full.txt` - Per-product documentation
- Per-page markdown: Append `/index.md` to any URL
- Markdown copy button: Available on all documentation pages

---

## Update Strategy

### Monitoring for Changes

**Monthly check**:
```bash
# Count products in llms.txt
curl -s https://developers.cloudflare.com/llms.txt | grep "^## " | wc -l
# Current: 36

# List all products
curl -s https://developers.cloudflare.com/llms.txt | grep "^## " | sed 's/^## //'
```

**Watch GitHub**:
- Star: https://github.com/cloudflare/cloudflare-docs
- Monitor: `src/content/docs/` for new directories
- Subscribe to releases

**Track announcements**:
- Developer Week (April & November)
- Birthday Week (September)
- Blog: https://blog.cloudflare.com/tag/developer-platform/

### When New Product Detected

1. **Verify scope**: Is it developer platform or infrastructure?
2. **Add to primitives-catalog.md**: Categorize and document capabilities
3. **Add to workers-integrations.md**: If it has Workers bindings
4. **Add patterns**: Update composition-patterns.md if needed
5. **Add examples**: Create complete example in workers-examples.md
6. **Update SKILL.md**: Only if mental models change (rare)

### What NOT to Duplicate

- ❌ Don't copy llms.txt content verbatim
- ❌ Don't duplicate full documentation
- ❌ Don't copy code examples without adding context
- ✅ DO add mental models, composition patterns, curated knowledge

---

## Coverage Scope

### Included in This Skill (Developer Platform)

Products from llms.txt that developers BUILD WITH:
- Compute, Storage, AI, Real-time, Data, Messaging
- Platform features (Service Bindings, Smart Placement)
- Security integration patterns (WAF + Workers, Turnstile)

### Excluded (Infrastructure/Enterprise Focus)

Products not in llms.txt or out of scope:
- Pure infrastructure (Magic products, Spectrum)
- DNS management (standalone, not Workers integration)
- Zero Trust detailed configuration (enterprise-focused)
- Network infrastructure products

For these products, refer users to official Cloudflare documentation.

---

## Quick Links

### Documentation
- llms.txt: https://developers.cloudflare.com/llms.txt
- llms-full.txt: https://developers.cloudflare.com/llms-full.txt
- Products: https://developers.cloudflare.com/products/
- Workers: https://developers.cloudflare.com/workers/

### Source Code
- GitHub: https://github.com/cloudflare/cloudflare-docs
- Issues: https://github.com/cloudflare/cloudflare-docs/issues

### Community
- Discord: https://discord.cloudflare.com
- Community: https://community.cloudflare.com
- Workers Discord: #workers channel

### Tools
- Wrangler: https://developers.cloudflare.com/workers/wrangler/
- Playground: https://workers.cloudflare.com/playground
- Dashboard: https://dash.cloudflare.com

---

## Last Updated

- **Date**: 2025-11-20
- **llms.txt product count**: 27 (verified via WebFetch)
- **GitHub directories**: 97 (total Cloudflare products)
- **Skill coverage**: 27/27 products = **100% of developer platform products** ✅

---

## Notes

- llms.txt is the RIGHT source for this skill (developer platform focus)
- GitHub repo is authoritative for complete product catalog
- New products appear in GitHub first, then llms.txt
- This skill maintains curated knowledge, not raw documentation
- Always prefer teaching mental models over syntax reference
