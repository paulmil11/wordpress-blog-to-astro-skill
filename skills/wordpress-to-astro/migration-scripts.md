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
  bulletListMarker: '-',
  codeBlockStyle: 'fenced',
  emDelimiter: '*',
  strongDelimiter: '**',
  hr: '---',
});

// CUSTOMIZE: Add rules for your site's embed types
// Preserve iframes (YouTube, podcast players, etc.)
turndown.addRule('iframes', {
  filter: 'iframe',
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Preserve div wrappers containing iframes (responsive embed wrappers)
turndown.addRule('embedWrappers', {
  filter: (node) =>
    node.nodeName === 'DIV' &&
    node.querySelector('iframe') !== null,
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Preserve WordPress styled callout boxes as raw HTML
turndown.addRule('styledCallouts', {
  filter: (node) => {
    if (node.nodeName !== 'P') return false;
    const cls = node.getAttribute('class') || '';
    return cls.includes('has-background');
  },
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Preserve HTML tables (Turndown's table conversion is lossy)
turndown.addRule('tables', {
  filter: 'table',
  replacement: (content, node) => `\n\n${node.outerHTML}\n\n`,
});

// Convert <figure> with <img> to markdown image + optional caption
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

// Remove WordPress spacer blocks
turndown.addRule('wpSpacer', {
  filter: (node) =>
    node.nodeName === 'DIV' &&
    node.classList && node.classList.contains('wp-block-spacer'),
  replacement: () => '',
});

// Clean img tags to proper markdown format
turndown.addRule('cleanImg', {
  filter: 'img',
  replacement: (content, node) => {
    const src = node.getAttribute('src') || '';
    const alt = node.getAttribute('alt') || '';
    return `![${alt}](${src})`;
  },
});

// Strip WordPress block comments
turndown.addRule('wpComments', {
  filter: (node) =>
    node.nodeType === 8, // Comment node
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

## 7. Reading Time

```typescript
// src/utils/reading-time.ts — English version (238 words/min)
export function getReadingTime(content: string): string {
  const text = content.replace(/<[^>]*>/g, '');
  const words = text.trim().split(/\s+/).length;
  const minutes = Math.ceil(words / 238);
  return `${minutes} min read`;
}
```

For CJK/multilingual content, count characters separately:

```typescript
// CJK-aware version
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

## 8. Astro Page to Content Collection Converter

When migration was done in two stages (WordPress → Astro pages → content collections), use this script to convert `.astro` page files to `.md`:

```javascript
// convert-post.mjs
// Convert .astro blog post pages to content collection markdown
// Usage:
//   node scripts/convert-post.mjs src/pages/my-post.astro    # single
//   node scripts/convert-post.mjs --all                       # batch

import fs from 'fs';
import path from 'path';
import TurndownService from 'turndown';

function decodeEntities(str) {
  return str
    .replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>')
    .replace(/&quot;/g, '"').replace(/&#039;/g, "'").replace(/&nbsp;/g, ' ');
}

// Optional: load a posts.json for supplemental metadata (e.g. rawDate)
const postsJsonPath = path.resolve('src/data/posts.json');
let postsLookup = {};
if (fs.existsSync(postsJsonPath)) {
  for (const p of JSON.parse(fs.readFileSync(postsJsonPath, 'utf-8'))) {
    postsLookup[p.slug] = p;
  }
}

function parseAstroPost(filePath) {
  const raw = fs.readFileSync(filePath, 'utf-8');
  const fmMatch = raw.match(/^---\n([\s\S]*?)\n---/);
  if (!fmMatch) throw new Error('No frontmatter in ' + filePath);
  const fm = fmMatch[1];

  const getString = (name) => {
    const m = fm.match(new RegExp(`const\\s+${name}\\s*=\\s*["'\`]([\\s\\S]*?)["'\`]\\s*;`));
    return m ? m[1].replace(/\\"/g, '"').replace(/\\'/g, "'") : undefined;
  };
  const getArray = (name) => {
    const m = fm.match(new RegExp(`const\\s+${name}\\s*=\\s*\\[([^\\]]*?)\\]`));
    if (!m) return [];
    return m[1].split(',').map(s => s.trim().replace(/^["']|["']$/g, '')).filter(Boolean).map(decodeEntities);
  };

  // Extract HTML from content template literal
  const contentMatch = fm.match(/const\s+content\s*=\s*`([\s\S]*?)`\s*;/);
  if (!contentMatch) throw new Error('No content in ' + filePath);

  return {
    title: getString('title'), description: getString('description'),
    date: getString('date'), slug: getString('slug'),
    author: getString('author'), image: getString('image'),
    categories: getArray('categories'), html: contentMatch[1],
  };
}

// Pages that are NOT blog posts — skip during --all
const NON_BLOG_PAGES = new Set([
  'index', '404', 'team', 'subscribe', 'success', 'thank-you',
  // CUSTOMIZE: add your non-blog page filenames
]);

function isBlogPostFile(filePath) {
  const basename = path.basename(filePath, '.astro');
  if (NON_BLOG_PAGES.has(basename)) return false;
  const raw = fs.readFileSync(filePath, 'utf-8');
  return /const\s+content\s*=\s*`/.test(raw);
}

// Use Turndown with all custom rules (see section 2) to convert,
// then write markdown with frontmatter to src/content/posts/<slug>.md
```
