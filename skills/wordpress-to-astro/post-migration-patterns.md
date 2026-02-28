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
