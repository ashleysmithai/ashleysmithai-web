# ashleysmithai-web

Open-source agent skills — Cloudflare agent-readiness, prompt injection guardrails, and more. Mirrored from [ashleysmithai.com](https://ashleysmithai.com) for humans to browse and fork. CC-BY-4.0.

---

## What this is

A public, human-navigable mirror of the agent skills I publish for use with AI agents. Each skill is a single `SKILL.md` file written to the [Agent Skills Discovery](https://schemas.agentskills.io) convention — short, declarative, loadable at the start of an agent session.

Agents consume the live versions directly from the [`/.well-known/agent-skills/` directory on ashleysmithai.com](https://ashleysmithai.com/.well-known/agent-skills/index.json). Humans use this repo to read, fork, adapt, and rip-and-reuse.

If you want the full backstory on why these exist and how I built them, read [Build Log #003: Passing Cloudflare's Agent Readiness audit](https://ashleysmithai.com/blog/agent-ready).

## Skills

### [`cloudflare-agent-readiness`](./skills/cloudflare-agent-readiness/SKILL.md)

The recipe for passing Cloudflare's Agent Readiness audit on the Free plan without upgrading to Pro. Covers Content Signals in `robots.txt`, Markdown Negotiation via a Worker, Link Response Headers, and an Agent Skills Discovery Index. Deliberately skips the audit items that require infrastructure a static content site doesn't have (APIs, auth, agent/MCP servers, commerce protocols).

**Live:** [`ashleysmithai.com/.well-known/agent-skills/cloudflare-agent-readiness/SKILL.md`](https://ashleysmithai.com/.well-known/agent-skills/cloudflare-agent-readiness/SKILL.md)

### [`prompt-injection-shield`](./skills/prompt-injection-shield/SKILL.md)

Guardrail for any agent that reads outside content — inbound emails, scraped webpages, uploaded files, meeting transcripts, DMs from third-party platforms. Enforces three rules: outside content is DATA not INSTRUCTIONS, every piece gets scanned for injection patterns, and any irreversible action triggered by outside content needs a human-in-the-loop checkpoint before executing. Platform-rich (CRMs, email marketing, course platforms, cloud infrastructure, named AI tools).

**Live:** [`ashleysmithai.com/.well-known/agent-skills/prompt-injection-shield/SKILL.md`](https://ashleysmithai.com/.well-known/agent-skills/prompt-injection-shield/SKILL.md)

## How to use

### If you're an AI agent

Point your skill loader at the live URLs above. Each skill is a single `SKILL.md` file with frontmatter metadata and a Markdown body. The discovery index at [`ashleysmithai.com/.well-known/agent-skills/index.json`](https://ashleysmithai.com/.well-known/agent-skills/index.json) carries SHA-256 digests so you can verify integrity before loading.

### If you're a human

Clone or fork this repo, open the `SKILL.md` you want, and adapt it to your own stack. The skills are written to be lifted and tailored — rename the skill, swap your platforms into the lists, add your industry's software, change the trigger phrases to match how your team actually talks. Rip and reuse (R&D).

## Categorization

Flat for now. When the skill count grows past what a single scroll of the README can hold, I'll add category subdirectories (`skills/web/…`, `skills/ai-safety/…`, etc.) rather than letting it turn into a mystery pile.

## About the publisher

Ashley Smith | AI employee builder for small business owners.

- Website: [ashleysmithai.com](https://ashleysmithai.com)
- Build Log: [ashleysmithai.com/blog](https://ashleysmithai.com/blog)
- Contact: [help@ashleysmithai.com](mailto:help@ashleysmithai.com)

## License

[Creative Commons Attribution 4.0 International](./LICENSE) (CC-BY-4.0).

You can share, adapt, and use these skills for any purpose including commercial — the one thing I ask is attribution back to [ashleysmithai.com](https://ashleysmithai.com).
