# WordPress to Astro Migration

Migrate WordPress sites to Astro static sites with content collections, localized images, and clean markdown.

## When to Use

Invoke with `/wordpress-to-astro` when:
- Starting a new WordPress-to-Astro migration
- Resuming a migration that's partially complete
- Cleaning up migrated content (boilerplate, broken links, embeds)
- Generating missing content from RSS feeds

## Migration Pipeline

Run these phases in order. Each phase is idempotent — safe to re-run.

### Phase 0: Export from WordPress

Get the XML export file from the WordPress admin:

- **Self-hosted WordPress**: Go to **Tools → Export** (`/wp-admin/export.php`). Select "Posts" (or "All content" if you want pages too). Click "Download Export File". You'll get a `.xml` file.
- **WordPress.com**: Go to **Settings → General → Export** or navigate to `https://your-site.wordpress.com/wp-admin/export.php`. Same flow — select content type, download the XML.
- **WP Engine / Managed hosts**: Same Tools → Export path. Some managed hosts also offer database exports, but the XML export is what the migration scripts expect.

The export file contains all posts, pages, categories, tags, comments, custom fields, and media attachment metadata. It does NOT contain the actual image files — those get downloaded in Phase 3.

If the site has many posts (500+), WordPress may split the export into multiple XML files. Process each one separately or concatenate them.

**Important:** A WordPress export is only a starting point, not a clean source of truth. Expect to reconcile exported content with live site behavior, old embeds, redirects, missing media, search indexes, and third-party integrations throughout the migration.

### Phase 1: Extract (WordPress XML → JSON)

```bash
node scripts/import-wordpress.js
```

Parse the WordPress XML export. Extract published posts with metadata (title, slug, date, categories, tags, featured images). Build an attachment lookup for featured images via `_thumbnail_id` postmeta.

### Phase 2: Convert (HTML → Markdown)

```bash
node scripts/convert-posts.mjs
```

Convert WordPress HTML to Astro-compatible markdown using Turndown. Apply custom rules to preserve embeds (YouTube, Substack, podcast players) as raw HTML. Strip WordPress block comments (`<!-- wp:xxx -->`), spacer blocks, and GenerateBlocks wrappers. Generate YAML frontmatter with dual date fields (`date` for display, `rawDate` for sorting).

### Phase 3: Download & Audit Images

```bash
node scripts/download-images.mjs
```

Scan all markdown files for external image URLs. Download to `public/images/posts/`. Rewrite URLs in markdown to local paths. Handle redirects (up to 5 hops), track failures separately. Idempotent — skips already-downloaded files.

**Image audit (run immediately after download):**

Images are consistently the hardest part of any WordPress migration. The download script catches URLs that are still live, but many images will be missing — deleted from the WordPress media library, served by a CDN that's since been reconfigured, or referenced by posts that were edited after export. Run an audit immediately:

1. Grep all markdown for image references (both `![](url)` and `<img src="url">`)
2. Check that every referenced file actually exists in `public/images/posts/`
3. Check that every `featuredImage` in frontmatter points to a real file
4. Log missing images with the post slug so you can fix them in batches

Missing images tend to be concentrated in a few heavily-edited posts rather than spread evenly. Fix the biggest posts first — that usually covers 80% of the problem.

If images are missing because the WordPress host is no longer serving them, see the **legacy subdomain recovery** pattern in `common-issues.md`.

### Phase 4: Clean Up Content

```bash
node scripts/cleanup-podcast-notes.mjs  # podcast-specific
node scripts/cleanup-categories.mjs      # standardize categories
node scripts/fix-curiousbarbell-links.mjs # domain migration
node scripts/fix-embed-wrappers.mjs       # WordPress embed wrappers
node scripts/fix-percent-encoded-images.mjs # CJK filenames
```

Run cleanup scripts for boilerplate removal, category standardization, domain link rewrites, embed fixes, and percent-encoded filename normalization.

### Phase 5: Generate Missing Content

```bash
node scripts/generate-missing-episodes.mjs  # from RSS feed
node scripts/add-podcast-embeds.mjs          # inject players
```

Auto-generate landing pages for content that exists in external feeds (e.g., podcast RSS) but has no blog post. Match episodes by number, inject embed players.

### Phase 6: Build, Redirects & Deploy

```bash
npm run build  # outputs to dist/
```

Verify build succeeds, check for broken links and missing images.

**Redirects** — Critical for preserving SEO. Redirects matter as much as content migration — the site can look perfect and still fail badly after launch if old WordPress URLs, category archives, series pages, and homepage redirects are not mapped carefully. Generate a redirect map from old WordPress URLs (e.g., `/2024/01/slug/` or `/blog/slug/`) to new Astro URLs (`/slug`). Don't forget `/feed/` → `/rss.xml` and `/wp-admin` → `/`. See `post-migration-patterns.md` section 11 for the full redirect generation script and platform-specific formats.

For **domain migrations** (old-domain.com → new-domain.com), also generate a redirect file the user can import into the old site's WordPress Redirection plugin (CSV format). Include blog posts, special pages, categories (`/category/` → `/categories/`), RSS feed, and homepage. See `post-migration-patterns.md` section 14 for the WordPress Redirection plugin CSV format.

**Deploy** — Both Vercel and Cloudflare Pages are free for static sites (unlimited bandwidth, custom domains, automatic HTTPS, global CDN). See `post-migration-patterns.md` sections 12-13 for step-by-step deploy guides for each platform.

**Legacy subdomain for post-cutover media recovery** — After pointing the main domain to the new host, you may discover missing images or need to reference old pages. Instead of flipping the root domain back to WordPress, point a temporary subdomain like `old.yoursite.com` at the old WordPress host and map it to the original docroot. This lets scripts fetch `wp-content/uploads/...` assets and you can inspect legacy pages without disrupting the live site. Remove the subdomain once recovery is complete.

### Phase 7: Post-Migration Polish

After the content is migrated and building, there's always a design/polish phase:

1. **Embed cleanup** — Fix protocol-relative URLs (`//www.youtube.com` → `https://www.youtube.com`), malformed iframes, broken embed wrappers. Fix bare URLs that should be markdown links (especially Twitter/X URLs).

2. **Internal link audit** — Scan all posts for internal links pointing to old domains or dead paths. Fix systematically: some become external links to other domains, some get killed (link removed, text kept), some point to new pages you need to build.

3. **Build standalone pages** — Recreate important non-blog pages from the old site (about, coaching, tools, reading lists, etc.). Use WebFetch to pull content from the old site, then build clean Astro pages. Download hero images.

4. **Design testing page** — Create a `/test/` page that renders every component, typography size, color, and layout pattern in one place. Use it to iterate on the design system before building real pages. Remove before launch.

5. **Blog index features** — Build an interactive blog page with: inline JS category filtering (buttons, not links), search, sort by reading time, show more pagination, "I'm feeling lucky" random post button, blog stats (post count, word count, reading time, years writing).

6. **Dark mode** — Implement via CSS custom properties and a `[data-theme="dark"]` attribute on `<html>`. Toggle button in header. Persist preference to localStorage. Redefine all colors in a `[data-theme="dark"]` block.

7. **New categories** — Add new blog categories to posts as needed (YAML array format in frontmatter). Category pages auto-generate via `[category].astro` with slugified names.

8. **Gradient/visual polish** — Smooth section transitions (vertical gradient fades), consistent max-widths across sections, responsive mobile layouts.

9. **Build-time search index** — If adding site search, generate the search index from content at build time rather than maintaining a separate JSON file by hand. Hand-maintained search data drifts quickly as posts are added or edited. A build script that reads the content collection and outputs the index stays aligned automatically.

10. **Full QA pass** — Static-site migrations still need real application QA. Forms, search, analytics, embeds, and structured metadata all need testing after the content itself is migrated. Expect a final polish pass after launch-quality is reached — that is where the subtle problems usually show up.

**CRITICAL RULE**: Never shorten, paraphrase, or edit the user's original blog content. All content edits must preserve the author's exact words. Only fix structural issues (links, embeds, frontmatter).

## Key Architecture Decisions

| Decision | Why |
|----------|-----|
| Content Collections over JSON | Frontmatter collocated with content, type-safe with Zod |
| Plain Markdown, not MDX | WordPress HTML is messy; MDX parsing too strict |
| Static output | Fastest loads, no server costs, simplest deployment |
| GitHub as CMS | Edit markdown directly, version control built-in |
| Single `[slug].astro` | One template for all posts, no duplication |
| Dual date fields | `date` for display, `rawDate` for reliable sorting |

## Content Collection Schema

```typescript
const posts = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    description: z.string().optional(),
    date: z.string(),
    rawDate: z.string().optional(),
    author: z.string().default('Author Name'),
    image: z.string().optional(),
    categories: z.array(z.string()).optional(),
    tags: z.array(z.string()).optional(),
    draft: z.boolean().default(false),
  }),
});
```

## Project Structure

```
project/
├── scripts/
│   ├── import-wordpress.js          # XML → JSON
│   ├── convert-posts.mjs            # HTML → Markdown (Turndown)
│   ├── download-images.mjs          # Localize external images
│   ├── cleanup-podcast-notes.mjs    # Remove show notes boilerplate
│   ├── cleanup-categories.mjs       # Standardize categories
│   ├── fix-embed-wrappers.mjs       # WordPress embeds → iframes
│   ├── fix-percent-encoded-images.mjs # CJK filename normalization
│   ├── generate-missing-episodes.mjs  # RSS → landing pages
│   └── add-podcast-embeds.mjs       # Inject podcast players
├── src/
│   ├── content/
│   │   ├── config.ts                # Zod schema
│   │   └── posts/                   # Markdown files
│   ├── pages/
│   │   ├── [slug].astro             # Dynamic post route
│   │   ├── blog/index.astro         # Blog listing
│   │   ├── rss.xml.ts               # RSS feed
│   │   └── index.astro              # Homepage
│   ├── layouts/
│   │   ├── Layout.astro             # Base layout (head, meta, CSS)
│   │   └── BlogPost.astro           # Post template
│   ├── components/
│   │   ├── Header.astro
│   │   ├── Footer.astro
│   │   ├── FeaturedImage.astro      # Auto-generated SVG featured images
│   │   └── RelatedPosts.astro       # Category-based recommendations
│   └── utils/
│       └── reading-time.ts          # CJK-aware reading time
├── public/
│   └── images/posts/                # Localized images
├── astro.config.mjs
└── package.json
```

## Dependencies

```json
{
  "astro": "^4.0.0",
  "@astrojs/rss": "^4.0.15",
  "@astrojs/sitemap": "^3.2.1",
  "turndown": "^7.2.2",
  "xml2js": "^0.6.2"
}
```

Optional: `@playform/compress` for automatic asset optimization.

## Customization Points

When starting a migration, ask the user about:

1. **Source platform**: WordPress.com vs self-hosted (affects XML structure)
2. **Hosting target**: Cloudflare Pages, Vercel, Netlify (affects redirect config and deploy)
3. **Content types**: Blog only? Podcast? Events? (determines which scripts to create)
4. **Languages**: Monolingual or multilingual? (affects filename handling, reading time)
5. **Old domain**: Need redirects from old URLs? (create redirect mapping)
6. **External feeds**: Podcast RSS, newsletter? (auto-generation scripts)
7. **Embeds**: YouTube, podcast players, Instagram? (Turndown custom rules)

8. **Dark mode**: Does the user want light/dark theme toggle? (CSS custom properties + `[data-theme]` approach)
9. **Blog features**: Category filtering, search, sort, stats? (inline JS on blog index)

See supporting files for detailed patterns:
- `migration-scripts.md` — Script templates and implementation details
- `common-issues.md` — Troubleshooting guide from real migrations
- `post-migration-patterns.md` — SEO, featured images, CTAs, WordPress CSS compatibility, related posts, newsletter integration, dark mode, blog filtering, domain redirect CSV generation
