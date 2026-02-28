# Migration Script Templates

Reusable script patterns for WordPress-to-Astro migrations. Customize per project.

## 1. WordPress XML Parser

```javascript
// import-wordpress.js
// Parse WordPress XML export → extracted-posts.json
import { readFileSync, writeFileSync } from 'fs';
import { parseStringPromise } from 'xml2js';

const xml = readFileSync('export.xml', 'utf8');
const result = await parseStringPromise(xml);
const channel = result.rss.channel[0];

// Build attachment lookup (for featured images)
const attachments = {};
channel.item
  .filter(item => item['wp:post_type']?.[0] === 'attachment')
  .forEach(item => {
    attachments[item['wp:post_id'][0]] = item['wp:attachment_url'][0];
  });

// Extract published posts
const posts = channel.item
  .filter(item =>
    item['wp:post_type']?.[0] === 'post' &&
    item['wp:status']?.[0] === 'publish'
  )
  .map(item => {
    // Get featured image via _thumbnail_id postmeta
    const meta = item['wp:postmeta'] || [];
    const thumbMeta = meta.find(m => m['wp:meta_key']?.[0] === '_thumbnail_id');
    const thumbId = thumbMeta?.['wp:meta_value']?.[0];
    const featuredImage = thumbId ? attachments[thumbId] : null;

    // Extract categories and tags
    const cats = (item.category || [])
      .filter(c => c.$.domain === 'category')
      .map(c => c._);
    const tags = (item.category || [])
      .filter(c => c.$.domain === 'post_tag')
      .map(c => c._);

    return {
      title: item.title[0],
      slug: item['wp:post_name'][0],
      date: new Date(item['wp:post_date'][0]),
      content: item['content:encoded'][0],
      excerpt: item['excerpt:encoded']?.[0] || '',
      categories: cats,
      tags,
      image: featuredImage,
    };
  });

writeFileSync('extracted-posts.json', JSON.stringify(posts, null, 2));
console.log(`Extracted ${posts.length} posts`);
```

## 2. HTML to Markdown Converter (Turndown)

```javascript
// convert-posts.mjs
// Convert extracted posts to Astro content collection markdown
import TurndownService from 'turndown';
import { readFileSync, writeFileSync, mkdirSync } from 'fs';

const turndown = new TurndownService({
  headingStyle: 'atx',
  codeBlockStyle: 'fenced',
});

// CUSTOMIZE: Add rules for your site's embed types
// Preserve iframes (YouTube, podcast players, etc.)
turndown.addRule('iframes', {
  filter: 'iframe',
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Preserve div wrappers containing iframes
turndown.addRule('iframeWrappers', {
  filter: (node) =>
    node.nodeName === 'DIV' &&
    node.querySelector('iframe'),
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Preserve HTML tables (Turndown's table conversion is lossy)
turndown.addRule('tables', {
  filter: 'table',
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Strip WordPress block comments
turndown.addRule('wpComments', {
  filter: (node) =>
    node.nodeType === 8 && // Comment node
    node.data.trim().startsWith('wp:'),
  replacement: () => '',
});

const posts = JSON.parse(readFileSync('extracted-posts.json', 'utf8'));
mkdirSync('src/content/posts', { recursive: true });

for (const post of posts) {
  // Clean WordPress HTML before conversion
  let html = post.content
    .replace(/<!--\s*\/?wp:.*?-->/gs, '')     // Block comments
    .replace(/<div class="wp-block-spacer[^"]*"[^>]*><\/div>/g, '') // Spacers
    .replace(/<p>\s*<\/p>/g, '');              // Empty paragraphs

  const markdown = turndown.turndown(html);

  // Format date for display
  const dateObj = new Date(post.date);
  const displayDate = dateObj.toLocaleDateString('en-US', {
    year: 'numeric', month: 'long', day: 'numeric'
  });
  const rawDate = dateObj.toISOString().replace('T', ' ').slice(0, 19);

  // Truncate description
  const desc = (post.excerpt || post.title).replace(/\n/g, ' ').slice(0, 200);

  const frontmatter = [
    '---',
    `title: ${JSON.stringify(post.title)}`,
    `description: ${JSON.stringify(desc)}`,
    `date: "${displayDate}"`,
    `rawDate: "${rawDate}"`,
    `author: "Author Name"`, // CUSTOMIZE
    post.image ? `image: "${post.image}"` : null,
    post.categories.length
      ? `categories:\n${post.categories.map(c => `  - "${c}"`).join('\n')}`
      : null,
    '---',
  ].filter(Boolean).join('\n');

  writeFileSync(
    `src/content/posts/${post.slug}.md`,
    `${frontmatter}\n\n${markdown}\n`
  );
}

console.log(`Converted ${posts.length} posts to markdown`);
```

## 3. Image Downloader

```javascript
// download-images.mjs
// Download external images and rewrite URLs to local paths
import { readFileSync, writeFileSync, existsSync, mkdirSync } from 'fs';
import { join, basename } from 'path';
import https from 'https';
import http from 'http';
import { globSync } from 'fs';

const POSTS_DIR = 'src/content/posts';
const IMAGES_DIR = 'public/images/posts';
mkdirSync(IMAGES_DIR, { recursive: true });

// Extract all image URLs from markdown files
function extractImageUrls(content) {
  const urls = new Set();
  // Markdown images: ![alt](url)
  for (const m of content.matchAll(/!\[[^\]]*\]\(([^)]+)\)/g)) {
    if (m[1].startsWith('http')) urls.add(m[1]);
  }
  // HTML images: <img src="url">
  for (const m of content.matchAll(/<img[^>]+src="([^"]+)"/g)) {
    if (m[1].startsWith('http')) urls.add(m[1]);
  }
  // Frontmatter image
  for (const m of content.matchAll(/^image:\s*"(http[^"]+)"/gm)) {
    urls.add(m[1]);
  }
  return urls;
}

// Download with redirect following
function downloadFile(url, dest, maxRedirects = 5) {
  return new Promise((resolve, reject) => {
    const client = url.startsWith('https') ? https : http;
    client.get(url, (res) => {
      if (res.statusCode >= 300 && res.statusCode < 400 && res.headers.location) {
        if (maxRedirects <= 0) return reject(new Error('Too many redirects'));
        return downloadFile(res.headers.location, dest, maxRedirects - 1)
          .then(resolve).catch(reject);
      }
      if (res.statusCode !== 200) return reject(new Error(`HTTP ${res.statusCode}`));
      const chunks = [];
      res.on('data', c => chunks.push(c));
      res.on('end', () => {
        writeFileSync(dest, Buffer.concat(chunks));
        resolve();
      });
      res.on('error', reject);
    }).on('error', reject);
  });
}

// Process all posts
const files = globSync(`${POSTS_DIR}/*.md`);
const allUrls = new Map(); // url → local filename
const failures = [];

for (const file of files) {
  const content = readFileSync(file, 'utf8');
  for (const url of extractImageUrls(content)) {
    if (!allUrls.has(url)) {
      const name = basename(new URL(url).pathname).split('?')[0] || 'image.jpg';
      allUrls.set(url, name);
    }
  }
}

console.log(`Found ${allUrls.size} unique image URLs`);

for (const [url, filename] of allUrls) {
  const dest = join(IMAGES_DIR, filename);
  if (existsSync(dest)) continue;
  try {
    await downloadFile(url, dest);
    console.log(`  Downloaded: ${filename}`);
  } catch (err) {
    failures.push({ url, error: err.message });
    console.error(`  Failed: ${url} — ${err.message}`);
  }
}

// Rewrite URLs in markdown files
for (const file of files) {
  let content = readFileSync(file, 'utf8');
  let changed = false;
  for (const [url, filename] of allUrls) {
    if (failures.some(f => f.url === url)) continue;
    if (content.includes(url)) {
      content = content.replaceAll(url, `/images/posts/${filename}`);
      changed = true;
    }
  }
  if (changed) writeFileSync(file, content);
}

console.log(`Done. ${failures.length} failures.`);
```

## 4. HTML-to-Markdown for RSS Content

When generating pages from RSS feeds (not WordPress XML), use simpler conversion:

```javascript
function htmlToMarkdown(html) {
  if (!html) return '';
  let md = html;
  // Convert <a> tags — preserve both text and URL with spacing
  // CRITICAL: add spaces around to prevent URL jamming
  md = md.replace(/<a\s+[^>]*href="([^"]*)"[^>]*>([\s\S]*?)<\/a>/gi,
    (match, href, text) => {
      const cleanText = text.replace(/<[^>]+>/g, '').trim();
      if (!cleanText || /^https?:\/\//.test(cleanText)) return ` ${href} `;
      return ` ${cleanText}: ${href} `;
    });
  md = md.replace(/<li[^>]*>/gi, '\n- ');
  md = md.replace(/<\/li>/gi, '');
  md = md.replace(/<br\s*\/?>/gi, '\n');
  md = md.replace(/<\/p>/gi, '\n\n');
  md = md.replace(/<\/div>/gi, '\n');
  md = md.replace(/<\/h[1-6]>/gi, '\n\n');
  md = md.replace(/<[^>]+>/g, '');
  // HTML entities
  md = md.replace(/&amp;/g, '&').replace(/&lt;/g, '<')
    .replace(/&gt;/g, '>').replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'").replace(/&nbsp;/g, ' ');
  md = md.replace(/ {2,}/g, ' ').replace(/\n{3,}/g, '\n\n');
  return md.trim();
}
```

## 5. Boilerplate Cleanup Pattern

```javascript
// Pattern: Remove host/platform boilerplate while keeping guest content

const HOST_URL_PATTERNS = [
  /yourdomain\.com/i,
  /yourhandle/i,
  // CUSTOMIZE: add your own URLs/handles
];

function isHostBoilerplate(line) {
  const urls = [...line.matchAll(/https?:\/\/[^\s)>\]]+/g)].map(m => m[0]);
  const mdUrls = [...line.matchAll(/\]\(([^)]+)\)/g)].map(m => m[1]);
  const allUrls = [...urls, ...mdUrls].filter(u => /^https?:\/\//.test(u));
  if (allUrls.length === 0) return false;
  return allUrls.every(url => HOST_URL_PATTERNS.some(p => p.test(url)));
}

// Use in cleanup: skip lines that are host boilerplate
// but keep lines with guest URLs
```

## 6. Platform-Specific Redirects

```javascript
// Netlify/Cloudflare Pages: generate _redirects file
function generateRedirects(mapping) {
  const lines = mapping.map(({ from, to }) => `${from} ${to} 301`);
  writeFileSync('public/_redirects', lines.join('\n'));
}

// Vercel: generate vercel.json redirects
function generateVercelRedirects(mapping) {
  const redirects = mapping.map(({ from, to }) => ({
    source: from,
    destination: to,
    permanent: true,
  }));
  const config = JSON.parse(readFileSync('vercel.json', 'utf8') || '{}');
  config.redirects = redirects;
  writeFileSync('vercel.json', JSON.stringify(config, null, 2));
}
```

## 7. CJK-Aware Reading Time

```typescript
// src/utils/reading-time.ts
export function getReadingTime(content: string): string {
  const text = content.replace(/<[^>]*>/g, '');
  const cjkChars = (text.match(/[\u4e00-\u9fff\u3400-\u4dbf\uf900-\ufaff]/g) || []).length;
  const latinText = text.replace(/[\u4e00-\u9fff\u3400-\u4dbf\uf900-\ufaff]/g, '');
  const latinWords = latinText.trim().split(/\s+/).filter(w => w.length > 0).length;
  // CJK: ~350 chars/min, Latin: ~238 words/min
  const minutes = Math.ceil((cjkChars / 350) + (latinWords / 238));
  return `${Math.max(1, minutes)} 分鐘閱讀`;
}
```
