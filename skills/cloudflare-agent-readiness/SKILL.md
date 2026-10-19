---
name: cloudflare-agent-readiness
description: Make a Cloudflare Pages site pass Cloudflare's Agent Readiness audit on the free plan — Content Signals in robots.txt, Markdown content negotiation via a Pages Function, Link response headers, and an Agent Skills Discovery Index. Load when a Cloudflare Pages site needs to pass the AI Crawl Control audit or when adding agent-facing surfaces to a static site.
---

# Cloudflare Agent Readiness — Free-Plan Recipe for Pages Sites

> **This skill makes a Cloudflare Pages site pass the Agent Readiness audit without upgrading to Pro.** It covers four audit items that apply to a static marketing or blog site: Content Signals, Markdown Negotiation, Link Response Headers, and the Agent Skills Discovery Index. It deliberately skips the 10+ items in the audit that require infrastructure a static site doesn't have (APIs, auth, agent/MCP servers, commerce protocols).

**Version:** v1.0 — 2026-10-18 (initial release)
**Audit tool:** `POST https://isitagentready.com/api/scan` with `{"url": "https://your-site.example"}`

---

## When to load this skill

Load when any of these are true:

- You run a Cloudflare Pages site (static or SSR) and want to pass Cloudflare's Agent Readiness audit
- You want AI crawlers and agents to retrieve clean markdown instead of parsing your rendered HTML
- You publish reusable agent skills or policies and want them discoverable at a well-known URL
- You want to declare a stance on AI training vs. AI retrieval of your content

**Do not load** if the site is a public API, auth-protected service, agent/MCP server, or commerce platform — those need their own audit items (OAuth metadata, A2A agent card, MCP server card, ACP/AP2/MPP/UCP/x402) that are outside this skill's scope.

## Audit items this skill covers

Four items, all free, all passable without upgrading from the Free plan:

| Item | File it adds | Dashboard setting required? |
|---|---|---|
| Content Signals | `public/robots.txt` (one new line) | No |
| Markdown Negotiation | `functions/_middleware.js` + `public/index.md` + optional per-post generator | No |
| Link Response Headers | `public/_headers` | No |
| Agent Skills Discovery Index | `public/.well-known/agent-skills/index.json` + SKILL.md files | No |

## Skips — explain-and-document, don't implement

These audit items will show as failing after you ship this skill. That is correct for a static site. Document the skip in your own notes:

- **API Catalog, OAuth/OIDC Discovery, OAuth Protected Resource Metadata, Auth.md** — require public APIs or auth
- **A2A Agent Card, MCP Server Card** — require an agent or MCP server on the domain
- **Web Bot Auth** — for sites that OPERATE outbound bots, not destinations
- **WebMCP** — requires interactive site tools (search, booking, forms) to expose
- **DNS-AID** — requires actual agent endpoints to point DNS at
- **ACP, AP2, MPP, UCP, x402** — commerce/payment protocols for merchant APIs

If the site later ships any of that infrastructure, revisit the audit. Until then, treat these as "N/A, by design."

## Step 1 — Content Signals

Add one line to `public/robots.txt` under `User-agent: *`:

```
User-agent: *
Allow: /
Content-Signal: search=yes, ai-train=no, ai-input=yes
```

**Values:**
- `search=yes` — allow traditional search indexing
- `ai-train=no` — do NOT consent to your content being baked into model weights
- `ai-input=yes` — allow AI tools to retrieve your content at inference time (RAG, grounding, cited answers)

**Why these defaults for a content-forward site:** `ai-input=yes` enables ChatGPT/Perplexity/Claude citations with attribution and backlinks — the AEO (Answer Engine Optimization) play. `ai-train=no` denies silent absorption into generic model outputs. If your content is pure IP and you want to monetize AI access separately, set `ai-input=no`; if you don't care about training usage, set `ai-train=yes`.

**Spec:** Content Signals Policy (Cloudflare, Sept 2025) — https://contentsignals.org. Omitting a signal is "no stated preference," not "no."

**Validate:** `POST https://isitagentready.com/api/scan` with `{"url": "https://your-site.example"}` — expect `checks.botAccessControl.contentSignals.status === "pass"`.

## Step 2 — Markdown Negotiation

Cloudflare's audit probes the homepage with `Accept: text/markdown` and expects a `text/markdown` response. On Free plan (no "Markdown for Agents" feature), roll your own:

### 2a. Pages Function middleware

`functions/_middleware.js`:

```javascript
export const onRequest = async ({ request, next, env }) => {
  const accept = request.headers.get('Accept') || ''
  if (!accept.includes('text/markdown')) return next()

  const url = new URL(request.url)
  let mdPath = url.pathname
  if (mdPath === '/' || mdPath === '') mdPath = '/index.md'
  else mdPath = mdPath.replace(/\/$/, '') + '.md'

  try {
    const mdRes = await env.ASSETS.fetch(new Request(new URL(mdPath, url.origin)))
    if (!mdRes.ok) return next()
    const body = await mdRes.text()
    return new Response(body, {
      headers: {
        'Content-Type': 'text/markdown; charset=utf-8',
        'Vary': 'Accept',
        'Cache-Control': 'public, max-age=3600',
      },
    })
  } catch {
    return next()
  }
}
```

Pages auto-discovers `functions/`. No `wrangler.toml` needed.

### 2b. Homepage markdown

Hand-author `public/index.md`. Content should mirror what a human sees on the homepage: title, hero line, pages list, latest content pointers, agent-oriented resource list, author identity. ~50-80 lines. Think of it as the human-friendly homepage condensed to pure markdown.

### 2c. Per-post markdown (optional but recommended)

If the site has a blog, add a build-time script that converts each MDX/Markdown post to a flat `.md` file under `public/<path>/<slug>.md`. The middleware already routes `/<path>/<slug>` → `/<path>/<slug>.md` when the Accept header matches.

Example `scripts/generate-post-markdown.mjs` (for MDX source):

```javascript
import { readFileSync, writeFileSync, readdirSync, mkdirSync, existsSync } from 'node:fs'
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __dirname = dirname(fileURLToPath(import.meta.url))
const root = join(__dirname, '..')
const postsDir = join(root, 'src/posts')
const outDir = join(root, 'public/blog')
if (!existsSync(outDir)) mkdirSync(outDir, { recursive: true })

function toMarkdown(mdx) {
  let text = mdx.replace(/\r\n/g, '\n')
  const fm = text.match(/^---\n([\s\S]*?)\n---\n?/)
  let title = '', date = ''
  if (fm) {
    const t = fm[1].match(/^title:\s*["']?(.*?)["']?\s*$/m)
    const d = fm[1].match(/^date:\s*["']?(.*?)["']?\s*$/m)
    title = t ? t[1].trim() : ''
    date = d ? d[1].trim() : ''
    text = text.slice(fm[0].length)
  }
  text = text.replace(/<img\b([^>]*?)\/?>/g, (_f, attrs) => {
    const s = (attrs.match(/\bsrc=["']([^"']*)["']/) || [])[1] || ''
    const a = (attrs.match(/\balt=["']([^"']*)["']/) || [])[1] || ''
    return `![${a}](${s})`
  })
  text = text.replace(/<figure[^>]*>([\s\S]*?)<\/figure>/g, (_f, inner) => {
    const cap = (inner.match(/<figcaption[^>]*>([\s\S]*?)<\/figcaption>/) || [])[1] || ''
    const img = inner.replace(/<figcaption[^>]*>[\s\S]*?<\/figcaption>/, '').trim()
    return `${img}${cap ? `\n\n*${cap.trim()}*` : ''}`
  })
  return (title ? `# ${title}\n${date ? `\n*${date}*\n` : ''}\n` : '') + text.trim() + '\n'
}

for (const file of readdirSync(postsDir).filter((f) => f.endsWith('.mdx'))) {
  const slug = file.replace(/\.mdx$/, '')
  writeFileSync(join(outDir, `${slug}.md`), toMarkdown(readFileSync(join(postsDir, file), 'utf8')))
}
```

Wire it into `package.json`:

```json
"scripts": {
  "build": "npm run post-markdown && npm run skills-index && vite build",
  "post-markdown": "node scripts/generate-post-markdown.mjs",
  "skills-index": "node scripts/generate-skills-index.mjs"
}
```

**Validate:** The audit only probes the homepage. Pass requires `Content-Type: text/markdown` and a markdown-shaped body (frontmatter or H1 prefix). `Vary: Accept` is recommended.

## Step 3 — Link Response Headers

Add `public/_headers`:

```
/
  Link: </sitemap.xml>; rel="sitemap"
  Link: </robots.txt>; rel="describedby"
  Link: </.well-known/agent-skills/index.json>; rel="agent-skills"
```

Only add the `agent-skills` line if Step 4 is in play.

**Validate:** `POST https://isitagentready.com/api/scan` — expect `checks.discoverability.linkHeaders.status === "pass"`.

## Step 4 — Agent Skills Discovery Index

For sites that publish reusable agent skills. Serve `/.well-known/agent-skills/index.json` listing each SKILL.md file hosted on the domain.

### 4a. File layout

```
public/
  .well-known/
    agent-skills/
      index.json                    (generated)
      <skill-slug>/
        SKILL.md                    (hand-authored)
      <another-skill>/
        SKILL.md
```

### 4b. SKILL.md shape

Each SKILL.md starts with YAML frontmatter:

```markdown
---
name: your-skill-slug
description: One-line description of what this skill does and when to load it. ~200 chars.
---

# Skill Title

Body of the skill...
```

### 4c. Index generator

`scripts/generate-skills-index.mjs`:

```javascript
import { readFileSync, writeFileSync, readdirSync, statSync, existsSync } from 'node:fs'
import { createHash } from 'node:crypto'
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __dirname = dirname(fileURLToPath(import.meta.url))
const root = join(__dirname, '..')
const skillsDir = join(root, 'public/.well-known/agent-skills')
const outPath = join(skillsDir, 'index.json')
const SITE = 'https://your-site.example'
const SKILL_BASE = `${SITE}/.well-known/agent-skills`

if (!existsSync(skillsDir)) process.exit(0)

const skills = []
for (const name of readdirSync(skillsDir).filter((n) => statSync(join(skillsDir, n)).isDirectory())) {
  const skillPath = join(skillsDir, name, 'SKILL.md')
  if (!existsSync(skillPath)) continue
  const body = readFileSync(skillPath)
  const hash = createHash('sha256').update(body).digest('hex')
  const text = body.toString('utf8')
  const fm = text.match(/^---\n([\s\S]*?)\n---/)
  const desc = fm && (fm[1].match(/^description:\s*["']?(.*?)["']?\s*$/m) || [])[1] || name
  skills.push({
    name,
    type: 'skill-md',
    description: desc.trim(),
    url: `${SKILL_BASE}/${name}/SKILL.md`,
    digest: `sha256:${hash}`,
  })
}

const index = {
  $schema: 'https://schemas.agentskills.io/discovery/0.2.0/schema.json',
  skills: skills.sort((a, b) => a.name.localeCompare(b.name)),
}
writeFileSync(outPath, JSON.stringify(index, null, 2) + '\n')
```

Replace `SITE` with the production domain. Wire into `package.json` build chain.

**Validate:** `POST https://isitagentready.com/api/scan` — expect `checks.discovery.agentSkills.status === "pass"`.

## Step 5 — Deploy and audit

1. Commit all changes
2. Push to the Pages-connected branch
3. Wait for Pages deploy to complete
4. `POST https://isitagentready.com/api/scan` with `{"url": "https://your-site.example"}` and verify each check flips to `"pass"`

## Known limits

- Markdown Negotiation at the audit level only requires the homepage to respond correctly. Per-post markdown is bonus value for real agent retrieval, not an audit requirement.
- Pages Function middleware runs on every request. The code above falls through to HTML on any error, so a bad `.md` file never breaks the HTML path.
- Pages Functions on Free plan have a 100,000 request/day limit. For a typical marketing site this is irrelevant; for high-traffic sites, factor in.
- The SHA-256 digests in `index.json` must match the exact bytes of each SKILL.md as served. Regenerate the index whenever a SKILL.md changes, or your digests will drift and some agent clients will refuse to load the skill.

## Change log

- **2026-10-18 — v1.0.** Initial release. Four-item recipe for Content Signals + Markdown Negotiation + Link Response Headers + Agent Skills Discovery Index on Cloudflare Pages Free plan.
