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

### Phase 3: Download Images

```bash
node scripts/download-images.mjs
```

Scan all markdown files for external image URLs. Download to `public/images/posts/`. Rewrite URLs in markdown to local paths. Handle redirects (up to 5 hops), track failures separately. Idempotent — skips already-downloaded files.

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

### Phase 6: Build & Deploy

```bash
npm run build  # outputs to dist/
```

Verify build succeeds, check for broken links and missing images. Configure redirects for old WordPress URLs (platform-specific: `_redirects` for Netlify/Cloudflare, `vercel.json` for Vercel).

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

See supporting files for detailed patterns:
- `migration-scripts.md` — Script templates and implementation details
- `common-issues.md` — Troubleshooting guide from real migrations
- `post-migration-patterns.md` — SEO, featured images, CTAs, WordPress CSS compatibility, related posts, newsletter integration
