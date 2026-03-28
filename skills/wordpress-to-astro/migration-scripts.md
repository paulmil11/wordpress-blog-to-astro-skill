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

---

## Optional Templates

The following scripts are useful for specific migration scenarios but not needed for every project.

## 9. Restore Missing Images from WordPress XML (Optional)

Use when the image download script (section 3) didn't catch everything — images deleted from the media library, served by a reconfigured CDN, or lost during post edits after export. This script reconciles current markdown against the original WordPress XML to find and restore missing image blocks.

Three modes:
- **Default** — localize remote images (download from old server, rewrite URLs)
- **`--inject`** — compare source XML body with current markdown, inject missing image blocks in correct position
- **`--replace-body`** — nuclear option: replace entire body from XML source (preserves frontmatter)

```javascript
// restore-missing-images.mjs
// Usage:
//   node scripts/restore-missing-images.mjs <slug> [slug...]           # localize remote images
//   node scripts/restore-missing-images.mjs --inject <slug> [slug...]  # inject missing image blocks
//   node scripts/restore-missing-images.mjs --replace-body <slug>      # replace body from XML
import fs from 'fs';
import path from 'path';
import https from 'https';
import TurndownService from 'turndown';
import { XMLParser } from 'fast-xml-parser'; // or xml2js

// CUSTOMIZE: paths and old server details
const CONTENT_DIR = 'src/content/posts';
const IMAGES_DIR = 'public/images/posts';
const XML_FILES = ['export.xml']; // your WordPress XML export(s)
const OLD_HOST = 'old.yoursite.com'; // legacy subdomain pointing at old server
const OLD_IP = ''; // optional: old server IP if DNS is gone

// ── XML Parsing ──────────────────────────────────────────────────────────

const parser = new XMLParser({
  ignoreAttributes: false,
  cdataPropName: '__cdata',
  isArray: (name) => ['item', 'wp:postmeta', 'category'].includes(name),
});

function getCdata(val) {
  if (!val) return '';
  if (typeof val === 'string') return val;
  if (val.__cdata !== undefined) return val.__cdata;
  return String(val);
}

// ── Build source body map from XML ───────────────────────────────────────

const turndown = new TurndownService({ headingStyle: 'atx', codeBlockStyle: 'fenced' });
// Add your Turndown rules (iframes, figures, etc.) — see section 2

function buildSourceBodyMap() {
  const map = new Map();
  for (const xmlPath of XML_FILES) {
    const data = parser.parse(fs.readFileSync(xmlPath, 'utf8'));
    const items = data?.rss?.channel?.item || [];
    for (const item of items) {
      const slug = getCdata(item['wp:post_name']).trim();
      const postType = getCdata(item['wp:post_type']);
      const status = getCdata(item['wp:status']);
      if (!slug || postType !== 'post' || status !== 'publish') continue;
      const html = getCdata(item['content:encoded'])
        .replace(/<!--\s*\/?wp:.*?-->/gs, '');
      map.set(slug, turndown.turndown(html).trim());
    }
  }
  return map;
}

// ── Block-level comparison ───────────────────────────────────────────────

function splitBlocks(body) {
  return body.replace(/\r\n/g, '\n').trim().split(/\n{2,}/);
}

function isImageBlock(block) {
  const trimmed = block.trim();
  if (/^!\[[\s\S]*\]\([^)]+\)$/.test(trimmed)) return true;
  if (/^\[!\[[\s\S]*\]\([^)]+\)\]\([^)]+\)$/.test(trimmed)) return true;
  const lines = trimmed.split('\n').map(l => l.trim()).filter(Boolean);
  return lines.length > 0 && lines.every(l => l.startsWith('<img ') || l.startsWith('!['));
}

function normalizeBlock(block) {
  return block.replace(/\s+/g, ' ').trim();
}

// Fuzzy image name matching — ignore dimensions, dates, -copy suffixes
function normalizeImageName(value) {
  let name = path.basename(value).toLowerCase();
  try { name = decodeURIComponent(name); } catch {}
  return name
    .replace(/\.[a-z0-9]+$/i, '')  // strip extension
    .replace(/-\d+x\d+$/i, '')     // strip WP dimension suffix
    .replace(/^\d{4}-\d{2}-/, '')   // strip date prefix
    .replace(/-copy(?:-of)?/gi, '') // strip -copy
    .replace(/-\d+$/i, '')          // strip trailing number
    .replace(/[^a-z0-9]+/g, '');    // normalize separators
}

function extractImageUrls(content) {
  const urls = new Set();
  for (const m of content.matchAll(/!\[[^\]]*\]\(([^)]+)\)/g)) urls.add(m[1]);
  for (const m of content.matchAll(/<img[^>]+src="([^"]+)"/gi)) urls.add(m[1]);
  return [...urls];
}

// Check if an image block from the source already exists (by normalized name)
function hasEquivalentImage(block, currentBody) {
  const sourceNames = extractImageUrls(block).map(normalizeImageName).filter(Boolean);
  if (!sourceNames.length) return currentBody.includes(block.trim());
  const currentNames = extractImageUrls(currentBody).map(normalizeImageName);
  return sourceNames.every(s => currentNames.some(c => s === c || s.includes(c) || c.includes(s)));
}

// ── Inject missing image blocks ──────────────────────────────────────────
// Walks source blocks, finds image blocks missing from current content,
// inserts them next to the nearest matching text block (by content match).

function injectMissingImageBlocks(sourceBody, currentBody) {
  const sourceBlocks = splitBlocks(sourceBody);
  const currentBlocks = splitBlocks(currentBody);

  for (let i = 0; i < sourceBlocks.length; i++) {
    const block = sourceBlocks[i];
    if (!isImageBlock(block)) continue;
    if (hasEquivalentImage(block, currentBlocks.join('\n\n'))) continue;

    let inserted = false;

    // Try to insert after the preceding text block
    for (let j = i - 1; j >= 0; j--) {
      if (!sourceBlocks[j].trim() || isImageBlock(sourceBlocks[j])) continue;
      const idx = currentBlocks.findIndex(c => normalizeBlock(c) === normalizeBlock(sourceBlocks[j]));
      if (idx !== -1) { currentBlocks.splice(idx + 1, 0, block); inserted = true; break; }
    }
    if (inserted) continue;

    // Fall back: insert before the following text block
    for (let j = i + 1; j < sourceBlocks.length; j++) {
      if (!sourceBlocks[j].trim() || isImageBlock(sourceBlocks[j])) continue;
      const idx = currentBlocks.findIndex(c => normalizeBlock(c) === normalizeBlock(sourceBlocks[j]));
      if (idx !== -1) { currentBlocks.splice(idx, 0, block); inserted = true; break; }
    }

    if (!inserted) currentBlocks.push(block); // last resort: append
  }

  return currentBlocks.join('\n\n').trim() + '\n';
}

// ── Download from old server ─────────────────────────────────────────────
// CUSTOMIZE: adjust host/IP for your old WordPress server

function fetchFromOldServer(urlPath) {
  return new Promise((resolve, reject) => {
    const options = OLD_IP
      ? { host: OLD_IP, servername: OLD_HOST, headers: { Host: OLD_HOST } }
      : { host: OLD_HOST };
    https.get({ ...options, path: urlPath, port: 443, rejectUnauthorized: false },
      (res) => {
        if (res.statusCode >= 300 && res.statusCode < 400 && res.headers.location) {
          return fetchFromOldServer(new URL(res.headers.location).pathname).then(resolve, reject);
        }
        if (res.statusCode !== 200) return reject(new Error(`HTTP ${res.statusCode}`));
        const chunks = [];
        res.on('data', c => chunks.push(c));
        res.on('end', () => resolve(Buffer.concat(chunks)));
      }).on('error', reject);
  });
}

// ── Main ─────────────────────────────────────────────────────────────────

const args = process.argv.slice(2);
const inject = args.includes('--inject');
const replaceBody = args.includes('--replace-body');
const slugs = args.filter(a => !a.startsWith('--'));

if (!slugs.length) {
  console.error('Usage: node restore-missing-images.mjs [--inject|--replace-body] <slug> [slug...]');
  process.exit(1);
}

fs.mkdirSync(IMAGES_DIR, { recursive: true });
const sourceBodies = (inject || replaceBody) ? buildSourceBodyMap() : new Map();

for (const slug of slugs) {
  const filePath = path.join(CONTENT_DIR, `${slug}.md`);
  if (!fs.existsSync(filePath)) { console.warn(`Skip: ${slug} not found`); continue; }

  let raw = fs.readFileSync(filePath, 'utf8');
  const fmEnd = raw.indexOf('\n---\n', 4);
  const frontmatter = raw.slice(0, fmEnd + 5);
  const body = raw.slice(fmEnd + 5).trimStart();

  if (replaceBody && sourceBodies.has(slug)) {
    raw = frontmatter + sourceBodies.get(slug) + '\n';
    fs.writeFileSync(filePath, raw);
    console.log(`${slug}: body replaced from XML source`);
  } else if (inject && sourceBodies.has(slug)) {
    const merged = injectMissingImageBlocks(sourceBodies.get(slug), body);
    fs.writeFileSync(filePath, frontmatter + merged);
    console.log(`${slug}: injected missing image blocks`);
  }

  // Localize any remaining remote images
  const remoteUrls = extractImageUrls(raw).filter(u => u.startsWith('http'));
  for (const url of remoteUrls) {
    const filename = path.basename(new URL(url).pathname);
    const dest = path.join(IMAGES_DIR, filename);
    if (fs.existsSync(dest)) continue;
    try {
      const buf = await fetchFromOldServer(new URL(url).pathname);
      fs.writeFileSync(dest, buf);
      raw = fs.readFileSync(filePath, 'utf8').replaceAll(url, `/images/posts/${filename}`);
      fs.writeFileSync(filePath, raw);
      console.log(`  Downloaded: ${filename}`);
    } catch (e) { console.error(`  Failed: ${url} — ${e.message}`); }
  }
}
```

Key concepts:
- **Block-level diffing** — splits markdown into double-newline-separated blocks, matches text blocks between source and current, finds image blocks that exist in source but not in current
- **Fuzzy image name matching** — normalizes filenames by stripping dimensions (`-800x600`), dates, `-copy` suffixes to detect equivalent images even when WordPress generated multiple sizes
- **Position inference** — inserts recovered images next to the nearest matching text block (looks backward first, then forward) to preserve reading flow
- **Three modes** — default just localizes remote URLs; `--inject` does the smart merge; `--replace-body` is the nuclear option when a post is too mangled to patch

## 10. RSS Feed to Content Stubs (Optional)

Use when your WordPress site has a podcast, newsletter archive, or other external feed with episodes/issues that don't have corresponding blog posts. This script fetches the feed, compares against existing content, and generates markdown stubs for missing entries.

```javascript
// generate-feed-stubs.mjs
// Generate landing pages from an RSS feed for entries without blog posts.
// Usage: node scripts/generate-feed-stubs.mjs
import fs from 'fs';
import path from 'path';
import https from 'https';
import { parseStringPromise } from 'xml2js';

// CUSTOMIZE: your feed URL and content directory
const POSTS_DIR = 'src/content/posts';
const RSS_URL = 'https://feeds.transistor.fm/your-podcast-feed';

// ── Fetch RSS ────────────────────────────────────────────────────────────

function fetchRSS(url) {
  return new Promise((resolve, reject) => {
    https.get(url, { timeout: 30000 }, (res) => {
      if (res.statusCode >= 300 && res.statusCode < 400 && res.headers.location) {
        return fetchRSS(res.headers.location).then(resolve).catch(reject);
      }
      if (res.statusCode !== 200) return reject(new Error(`HTTP ${res.statusCode}`));
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve(data));
    }).on('error', reject);
  });
}

// ── Extract episode/entry number from title ──────────────────────────────

function extractEntryNumber(title) {
  // Match: #80, EP80, EP.80, Episode 80, Issue 12, Vol. 3, etc.
  // CUSTOMIZE: adjust patterns for your feed's naming convention
  const match = title.match(/(?:#|EP\.?\s?|Episode\s?|Issue\s?|Vol\.?\s?)(\d+)/i);
  return match ? parseInt(match[1], 10) : null;
}

// ── Simple HTML to text ──────────────────────────────────────────────────

function htmlToText(html) {
  if (!html) return '';
  return html
    .replace(/<a\s+[^>]*href="([^"]*)"[^>]*>([\s\S]*?)<\/a>/gi, (_, href, text) => {
      const clean = text.replace(/<[^>]+>/g, '').trim();
      return clean && !/^https?:\/\//.test(clean) ? `[${clean}](${href})` : ` ${href} `;
    })
    .replace(/<li[^>]*>/gi, '\n- ')
    .replace(/<br\s*\/?>/gi, '\n')
    .replace(/<\/p>/gi, '\n\n')
    .replace(/<[^>]+>/g, '')
    .replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>')
    .replace(/&quot;/g, '"').replace(/&#39;/g, "'").replace(/&nbsp;/g, ' ')
    .replace(/ {2,}/g, ' ').replace(/\n{3,}/g, '\n\n')
    .trim();
}

// ── Build slug from title ────────────────────────────────────────────────

function titleToSlug(title) {
  return title.toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '')
    .substring(0, 80);
}

// ── Find existing entry numbers in blog posts ────────────────────────────

function getExistingEntries() {
  const files = fs.readdirSync(POSTS_DIR).filter(f => f.endsWith('.md'));
  const numbers = new Set();
  const slugs = new Set();

  for (const file of files) {
    const content = fs.readFileSync(path.join(POSTS_DIR, file), 'utf-8');
    const titleMatch = content.match(/^title:\s*["']?(.+?)["']?\s*$/m);
    if (titleMatch) {
      const num = extractEntryNumber(titleMatch[1]);
      if (num !== null) numbers.add(num);
    }
    const slugMatch = content.match(/^slug:\s*["']?(.+?)["']?\s*$/m);
    if (slugMatch) slugs.add(slugMatch[1]);

    // CUSTOMIZE: detect existing embeds for your platform
    if (content.includes('share.transistor.fm') || content.includes('open.spotify.com/embed/episode')) {
      const fileNum = file.match(/(\d+)/);
      if (fileNum) numbers.add(parseInt(fileNum[1], 10));
    }
  }

  return { numbers, slugs };
}

// ── Build embed URL from feed item ───────────────────────────────────────

function getEmbedUrl(item) {
  const link = item.link?.[0] || '';
  // CUSTOMIZE: adjust for your podcast platform (Transistor, Anchor, etc.)
  if (link.includes('transistor.fm')) {
    try {
      const episodePath = new URL(link).pathname.replace(/^\//, '');
      return `https://share.transistor.fm/e/${episodePath}`;
    } catch { return ''; }
  }
  return link;
}

// ── Main ─────────────────────────────────────────────────────────────────

async function main() {
  const rssXml = await fetchRSS(RSS_URL);
  const result = await parseStringPromise(rssXml);
  const items = result.rss.channel[0].item || [];

  console.log(`Found ${items.length} items in RSS feed.`);

  const { numbers, slugs } = getExistingEntries();
  console.log(`Found ${numbers.size} existing entries in blog posts.\n`);

  let created = 0;

  for (const item of items) {
    const title = item.title?.[0] || '';
    const entryNum = extractEntryNumber(title);

    // Skip if we already have content for this entry
    if (entryNum !== null && numbers.has(entryNum)) continue;

    const embedUrl = getEmbedUrl(item);
    if (!embedUrl) { console.log(`  [skip] No embed: "${title}"`); continue; }

    const pubDate = new Date(item.pubDate?.[0] || '').toISOString().split('T')[0];
    const summary = item['itunes:summary']?.[0] || item.description?.[0] || '';
    const description = htmlToText(summary).substring(0, 200);
    const bodyHtml = item['content:encoded']?.[0] || item.description?.[0] || '';
    const bodyText = htmlToText(bodyHtml).split('\n').filter(l => l.trim()).slice(0, 30).join('\n\n');

    // CUSTOMIZE: slug prefix for your content type
    const slug = entryNum !== null ? `podcast-ep${entryNum}` : `podcast-${titleToSlug(title)}`;
    const filePath = path.join(POSTS_DIR, `${slug}.md`);

    if (fs.existsSync(filePath) || slugs.has(slug)) continue;

    const safeTitle = title.replace(/"/g, '\\"');
    const safeDesc = description.replace(/"/g, '\\"');

    // CUSTOMIZE: frontmatter fields, categories, embed HTML for your platform
    const markdown = `---
title: "${safeTitle}"
slug: "${slug}"
date: "${pubDate}"
description: "${safeDesc}"
categories:
  - Podcast
podcastEmbed: "${embedUrl}"
---

<iframe src="${embedUrl}" width="100%" height="180" frameborder="0" scrolling="no" seamless="true" style="width:100%;height:180px;"></iframe>

${bodyText}
`;

    fs.writeFileSync(filePath, markdown);
    console.log(`  [create] ${slug}`);
    created++;
  }

  console.log(`\nCreated ${created} new stubs.`);
}

main().catch(console.error);
```

Key concepts:
- **Entry number extraction** — flexible regex matching `#80`, `EP80`, `Episode 80`, etc. Customize for your feed's naming convention
- **Duplicate detection** — checks both parsed entry numbers AND existing embed URLs to avoid creating stubs for episodes that already have blog posts (even if titled differently)
- **HTML-to-text with link preservation** — critical: adds spaces around `<a>` tag replacements to prevent URL jamming (see common-issues.md)
- **Slug generation** — number-prefixed slugs (`podcast-ep80`) when possible, title-based fallback otherwise
- **Platform-agnostic** — works with Transistor, Anchor, Spotify, or any podcast platform that provides an RSS feed. Customize `getEmbedUrl()` for your embed format
