# Common Issues & Troubleshooting

Real problems encountered across multiple WordPress-to-Astro migrations and how to solve them.

## HTML-to-Markdown Conversion

### Jammed URLs from `<a>` tag stripping

**Problem:** When stripping HTML tags from RSS or WordPress content, `<a>` tags get removed without adding spaces, causing adjacent text and URLs to jam together.

```
Before: Website: https://example.com/Newsletter: https://news.com/
Should be: Website: https://example.com/  Newsletter: https://news.com/
```

**Solution:** Convert `<a>` tags to `label: URL` format WITH surrounding spaces before stripping remaining HTML. Never just strip `<a>` tags — always add whitespace around the replacement.

### WordPress block comments in markdown

**Problem:** `<!-- wp:paragraph -->`, `<!-- wp:image -->` etc. survive Turndown conversion and appear in markdown.

**Solution:** Strip before Turndown: `html.replace(/<!--\s*\/?wp:.*?-->/gs, '')`

### Turndown fails on messy HTML

**Problem:** MDX or Markdoc parsers choke on WordPress HTML (unclosed tags, invalid nesting).

**Solution:** Use plain markdown with Content Collections, not MDX. Let raw HTML blocks pass through. Astro renders them fine in `.md` files.

## Multilingual / CJK Content

### Percent-encoded filenames

**Problem:** WordPress exports Chinese/CJK filenames as `%e5%81%a5%e8%ba%ab...`. Astro uses these as slugs, causing ugly URLs and potential build issues.

**Solution:** Create a rename script that decodes, generates ASCII-safe names, copies files (keep originals during transition), and updates all references. Or accept the encoded slugs — they work fine technically.

### Fullwidth vs halfwidth characters in regex

**Problem:** Regex patterns for Chinese text fail because of fullwidth punctuation variants.

```
了解更多：  (fullwidth ：)
了解更多:   (halfwidth :)
認識我們！  (fullwidth ！)
認識我們!   (halfwidth !)
```

**Solution:** Match both variants: `[：:]`, `[！!]`, `[（(]`, `[）)]`

### CJK reading time calculation

**Problem:** Standard word-count reading time is wrong for Chinese text (no spaces between words).

**Solution:** Count CJK characters separately (~350 chars/min) and Latin words (~238 words/min), then combine.

## Podcast / RSS Integration

### Episode number matching

**Problem:** Podcast titles use inconsistent numbering: `#80`, `EP80`, `ep.80`, `EP 80`, `#090`.

**Solution:** Flexible regex: `/(?:#|EP\.?\s?)(\d+)/i` — then compare parsed integers.

### Transistor.fm embed format

**Pattern:** `https://share.transistor.fm/e/{episode-path}` — the episode path comes from the RSS `<enclosure>` URL or can be derived from the episode page URL.

### Missing episodes

**Problem:** Blog posts exist for some episodes but not all.

**Solution:** Fetch RSS feed, compare episode numbers against existing filenames, auto-generate landing pages for missing ones with embed + description from RSS.

## Boilerplate Removal

### Mixed guest/host content in sections

**Problem:** Show notes sections contain both guest links (keep) and host boilerplate (remove) mixed together.

```markdown
## Connect with us!
- [Guest's Instagram](https://instagram.com/guest)    ← KEEP
- [Host's Instagram](https://instagram.com/host)       ← REMOVE
- [Host's Newsletter](https://host.substack.com)       ← REMOVE
```

**Solution:** Check each line individually against host URL patterns. Keep lines with non-host URLs, remove lines where ALL URLs match host patterns.

### Boilerplate section headers (multilingual)

**Problem:** Boilerplate section headers appear in multiple languages and formats:

```
## 了解更多：
### 認識我們！
**Connect with us!**
Contacts:
Follow Us:
```

**Solution:** Build a regex array matching all variants with optional markdown prefixes (`##`, `**`) and both colon types.

### Bare URLs not caught after conversion

**Problem:** Cleanup script checks for boilerplate, then `convertBareUrls` turns bare host URLs into markdown links, but those new links aren't re-checked.

**Solution:** Re-run boilerplate detection after URL conversion step.

## Image Handling

### WordPress CDN redirects

**Problem:** Image URLs redirect through multiple CDNs (e.g., `wp-content` → Jetpack CDN → final URL).

**Solution:** Follow redirects (up to 5 hops) in download script. Some CDNs require specific headers.

### Broken image paths after migration

**Problem:** WordPress uses `wp-content/uploads/YYYY/MM/` paths that don't map to Astro's `public/` directory.

**Solution:** Download all images to flat `public/images/posts/` directory, rewrite all references. Simpler than recreating the date-based directory structure.

## Deployment

### Cloudflare Pages vs Workers

**Problem:** Cloudflare's `npx wrangler deploy` command tries to set up a Workers project with `@astrojs/cloudflare` adapter, but a static Astro site doesn't need this.

**Solution:** For static sites, use Cloudflare Pages (not Workers). Set build command to `npm run build`, build output directory to `dist`, and leave the deploy command empty. If the deploy command field is required, use `exit 0`.

### Vercel redirects

**Pattern:** Use `vercel.json` with `redirects` array. Each entry needs `source`, `destination`, `permanent: true`.

### Netlify/Cloudflare Pages redirects

**Pattern:** Create `public/_redirects` file with `from to 301` format (one per line). This gets copied to `dist/` at build time.

## WordPress-Specific Gotchas

### GenerateBlocks / page builder wrappers

**Problem:** Page builders add deeply nested `<div>` wrappers that create noise in markdown.

**Solution:** Add Turndown rules to unwrap or strip specific container classes (e.g., `gb-container`, `wp-block-group`).

### WordPress embed wrappers

**Problem:** WordPress stores embeds as URLs inside `<div class="wp-block-embed__wrapper">` instead of `<iframe>`.

**Solution:** Pre-process HTML to convert these to proper `<iframe>` elements before Turndown, or add a Turndown rule.

### Nested/broken markdown from WordPress

**Problem:** Some WordPress editors produce broken nested markdown links: `[[link text](url1) more text](url2)`

**Solution:** Write extraction function that parses the inner links, discards the broken outer wrapper.

## Build Issues

### Astro build fails on special characters in filenames

**Problem:** Files with CJK characters, spaces, or special characters in filenames can cause build failures on some platforms.

**Solution:** Rename files to ASCII-safe slugs, or ensure your build environment supports UTF-8 filenames (most modern ones do).

### Content collection validation errors

**Problem:** Zod schema rejects posts with missing required fields.

**Solution:** Make most fields optional with defaults. Only `title` and `date` should be required. Use `.default()` for author, `.optional()` for everything else.
