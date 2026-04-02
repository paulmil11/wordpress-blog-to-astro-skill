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

### Missing images are concentrated in a few posts

**Problem:** After running the image download script, many posts look "done" until you audit and discover broken images. The missing images aren't spread evenly — they're usually concentrated in a handful of heavily-edited or long-running posts that accumulated media over years of WordPress edits, plugin changes, and CDN reconfigurations.

**Solution:** Run an image audit script immediately after Phase 3 (not later). For every image reference in markdown, verify the local file exists and is a valid image (not a 0-byte file or HTML error page). Focus on fixing the posts with the most missing images first — this usually covers 80% of the problem. The long tail is often duplicate images or assets that were intentionally removed.

```bash
# Quick audit: find image refs that don't resolve to local files
grep -rn '!\[.*\](/images/' src/content/blog/ | while read line; do
  file=$(echo "$line" | grep -oP '/images/[^)]+')
  if [ ! -f "public$file" ]; then
    echo "MISSING: $file in $(echo "$line" | cut -d: -f1)"
  fi
done
```

### Legacy subdomain for post-cutover media recovery

**Problem:** After pointing the main domain to the new Astro site, you discover missing images that were served from `wp-content/uploads/` on the old WordPress host. You can't flip the root domain back without taking down the new site.

**Solution:** Point a temporary subdomain like `old.yoursite.com` at the old WordPress hosting server and map it to the original WordPress docroot. This gives you a working URL to fetch `wp-content/uploads/...` assets from:

```bash
# Recover a specific image
curl -o public/images/posts/photo.jpg https://old.yoursite.com/wp-content/uploads/2023/05/photo.jpg

# Batch recovery from a list of missing images
while read img; do
  filename=$(basename "$img")
  curl -o "public/images/posts/$filename" "https://old.yoursite.com$img"
done < missing-images.txt
```

This also lets you inspect legacy pages, check old redirects, and verify content without disrupting the live site. Remove the subdomain DNS record once recovery is complete.

### Featured image references to deleted files

**Problem:** WordPress frontmatter `featuredImage` fields reference files that were deleted from the media library or never downloaded. The image path looks valid but the file doesn't exist, resulting in broken thumbnails in related posts, OG images, and RSS feeds.

**Solution:** Add an asset existence check wherever featured images are used. At build time, verify the file exists in `public/` before rendering it:

```javascript
function assetExists(assetPath) {
  if (!assetPath.startsWith('/')) return true; // external URL, assume valid
  const localPath = path.join(process.cwd(), 'public', assetPath.replace(/^\//, ''));
  return fs.existsSync(localPath);
}
```

Use a fallback image (default OG image, category-specific placeholder, or auto-generated SVG) when the featured image is missing.

## WordPress Export vs Live Site

### Export is not a clean source of truth

**Problem:** The WordPress XML export captures a snapshot of the database, but it doesn't reflect the full reality of the live site. Old embeds survive in content long after the platform or URL pattern is obsolete. Inline images may reference CDN URLs that have since been reconfigured. Redirects configured in plugins aren't in the export. Third-party integrations (analytics, forms, search) need to be re-implemented.

**Solution:** Treat the export as a starting point, not the final word. Plan to reconcile exported content against the live site during multiple audit passes:
- **After Phase 2 (Convert)**: Spot-check 5-10 posts against the live site to catch systematic conversion issues
- **After Phase 3 (Images)**: Audit for missing images (see above)
- **After Phase 6 (Deploy)**: Full QA — forms, embeds, analytics, search, structured data
- **After launch**: Monitor Google Search Console for crawl errors and 404s that reveal gaps

## Deployment

### Cloudflare Workers
**Problem:** Old documentation suggests using Cloudflare Pages for static sites, leading to confusion when `npx wrangler deploy` tries to create a Worker.
**Solution:** Cloudflare now recommends **Workers** as the primary deployment target for both SSG and SSR. 
1. Run `npx wrangler deploy` in the root.
2. Wrangler auto-detects Astro and generates a `wrangler.jsonc`.
3. Static assets are served via the `assets` binding (e.g., `{ "directory": "./dist" }`) at edge speeds without using Worker CPU time unless requested.

### Middleware & "Run Worker First"
**Problem:** In the old Pages system, Functions (middleware) ran before static assets. In the new Workers system, static assets are served **first**, bypassing your Worker script and breaking authentication or custom redirects.
**Solution:** If you use middleware, authentication, or custom logic that must intercept asset requests, you must set `run_worker_first` to `true` in `wrangler.jsonc`.



```json
{
  "name": "my-astro-site",
  "compatibility_date": "2024-03-23", 
  "main": "./dist/_worker.js/index.js",
  "assets": {
    "directory": "./dist/client/",
    "run_worker_first": true
  }
}
```
## Local Dev with Bindings
**Problem:** Standard astro dev doesn't see Cloudflare KV, D1, or R2.
**Solution:** Use the `@astrojs/cloudflare` adapter. Running npx astro dev now utilizes the workerd runtime, giving you full access to production-like bindings locally.

See the Astro Guide for Cloudflare for more information https://docs.astro.build/en/guides/integrations-guide/cloudflare/

### Vercel redirects

**Pattern:** Use `vercel.json` with `redirects` array. Each entry needs `source`, `destination`, `permanent: true`.

### Netlify redirects

**Pattern:** Create `public/_redirects` file with `from to 301` format (one per line). This gets copied to `dist/` at build time.

## Cloudflare Redirects

Astro has a first-class, native routing engine that handles this out of the box. You can define your redirects directly inside your `astro.config.mjs` file.

When you use the `@astrojs/cloudflare` adapter, Astro automatically compiles these rules into the proper edge-compatible logic during your build process.

Here is what your `astro.config.mjs` would look like:

```JavaScript
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  output: 'server', // or 'hybrid'
  adapter: cloudflare(),
  redirects: {
    '/category/modern-careers/': '/categories/modern-careers/',
    '/about-us/': '/about/',
    // You can even set status codes explicitly
    '/old-blog': {
      status: 301,
      destination: '/new-blog'
    }
  }
});
```

## WordPress-Specific Gotchas

### WordPress styled callout boxes (`has-background`)

**Problem:** WordPress Gutenberg blocks with background colors (`<p class="has-background has-luminous-vivid-amber-background-color">`) get stripped to plain text by Turndown.

**Solution:** Add a Turndown rule to preserve these as raw HTML:

```javascript
turndown.addRule('styledCallouts', {
  filter: (node) => {
    if (node.nodeName !== 'P') return false;
    return (node.getAttribute('class') || '').includes('has-background');
  },
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});
```

Then style them in your blog post CSS to match the original appearance.

### Figure/figcaption not converting cleanly

**Problem:** `<figure>` elements with `<img>` and `<figcaption>` get mangled by Turndown, producing broken markdown or losing captions.

**Solution:** Add a custom rule to extract the image and caption separately:

```javascript
turndown.addRule('figure', {
  filter: 'figure',
  replacement: (content, node) => {
    const img = node.querySelector('img');
    if (!img) return content;
    const src = img.getAttribute('src') || '';
    const alt = img.getAttribute('alt') || '';
    const caption = node.querySelector('figcaption');
    const captionText = caption ? caption.textContent.trim() : '';
    if (captionText) return `\n\n![${alt}](${src})\n*${captionText}*\n\n`;
    return `\n\n![${alt}](${src})\n\n`;
  },
});
```

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

## Embed & URL Cleanup (Post-Migration)

### Protocol-relative URLs in embeds

**Problem:** Some WordPress embeds use protocol-relative URLs (`//www.youtube.com/embed/...` or `//www.slideshare.net/...`) which break when served from `file://` or don't render in some browsers.

**Solution:** Scan all markdown for `//www.` patterns in `src=` attributes and `href=` attributes. Replace with `https://www.`:
```bash
grep -rn '//www\.' src/content/blog/ | grep -v 'https://'
```
Fix each occurrence — usually in `<iframe src="//..."` and `<a href="//..."`.

### Bare Twitter/X URLs in blog content

**Problem:** WordPress content often contains bare Twitter URLs without markdown link formatting: `https://twitter.com/user/status/123456` sitting alone on a line or inline.

**Solution:** Convert bare Twitter URLs to markdown links. Use the tweet author's handle as the link text:
```
https://twitter.com/paulmmillerd/status/123456
→ [Paul Millerd on Twitter](https://twitter.com/paulmmillerd/status/123456)
```
Search with: `grep -rn 'https://twitter.com/' src/content/blog/ | grep -v '\[.*\](https://twitter'`

### Internal links pointing to old domain

**Problem:** After migrating from olddomain.com to newdomain.com, blog posts still contain internal links like `[link text](/old-page/)` or `[link text](https://olddomain.com/page/)` that point to content that no longer exists at those paths.

**Solution:** Systematic audit approach:
1. Grep all markdown for links to the old domain and relative internal links
2. Categorize each: (a) redirect to new domain equivalent, (b) redirect to external domain (e.g., podcast moved to separate site), (c) kill the link (keep text, remove `[text](url)` wrapper), (d) build a new page for it
3. Fix in batches — links to same destination can be done in parallel
4. For links to pages you need to build: use WebFetch to pull content from old site, then create clean Astro pages

### Responsive YouTube embed wrappers

**Problem:** YouTube iframes in migrated content render at fixed sizes or break on mobile. WordPress used `wp-block-embed__wrapper` divs that don't translate.

**Solution:** Use CSS `:has()` selector to target divs containing YouTube iframes:
```css
.post-content :global(div:has(> iframe[src*="youtube"])) {
  position: relative;
  width: 100%;
  max-width: 720px;
  margin: 2.5rem auto;
  padding-bottom: 56.25%; /* 16:9 */
  height: 0;
  overflow: hidden;
}
.post-content :global(div:has(> iframe[src*="youtube"]) iframe) {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  border: none;
}
```
This catches both WordPress-wrapped and bare `<div><iframe>` patterns.

### Stale build output (dist/ folder)

**Problem:** After creating new pages or making changes, checking `dist/` shows the page missing or old content. The dev server at localhost:4321 also may show stale content.

**Solution:** Always run a fresh `npx astro build` before checking `dist/`. The dev server auto-updates for most changes but occasionally needs a restart for new files. Don't trust stale `dist/` — rebuild.

## WordPress Redirection Plugin (Domain Migrations)

### Generating CSV for WordPress Redirection plugin

**Problem:** When migrating from olddomain.com to newdomain.com, you need the old WordPress site to redirect all URLs to the new domain. The WordPress Redirection plugin accepts CSV imports.

**Solution:** Generate a CSV with this header format:
```
source,target,regex,type,code,match,group
"/old-slug/","https://newdomain.com/old-slug/",0,url,301,url,Redirections
```
Include: all blog post slugs, special pages, homepage, category pages (note `/category/slug/` → `/categories/slug/` if path changed), RSS feed (`/feed/` → `/rss.xml`).

### Regex option missing in Redirection plugin

**Problem:** Some versions of the WordPress Redirection plugin don't show a regex checkbox in the UI, even after clicking the gear icon.

**Solution:** Skip regex rules entirely. Generate explicit redirect rules for every URL instead. This is actually more reliable — no risk of regex matching unintended URLs. A CSV with 200-500 explicit rules imports fine.

### Category path differences

**Problem:** WordPress uses `/category/slug/` but Astro generates `/categories/slug/` (or whatever you configured). Redirects need to account for this.

**Solution:** Include explicit category redirects in the CSV: `/category/modern-careers/` → `https://newdomain.com/categories/modern-careers/`. Don't rely on regex for this — list each category explicitly.

## Astro Build & Runtime Gotchas

### TypeScript syntax leaking into client-side scripts

**Problem:** Astro components let you write TypeScript in the frontmatter fence (`---`), but `<script>` tags in the template are shipped to the browser as-is by default. If you accidentally use TypeScript syntax (type annotations, `as` casts, interfaces) in a client `<script>` block, the build may succeed but the script silently fails in production browsers.

**Solution:** Keep client-side `<script>` blocks as plain JavaScript. If you need TypeScript in client code, use `<script lang="ts">` — Astro will transpile it. But for inline scripts (`<script is:inline>`), there's no transpilation — it must be valid browser JS.

### Hand-maintained search JSON drifts from content

**Problem:** If you create a search feature backed by a hand-maintained `search.json` or similar index file, it drifts out of sync as posts are added, edited, or removed. New posts don't appear in search, deleted posts return ghost results.

**Solution:** Generate search data from the content source at build time. Write a build script or Astro endpoint that reads the content collection and outputs the search index:

```javascript
// src/pages/search.json.ts
import { getCollection } from 'astro:content';

export async function GET() {
  const posts = await getCollection('blog', ({ data }) => !data.draft);
  const index = posts.map(p => ({
    slug: p.data.slug || p.slug,
    title: p.data.title,
    description: p.data.description || '',
    categories: p.data.categories || [],
  }));
  return new Response(JSON.stringify(index));
}
```

This way the search index is always aligned with the actual site content.
