# Post-Migration Patterns

Reusable patterns for building out the Astro site after content migration is complete. These cover SEO, components, styling, and monetization.

## 1. SEO Meta Tags (Layout.astro)

Set up the base layout with full OG, Twitter, and structured data support:

```astro
---
interface Props {
  title: string;
  description?: string;
  image?: string;
  type?: 'website' | 'article';
  publishedDate?: string;
  modifiedDate?: string;
  author?: string;
  noindex?: boolean;
}

const {
  title,
  description = "Site description here",
  image = "/images/og-default.jpg",
  type = "website",
  publishedDate,
  modifiedDate,
  author = "Author Name",
  noindex = false,
} = Astro.props;

const canonicalURL = new URL(Astro.url.pathname, Astro.site);
const imageURL = new URL(image, Astro.site);
---

<head>
  <!-- Primary -->
  <title>{title}</title>
  <meta name="description" content={description} />
  <meta name="author" content={author} />
  <link rel="canonical" href={canonicalURL} />
  {noindex && <meta name="robots" content="noindex, nofollow" />}

  <!-- Open Graph -->
  <meta property="og:type" content={type} />
  <meta property="og:url" content={canonicalURL} />
  <meta property="og:title" content={title} />
  <meta property="og:description" content={description} />
  <meta property="og:image" content={imageURL} />
  <meta property="og:site_name" content="Site Name" />

  <!-- Twitter -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content={title} />
  <meta name="twitter:description" content={description} />
  <meta name="twitter:image" content={imageURL} />

  <!-- Article meta (blog posts only) -->
  {type === 'article' && publishedDate && (
    <meta property="article:published_time" content={publishedDate} />
  )}
  {type === 'article' && modifiedDate && (
    <meta property="article:modified_time" content={modifiedDate} />
  )}

  <!-- JSON-LD -->
  <script type="application/ld+json" set:html={JSON.stringify({
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Site Name",
    "url": "https://example.com",
    "logo": "https://example.com/images/logo.png",
    // CUSTOMIZE: add founder, sameAs (social links), etc.
  })} />
</head>
```

Key points:
- `canonicalURL` and `imageURL` constructed from `Astro.site` (set in `astro.config.mjs`)
- `noindex` prop for pages that shouldn't be indexed (thank-you, download, etc.)
- Article-specific meta only renders when `type === 'article'`

## 2. Auto-Generated Featured Images

Generate SVG featured images from post titles when no custom image exists. Avoids needing to create a unique image for every migrated post.

**Pattern: FeaturedImage component**

```astro
---
interface Props {
  title: string;
  slug: string;
  variant?: 'dark' | 'light';
}

const { title, slug, variant = 'dark' } = Astro.props;

// 1. Map specific posts to existing graphics
const postGraphics: Record<string, string> = {
  'post-about-topic': '/images/frameworks/topic.png',
  // CUSTOMIZE: map slugs to existing images
};
const customGraphic = postGraphics[slug];

// 2. Detect icon type from title keywords
const lowerTitle = title.toLowerCase();
let iconType = 'default';
if (lowerTitle.includes('keyword1')) iconType = 'tree';
else if (lowerTitle.includes('keyword2')) iconType = 'pyramid';
// CUSTOMIZE: add keyword → icon mappings

// 3. Colors
const bgColor = variant === 'dark' ? '#272323' : '#FDFCFC';
const textColor = variant === 'dark' ? '#FDFCFC' : '#272323';
const accentColor = '#A22731'; // CUSTOMIZE: your brand color

// 4. Wrap title text for SVG (no CSS word-wrap in SVG)
function wrapText(text: string, maxChars: number, maxLines: number): string[] {
  const words = text.split(' ');
  const lines: string[] = [];
  let currentLine = '';
  for (const word of words) {
    if (lines.length >= maxLines) break;
    const testLine = currentLine ? `${currentLine} ${word}` : word;
    if (testLine.length <= maxChars) { currentLine = testLine; }
    else {
      if (currentLine) { lines.push(currentLine); currentLine = word; }
      else { lines.push(word.substring(0, maxChars - 3) + '...'); currentLine = ''; }
    }
  }
  if (currentLine && lines.length < maxLines) lines.push(currentLine);
  return lines;
}

const titleLines = wrapText(title, 32, 3);
---

{customGraphic ? (
  <div class="custom-graphic">
    <img src={customGraphic} alt={title} loading="lazy" />
  </div>
) : (
  <svg viewBox="0 0 400 200" xmlns="http://www.w3.org/2000/svg">
    <rect width="400" height="200" fill={bgColor}/>
    <!-- Icon in corner based on iconType -->
    <g transform="translate(320, 140)">
      {/* CUSTOMIZE: SVG icons per type */}
    </g>
    <!-- Title text -->
    <text x="24" y="70" fill={textColor} font-size="16" font-weight="600">
      {titleLines.map((line, i) => (
        <tspan x="24" dy={i === 0 ? "0" : "22"}>{line}</tspan>
      ))}
    </text>
    <!-- Branding watermark -->
    <text x="24" y="175" fill={textColor} font-size="10" opacity="0.4">SITENAME</text>
  </svg>
)}
```

Key points:
- Priority: custom mapped image → auto-generated SVG
- SVG viewBox 400x200 (2:1 aspect ratio) works well for OG images and cards
- Text wrapping must be done manually in SVG (no CSS word-wrap)
- Keyword detection maps post topics to relevant icons

## 3. Category-Matched CTAs with UTM Tracking

Drive readers to relevant content based on post categories:

```astro
---
// In BlogPost.astro layout
const ctaMap: Record<string, { text: string; path: string }> = {
  'Category A': { text: 'Watch the free intro', path: '/lessons/category-a-intro' },
  'Category B': { text: 'Try the workshop', path: '/workshops/category-b' },
  // CUSTOMIZE: map categories to CTAs
};
const defaultCta = { text: 'Check out our courses', path: '/courses' };

const matchedKey = categories.find((c: string) => ctaMap[c]);
const courseCta = matchedKey ? ctaMap[matchedKey] : defaultCta;

// UTM params for attribution
const ctaHref = `https://learn.example.com${courseCta.path}?utm_source=blog&utm_medium=post&utm_campaign=${slug}`;
---

<div class="course-cta-box">
  <p class="course-cta-label">Go deeper on this topic</p>
  <a href={ctaHref} class="course-cta-link" target="_blank">
    {courseCta.text} &rarr;
  </a>
</div>
```

## 4. Related Posts Component

Find related posts by category overlap, fall back to recent posts:

```astro
---
import { getCollection } from 'astro:content';

interface Props {
  currentSlug: string;
  currentCategories?: string[];
  maxPosts?: number;
}

const { currentSlug, currentCategories = [], maxPosts = 3 } = Astro.props;

const allPosts = await getCollection('posts', ({ data }) => {
  return import.meta.env.PROD ? !data.draft : true;
});

// Category overlap matching (case-insensitive partial match)
const relatedPosts = allPosts
  .filter(post => {
    if (post.slug === currentSlug) return false;
    if (currentCategories.length === 0) return true;
    const postCats = post.data.categories || [];
    return postCats.some(cat =>
      currentCategories.some(cc =>
        cat.toLowerCase().includes(cc.toLowerCase()) ||
        cc.toLowerCase().includes(cat.toLowerCase())
      )
    );
  })
  .slice(0, maxPosts);

// Fallback: recent posts if no category matches
const displayPosts = relatedPosts.length > 0
  ? relatedPosts
  : allPosts.filter(p => p.slug !== currentSlug).slice(0, maxPosts);
---
```

Key: uses partial string matching for categories so "MECE" matches "MECE Examples" etc.

## 5. Newsletter Integration (Lazy-Loaded)

Load email signup scripts only when the form scrolls into view:

```astro
<div class="newsletter-box">
  <h3>Want to learn more?</h3>
  <p>Join our free mini-course.</p>
  <form action="https://app.kit.com/forms/FORM_ID/subscriptions"
        method="post" data-sv-form="FORM_ID">
    <input type="email" name="email_address"
           placeholder="Email Address" required />
    <button type="submit">Get Free Course</button>
  </form>
</div>

<script>
  // Lazy-load ConvertKit JS only when newsletter box enters viewport
  const form = document.querySelector('.newsletter-box');
  if (form) {
    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const s = document.createElement('script');
          s.src = 'https://f.convertkit.com/ckjs/ck.5.js';
          s.async = true;
          document.head.appendChild(s);
          observer.disconnect();
        }
      });
    }, { rootMargin: '200px' }); // Start loading 200px before visible
    observer.observe(form);
  }
</script>
```

Works with ConvertKit/Kit. Adapt the form action URL and script src for other providers. The 200px rootMargin preloads slightly before the user scrolls to the form.

## 6. WordPress CSS Compatibility

WordPress-migrated content often contains raw HTML with WordPress classes. Add these styles to the blog post layout to handle them:

```css
/* WordPress alignment classes */
.post-content :global(.aligncenter) {
  display: block;
  margin-left: auto;
  margin-right: auto;
}
.post-content :global(.alignleft) {
  float: left;
  margin: 0.5rem 1.5rem 1rem 0;
  max-width: 50%;
}
.post-content :global(.alignright) {
  float: right;
  margin: 0.5rem 0 1rem 1.5rem;
  max-width: 50%;
}

/* Clear floats after paragraphs */
.post-content :global(p)::after {
  content: "";
  display: table;
  clear: both;
}

/* WordPress size classes */
.post-content :global(.size-large img),
.post-content :global(.size-full img) { max-width: 100%; }
.post-content :global(.size-medium img) { max-width: min(100%, 400px); }

/* Constrain unsized images for readability */
.post-content :global(img:not([width])) { max-width: min(100%, 680px); }

/* Hide WordPress spacer blocks */
.post-content :global(.wp-block-spacer) { display: none; }

/* Figure captions */
.post-content :global(figcaption),
.post-content :global(.wp-caption-text) {
  font-size: 0.8125rem;
  color: var(--color-text-muted);
  text-align: center;
  margin-top: 0.75rem;
  font-style: italic;
}

/* Responsive video embeds (YouTube, Vimeo) */
.post-content :global(.wp-block-embed),
.post-content :global(.video-container),
.post-content :global(div:has(> iframe[src*="youtube"])),
.post-content :global(figure:has(iframe[src*="youtube"])) {
  position: relative;
  width: 100%;
  max-width: 720px;
  margin: 2.5rem auto;
  padding-bottom: 56.25%; /* 16:9 aspect ratio */
  height: 0;
  overflow: hidden;
  border-radius: 12px;
  background: #000;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.2);
}

.post-content :global(.wp-block-embed iframe),
.post-content :global(.video-container iframe),
.post-content :global(div:has(> iframe[src*="youtube"]) iframe) {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  border: none;
}

/* Tables from WordPress */
.post-content :global(table) {
  width: 100%;
  border-collapse: collapse;
  margin: 2rem 0;
}
.post-content :global(th),
.post-content :global(td) {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid var(--color-bg-alt);
}
.post-content :global(th) {
  font-weight: 600;
  background: var(--color-bg-alt);
}
.post-content :global(tr:hover td) {
  background: rgba(228, 225, 225, 0.5);
}

/* Code blocks */
.post-content :global(pre) {
  background: var(--color-dark);
  color: #f8f8f2;
  padding: 1.25rem 1.5rem;
  border-radius: 4px;
  overflow-x: auto;
  margin: 2rem 0;
}
.post-content :global(:not(pre) > code) {
  background: var(--color-bg-alt);
  padding: 0.15rem 0.4rem;
  border-radius: 3px;
  color: var(--color-primary);
}

/* Blockquotes */
.post-content :global(blockquote) {
  border-left: 4px solid var(--color-primary);
  padding: 1rem 1.5rem;
  margin: 2rem 0;
  background: var(--color-bg-alt);
  font-style: italic;
  border-radius: 0 8px 8px 0;
}
```

Key insight: Astro scopes styles by default. Use `:global()` to target rendered markdown content and WordPress HTML classes that survive migration.

## 7. Dynamic Route with Draft Filtering

```astro
---
// src/pages/[slug].astro
import { getCollection } from 'astro:content';
import BlogPost from '../layouts/BlogPost.astro';

export async function getStaticPaths() {
  const posts = await getCollection('posts', ({ data }) => {
    return import.meta.env.PROD ? !data.draft : true; // Show drafts in dev only
  });
  return posts.map((post) => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

const { post } = Astro.props;
const { Content } = await post.render();
---

<BlogPost
  title={post.data.title}
  description={post.data.description}
  date={post.data.date}
  slug={post.slug}
  author={post.data.author}
  categories={post.data.categories}
  content={post.body}
>
  <Content />
</BlogPost>
```

Note: Posts live at root level (`/my-post-slug`) matching WordPress default permalink structure. If WordPress used `/blog/slug/`, add redirects or change the route to `src/pages/blog/[slug].astro`.

## 8. robots.txt

```
User-agent: *
Allow: /

Disallow: /api/
Disallow: /_astro/
Disallow: /keystatic/

Sitemap: https://example.com/sitemap-index.xml
```

Place in `public/robots.txt`. The `/_astro/` path contains Astro's hashed static assets — no need to index those. Point the sitemap to `sitemap-index.xml` (generated by `@astrojs/sitemap`).

## 9. Performance: DNS Prefetch for Third-Party Scripts

```html
<link rel="dns-prefetch" href="https://us-assets.i.posthog.com" />
<link rel="dns-prefetch" href="https://connect.facebook.net" />
<link rel="dns-prefetch" href="https://f.convertkit.com" />
```

Add `dns-prefetch` (and `preconnect` for critical ones) for any third-party domains your analytics, email, or ad scripts load from. This shaves 50-100ms off their first request.

## 10. Cross-Linking & Internal SEO

WordPress sites often have weak internal linking. Migration is a good time to fix that.

### Automated "Related Posts" by category

See section 4 above. This is the minimum — every post should link to 2-3 others.

### In-content contextual links

Write a script to scan all posts and insert links where one post mentions another's topic:

```javascript
// scripts/cross-link-posts.mjs
// Scan posts for mentions of other post titles/keywords, suggest internal links
import { readFileSync, readdirSync } from 'fs';
import path from 'path';

const POSTS_DIR = 'src/content/posts';
const files = readdirSync(POSTS_DIR).filter(f => f.endsWith('.md'));

// Build lookup: keyword → slug
const posts = files.map(f => {
  const content = readFileSync(path.join(POSTS_DIR, f), 'utf8');
  const titleMatch = content.match(/^title:\s*"(.+)"/m);
  const title = titleMatch ? titleMatch[1] : '';
  const slug = f.replace('.md', '');
  return { slug, title, file: f };
});

// For each post, find mentions of other post titles
for (const post of posts) {
  const content = readFileSync(path.join(POSTS_DIR, post.file), 'utf8');
  const body = content.split('---').slice(2).join('---'); // skip frontmatter

  for (const other of posts) {
    if (other.slug === post.slug) continue;
    // Check if body mentions the other post's key terms but doesn't already link to it
    const alreadyLinked = body.includes(`/${other.slug}`);
    const mentioned = body.toLowerCase().includes(other.title.toLowerCase().split(':')[0].trim());
    if (mentioned && !alreadyLinked) {
      console.log(`${post.slug} mentions "${other.title}" but doesn't link to /${other.slug}`);
    }
  }
}
```

Run this after migration to find cross-linking opportunities. Add links manually — automated insertion is too risky for quality.

### Other cross-linking strategies

- **"See also" blocks**: Add a short list at the end of sections that reference related posts. More targeted than the generic related posts grid.
- **Pillar/cluster model**: Pick 3-5 pillar topics. Each pillar page links to all posts in that cluster. Each cluster post links back to the pillar. Good for SEO topical authority.
- **Category index pages**: Build `/blog/category/[category].astro` pages that list all posts in a category. Gives Google a crawl path and users a browse path.
- **Breadcrumbs**: Add `Home > Blog > Category > Post Title` breadcrumbs. Helps both navigation and structured data (Google shows them in search results).
- **Previous/Next navigation**: Add chronological prev/next links at the bottom of each post. Keeps readers moving through the archive.

### SEO quick wins post-migration

- **Submit sitemap to Google Search Console**: `https://example.com/sitemap-index.xml` — do this immediately after launch.
- **Check for broken internal links**: Run `npx broken-link-checker https://example.com --recursive` after deploy.
- **Add `alt` text to all images**: WordPress exports often have empty alt tags. Scan for `![](` (empty alt) and fix.
- **Description meta for every post**: If WordPress excerpts were empty, generate descriptions from the first 1-2 sentences.
- **Canonical URLs**: Already handled in the Layout.astro pattern (section 1). Critical if old WordPress URLs are still live during transition.
- **JSON-LD Article schema**: For blog posts, add Article structured data beyond the Organization schema:

```astro
{type === 'article' && (
  <script type="application/ld+json" set:html={JSON.stringify({
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": title,
    "description": description,
    "image": imageURL,
    "datePublished": publishedDate,
    "dateModified": modifiedDate || publishedDate,
    "author": { "@type": "Person", "name": author },
    "publisher": {
      "@type": "Organization",
      "name": "Site Name",
      "logo": { "@type": "ImageObject", "url": "https://example.com/logo.png" }
    },
    "mainEntityOfPage": canonicalURL,
  })} />
)}
```

## 11. Redirects from Old WordPress URLs

This is critical. Without redirects, you lose all SEO juice from existing Google rankings and inbound links. Set up redirects **before** pointing the domain to the new host.

### Generating the redirect map

WordPress URLs typically follow patterns like `/2024/01/post-slug/` or `/blog/post-slug/`. Astro URLs are usually `/post-slug`. Write a script to generate the mapping:

```javascript
// scripts/generate-redirects.mjs
import { readFileSync, writeFileSync, readdirSync } from 'fs';
import path from 'path';

const POSTS_DIR = 'src/content/posts';
const files = readdirSync(POSTS_DIR).filter(f => f.endsWith('.md'));

const redirects = [];

for (const file of files) {
  const content = readFileSync(path.join(POSTS_DIR, file), 'utf8');
  const slug = file.replace('.md', '');
  const rawDateMatch = content.match(/^rawDate:\s*"(\d{4})-(\d{2})/m);

  // WordPress date-based permalink: /YYYY/MM/slug/
  if (rawDateMatch) {
    const [, year, month] = rawDateMatch;
    redirects.push({ from: `/${year}/${month}/${slug}/`, to: `/${slug}` });
  }

  // WordPress /blog/slug/ pattern
  redirects.push({ from: `/blog/${slug}/`, to: `/${slug}` });

  // Trailing slash variant
  redirects.push({ from: `/${slug}/`, to: `/${slug}` });
}

// Also redirect common WordPress paths
redirects.push({ from: '/feed/', to: '/rss.xml' });
redirects.push({ from: '/feed/atom/', to: '/rss.xml' });
redirects.push({ from: '/wp-login.php', to: '/' });
redirects.push({ from: '/wp-admin', to: '/' });
redirects.push({ from: '/wp-admin/*', to: '/' });

console.log(`Generated ${redirects.length} redirects`);
```

### Option A: Vercel (`vercel.json`)

```json
{
  "redirects": [
    { "source": "/2024/01/my-post/", "destination": "/my-post", "permanent": true },
    { "source": "/blog/:slug/", "destination": "/:slug", "permanent": true },
    { "source": "/feed/(.*)", "destination": "/rss.xml", "permanent": true }
  ]
}
```

Vercel supports wildcard `:slug` and regex `(.*)` patterns, so you can often cover everything with a few rules rather than listing every post. `permanent: true` sends a 301.

### Option B: Cloudflare Pages (`public/_redirects`)

```
/2024/01/my-post/ /my-post 301
/blog/:slug /slug 301
/feed/* /rss.xml 301
/wp-login.php / 301
/wp-admin/* / 301
```

Place in `public/_redirects` — gets copied to `dist/` at build time. Cloudflare Pages supports `:splat` and `:slug` placeholders. One redirect per line: `from to status`.

### Option C: Netlify (`public/_redirects`)

Same format as Cloudflare Pages. Netlify also supports a `netlify.toml` alternative:

```toml
[[redirects]]
  from = "/blog/:slug/"
  to = "/:slug"
  status = 301
```

### Option D: DNS-level redirects (for domain changes)

If moving from `olddomain.com` to `newdomain.com`, you need the old domain to redirect too. Options:
- **Cloudflare**: Add old domain, use Page Rules or Bulk Redirects to send everything to `https://newdomain.com/$1`
- **Vercel**: Add old domain as an alias, it auto-redirects to the primary domain
- **Simple redirect service**: Services like redirect.pizza handle this if you just need domain-level forwarding

### Testing redirects

After deploy, verify redirects work:

```bash
# Spot-check a few URLs
curl -sI https://example.com/2024/01/old-post-slug/ | grep -i location
curl -sI https://example.com/blog/old-post-slug/ | grep -i location
curl -sI https://example.com/feed/ | grep -i location

# Bulk check from a file of old URLs
while read url; do
  status=$(curl -sI "$url" | head -1 | awk '{print $2}')
  echo "$status $url"
done < old-urls.txt
```

## 12. Deploying to Vercel (Free)

Vercel's free Hobby plan includes unlimited static sites, custom domains, automatic HTTPS, global CDN, and preview deploys for every git push. More than enough for a migrated blog.

### Steps

1. **Push your repo to GitHub** (or GitLab/Bitbucket).

2. **Sign up at vercel.com** using your GitHub account.

3. **Import your repo**: Click "Add New Project" → select your repo. Vercel auto-detects Astro.

4. **Configure build settings** (usually auto-detected):
   - Framework Preset: Astro
   - Build Command: `npm run build`
   - Output Directory: `dist`

5. **Deploy**: Click "Deploy". First build takes 1-2 minutes.

6. **Add custom domain**: Go to Project Settings → Domains. Add your domain. Vercel gives you DNS records to add at your registrar (either an A record or CNAME). HTTPS is automatic.

7. **Add redirects**: Create `vercel.json` in project root with your redirect rules (see section 11).

### What you get for free

- Unlimited bandwidth for static sites
- Automatic deploys from git push
- Preview URLs for every branch/PR
- Global CDN (edge network)
- Free SSL/HTTPS
- Serverless Functions (100GB-hours/month) if you ever need them
- Web analytics (basic, free tier)

### vercel.json minimal config

```json
{
  "redirects": [
    { "source": "/blog/:slug/", "destination": "/:slug", "permanent": true },
    { "source": "/feed/(.*)", "destination": "/rss.xml", "permanent": true }
  ]
}
```

## 13. Deploying to Cloudflare Pages (Free)

Cloudflare Pages free tier is also genuinely free for static sites — unlimited bandwidth, unlimited requests, 500 builds/month, custom domains, automatic HTTPS.

### Steps

1. **Push your repo to GitHub** (or GitLab).

2. **Sign up at dash.cloudflare.com** (free account).

3. **Create a Pages project**: Go to Workers & Pages → Create → Pages → Connect to Git.

4. **Select your repo and configure**:
   - Production branch: `main`
   - Framework preset: Astro
   - Build command: `npm run build`
   - Build output directory: `dist`

5. **Deploy**: Click "Save and Deploy". First build takes 2-3 minutes.

6. **Add custom domain**: Go to your Pages project → Custom domains → Set up a domain. If your domain is already on Cloudflare DNS, it auto-configures. Otherwise, add the CNAME record they provide.

7. **Add redirects**: Create `public/_redirects` with your redirect rules (see section 11).

**Important**: Use Cloudflare **Pages**, not Workers. A static Astro site doesn't need the `@astrojs/cloudflare` adapter or `wrangler deploy`. Those are for server-rendered Astro. Pages is simpler and correct for static output.

### What you get for free

- Unlimited bandwidth (no caps)
- Unlimited requests
- 500 builds per month
- Global CDN (Cloudflare's edge network — 300+ cities)
- Free SSL/HTTPS
- Preview deploys for every branch
- Web analytics (free, privacy-focused)
- If domain is on Cloudflare: free DDoS protection, caching rules, page rules

### Cloudflare vs Vercel — which to pick

| | Vercel | Cloudflare Pages |
|---|---|---|
| **Bandwidth** | Unlimited (static) | Unlimited |
| **Builds** | 6000/month | 500/month |
| **CDN** | Fast, global | Fast, global (slightly more PoPs) |
| **Custom domains** | Easy | Easy (best if DNS already on Cloudflare) |
| **Redirects** | `vercel.json` (supports regex) | `_redirects` file (simpler syntax) |
| **Preview deploys** | Yes | Yes |
| **Serverless** | Yes (free tier) | Yes (Workers, free tier) |
| **Best for** | Quick setup, Vercel ecosystem | Sites already on Cloudflare DNS |

Both are genuinely free for static sites. Pick based on where your DNS lives or personal preference.
