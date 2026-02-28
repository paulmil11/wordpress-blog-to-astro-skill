# WordPress to Astro Migration Skill

A Claude Code skill for migrating WordPress blogs to Astro static sites. Built from real migrations across multiple sites — a bilingual podcast/blog ([angiecreates.io](https://angiecreates.io)) and a consulting content site ([strategyu.co](https://strategyu.co)).

| [strategyu.co](https://strategyu.co) | [angiecreates.io](https://angiecreates.io) |
|---|---|
| ![StrategyU — consulting skills site on Vercel](images/strategyu-screenshot.png) | ![Angie Creates — bilingual podcast/blog on Cloudflare Pages](images/angiecreates-screenshot.png) |

## What This Skill Does

When you invoke `/wordpress-to-astro` in any project, Claude gets loaded with the full migration playbook: a 6-phase pipeline, ready-to-customize script templates, and a troubleshooting guide covering dozens of real issues.

Claude will walk you through the migration step by step, asking the right questions upfront and adapting scripts to your specific site.

## Install

Personal (just you):
```bash
# Copy skill files to your local Claude skills directory
mkdir -p ~/.claude/skills/wordpress-to-astro
# Then copy SKILL.md, migration-scripts.md, common-issues.md into it
```

As a plugin (shareable):
```
/install github:paulmil11/wordpress-blog-to-astro-skill
```

## How It Works

### The Migration Pipeline

The skill guides Claude through 6 phases, run in order:

1. **Extract** — Parse your WordPress XML export into structured JSON. Pulls out titles, slugs, dates, categories, tags, featured images (via `_thumbnail_id` postmeta lookup), and full HTML content.

2. **Convert** — Transform WordPress HTML to clean Markdown using Turndown with custom rules. Preserves YouTube/podcast/Substack embeds as raw HTML. Strips WordPress block comments, spacer blocks, and page builder wrappers. Generates YAML frontmatter with dual date fields (`date` for display, `rawDate` for sorting).

3. **Download Images** — Scan all markdown for external image URLs, download them locally to `public/images/posts/`, and rewrite all references. Handles CDN redirects (up to 5 hops), tracks failures, and is idempotent (safe to re-run).

4. **Clean Up** — Remove boilerplate (host social links, platform CTAs, subscription prompts) while intelligently preserving guest content. Standardize categories. Fix broken links from old domains. Normalize percent-encoded CJK filenames.

5. **Generate Missing Content** — Auto-create landing pages for content that exists in external feeds (podcast RSS, newsletters) but has no blog post yet. Match by episode number, inject embed players.

6. **Build & Deploy** — Verify the Astro build, configure platform-specific redirects (Netlify `_redirects`, Vercel `vercel.json`, Cloudflare Pages), and deploy.

### What Claude Asks You First

Before writing any code, the skill prompts Claude to ask about your specific setup:

| Question | Why It Matters |
|----------|---------------|
| **Source platform** — WordPress.com or self-hosted? | Affects XML export structure and image URL patterns |
| **Hosting target** — Cloudflare Pages, Vercel, or Netlify? | Determines redirect config format and deploy setup |
| **Content types** — Blog only? Podcast? Events? Portfolio? | Controls which scripts get created |
| **Languages** — Monolingual or multilingual? | Affects filename handling, reading time calculation, regex patterns for cleanup |
| **Old domain** — Need redirects from old URLs? | Creates redirect mapping from WordPress slugs |
| **External feeds** — Podcast RSS? Newsletter? | Enables auto-generation of missing content pages |
| **Embeds** — YouTube, podcast players, Instagram, Substack? | Configures Turndown custom rules to preserve specific embed types |

### Key Architecture Decisions

The skill encodes opinions that come from doing this multiple times:

- **Content Collections over JSON** — Frontmatter lives with the content, type-safe via Zod
- **Plain Markdown, not MDX** — WordPress HTML is too messy for MDX's strict parser
- **Static output** — No server needed, fastest loads, simplest deploys
- **GitHub as CMS** — Edit markdown directly, version control built in
- **Single `[slug].astro` route** — One template for all posts, zero duplication
- **Dual date fields** — `date` for human-readable display, `rawDate` for reliable sorting

## What's Included

### `SKILL.md` — Main Migration Guide
The core skill file. Contains the 6-phase pipeline, project structure template, content collection schema, dependency list, architecture decisions, and customization points. This is what Claude reads when you invoke the skill.

### `migration-scripts.md` — Script Templates
Ready-to-customize code for every phase:

- **WordPress XML Parser** — Extract posts, categories, tags, featured images from export XML using `xml2js`
- **HTML-to-Markdown Converter** — Turndown with custom rules for embeds, tables, WordPress block comments
- **Image Downloader** — Bulk download external images with redirect following, failure tracking, URL rewriting
- **RSS-to-Markdown Converter** — Simpler HTML conversion for RSS feed content (critical: prevents URL jamming by adding spaces around `<a>` tag replacements)
- **Boilerplate Cleanup Pattern** — URL-pattern-based detection that distinguishes host content from guest content
- **Platform Redirect Generators** — Output `_redirects` (Netlify/Cloudflare) or `vercel.json` format
- **CJK Reading Time** — Counts Chinese characters (~350/min) separately from Latin words (~238/min)
- **Astro-to-Markdown Converter** — For two-stage migrations where posts were first generated as `.astro` pages then converted to content collections

### `post-migration-patterns.md` — Post-Migration Enhancements
Patterns for building out the Astro site after content is migrated:

- **SEO Meta Tags** — Full OG, Twitter Card, JSON-LD structured data, canonical URLs, noindex support
- **Auto-Generated Featured Images** — SVG generation from post titles with keyword-based icons (no need to create a unique image per post)
- **Category-Matched CTAs** — Dynamic calls-to-action mapped to post categories with UTM tracking
- **Related Posts** — Category-overlap matching with fallback to recent posts
- **Newsletter Integration** — ConvertKit/Kit forms with Intersection Observer lazy loading
- **WordPress CSS Compatibility** — Styles for alignment classes, video embeds, tables, code blocks, blockquotes, spacer blocks
- **Dynamic Routes with Draft Filtering** — `[slug].astro` pattern with dev-only draft visibility
- **robots.txt** — Template for Astro static sites with sitemap reference
- **DNS Prefetch** — Performance optimization for third-party scripts
- **Cross-Linking & Internal SEO** — Automated cross-link detection script, pillar/cluster model, category index pages, breadcrumbs, Article JSON-LD schema, and post-migration SEO checklist
- **Redirects from Old WordPress URLs** — Full redirect generation script plus platform-specific configs (Vercel, Cloudflare Pages, Netlify, DNS-level) with testing commands
- **Deploying to Vercel (Free)** — Step-by-step deploy guide, what you get on the free tier
- **Deploying to Cloudflare Pages (Free)** — Step-by-step deploy guide, free tier details, comparison table vs Vercel

### `common-issues.md` — Troubleshooting Guide
Solutions for problems we actually hit:

- **Jammed URLs** — `<a>` tag stripping without spacing creates `https://example.com/Newsletter:https://other.com/`
- **Fullwidth vs halfwidth punctuation** — `：` vs `:` and `！` vs `!` breaking regex matches in CJK content
- **Percent-encoded filenames** — Chinese WordPress slugs like `%e5%81%a5%e8%ba%ab...`
- **Mixed guest/host boilerplate** — Show notes sections with both guest links (keep) and host links (remove)
- **WordPress embed wrappers** — `<div class="wp-block-embed__wrapper">` containing bare URLs instead of iframes
- **Broken nested markdown** — WordPress editors producing `[[link](url1) text](url2)`
- **WordPress styled callouts** — `has-background` class paragraphs getting stripped by Turndown
- **Figure/figcaption mangling** — `<figure>` elements losing captions during conversion
- **Cloudflare Pages vs Workers** — Why `npx wrangler deploy` fails for static sites and what to use instead
- **MDX parsing failures** — Why plain markdown wins for WordPress content
- **Content collection validation** — Making Zod schemas forgiving enough for messy WordPress data

## What This Skill Does NOT Handle

### Email

This migration covers your website only — it does not migrate email. If you're using WordPress-hosted email (e.g., `you@yourdomain.com` through your WordPress hosting provider), **you may need to keep your WordPress hosting account active** for email, or migrate email separately to:

- **Google Workspace** ($7/mo per user)
- **Zoho Mail** (free for up to 5 users)
- **Fastmail**, **Proton Mail**, or **iCloud+ Custom Domains**
- **Your domain registrar** — many (Cloudflare, Namecheap, Porkbun) offer email forwarding for free

If your email is already on a separate provider (Gmail, Outlook, etc.), you're fine — just update DNS records when you point the domain to Vercel/Cloudflare.

Check your current MX records (`dig MX yourdomain.com`) before canceling WordPress hosting to make sure you won't lose email.

## Real-World Results

This skill was built from:

- **[angiecreates.io](https://angiecreates.io)** — 84 blog posts + 80 podcast episodes, bilingual Chinese/English, Transistor.fm integration, Cloudflare Pages deployment. Special challenges: RSS-based episode generation, multilingual boilerplate removal, jammed URL fixes.

- **[strategyu.co](https://strategyu.co)** — 59 consulting articles, Vercel deployment, dynamic SVG featured images, YouTube/Vimeo embed preservation, ConvertKit integration.

## Contributing

Found a new WordPress quirk? Fixed a migration edge case? PRs welcome — especially for:
- New CMS source formats (Ghost, Squarespace, etc.)
- Additional hosting platform configs
- Embed types we haven't encountered
- Non-Latin language handling patterns
