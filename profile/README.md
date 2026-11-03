<p align="center">
  <img src="https://raw.githubusercontent.com/inixiative/.github/main/icon-400x400.png" alt="inixiative" width="120" />
</p>

<p align="center"><strong>Technology for cooperation.</strong></p>

<p align="center">
  <a href="#building-blocks">Building blocks</a> · <a href="#developer-experience">Developer experience</a> · <a href="#agentic">Agentic</a> · <a href="#governance">Governance</a>
</p>

I'm Aron Greenspan. These repos are a decade of lessons from building software, packaged so I don't have to figure the same things out on every project. The building blocks are those answers. Coding agents ignored them and wrote parallel versions, so I built Foundry to point them at what the codebase already has. The long-term goal is tools for people, and their agents, to cooperate.

---

## Building blocks

**Stop rediscovering what you need to build.**

Across my career I kept running into the same shapes of problem. More often than not, people built a one-off solution for each, often right next to another problem with the same shape. I pulled those problems out and built the best-shaped solution I could for each, so it can be reused wherever that problem shows up. Template is where they come together, and the packages below handle the most common problems on their own. json-rules, transitions, prisma-map and rules-builder run in production at the company where I work.

### template

Like shadcn, but for the whole SaaS app. You start from it and own the code: an API, web, admin and superadmin apps and a shared UI package on Bun, Hono, Prisma 7 and React. It already solves sign-in with email, OAuth and API tokens; organizations and spaces with roles; permissions; field-level encryption with key rotation; an audit log; background jobs; signed webhooks with retries; live data over WebSockets; email templates and delivery; and a typed API client generated from the routes. If your coding agent writes its own versions instead, Foundry points it back at these.

[GitHub](https://github.com/inixiative/template)

### The packages

The engine, the library and the builders for the most common problems, in dependency order from the bottom up.

- **[prisma-map](https://github.com/inixiative/prisma-map)**: a runtime model map (fields, relations, foreign-key direction, enums) from a generated Prisma client.
- **[json-rules](https://github.com/inixiative/json-rules)**: one rule AST compiles to in-memory checks, Prisma queries and SQL `WHERE` clauses. Serializable, with type-safe operators, paths and binding.
- **[permissions](https://github.com/inixiative/permissions)**: RBAC, ABAC and ReBAC with entitlements and delegates, decided on your own records with no separate store to sync. A small core you can read; its runtime dependencies are json-rules and lodash-es.
- **[transitions](https://github.com/inixiative/transitions)**: declarative lifecycles on json-rules. Which move is legal now, for this actor, on this record.
- **[rules-builder](https://github.com/inixiative/rules-builder)**: a headless rule builder that offers only what the engine can run. [Live demo](https://www.inixiative.com/rules-builder/).

Paused, not recommended for use: [hivemind](https://github.com/inixiative/hivemind), [conditional-form](https://github.com/inixiative/conditional-form).

---

## Developer experience

Tools for working on the code. None of them ships in your product.

- **[atlas](https://github.com/inixiative/atlas)**: a codebase map built from `@atlas` annotations; `atlas check` fails CI when they drift.
- **[gloss](https://github.com/inixiative/gloss)**: comment-free source, with explanations in a sidecar tree.
- **[config](https://github.com/inixiative/config)**: the shared toolchain and theme for every repo here, with a CLI that checks, syncs and releases them together.

Comes with template:

- **Worktrees:** `bun run worktree:create` gives each worktree its own Postgres databases, Redis database, storage buckets and ports, so several branches or agents run side by side.
- **Init:** a guided setup in the terminal that provisions a new project's services (GitHub, PlanetScale, Railway, Vercel, Cloudflare, Resend) and keeps their secrets in Infisical.
- **CI rules:** checks the repo owns, run by `bun run check`, that fail when code skips a system the repo already has or a doc names something that no longer exists.

**[Full vocabulary](https://www.inixiative.com/vocabulary.html)** of every primitive ([source](https://github.com/inixiative/.github/blob/main/VOCABULARY.md)).

---

## Agentic

**An agent that works like you do.** Giving your work a voice.

Coding agents kept writing their own versions of what my repos already had, such as a second auth path beside the existing one. Every agent session starts half-amnesiac. You pay a re-alignment tax: re-explaining your conventions, your decisions, your taste. The model and the harness are outside your control. Your corpus (docs, rules, `CLAUDE.md`) is the one layer you do control, and Foundry gives you a tool to control it. The agent does what you want when it's told the right thing. The bet is that your corrections compile into docs that tell it automatically. In early runs, we gave the agent Aron's corrections before it planned, and the plans came out right. Oracle runs the tuning loop.

### Foundry

The workspace around your agent: Claude Code and Codex, plus ACP apps. The voice of the codebase.

- **Stop repeating yourself to Claude.** What you've already decided reaches every session. Wardens notice what your request touches and point at what your codebase already has and where it's written down. They never block; your agent decides. *(Built; off by default. Turn the chorus on in settings.)*
- **Every agent you pay for, in one place.** Claude Code and Codex on your own subscriptions, a worker per thread, every session captured.
- **Stop being a meat proxy.** Your agent, their agent and the people in one thread, hashing out the work. *(Host side built; joiner view in progress.)*
- **Stop checking a dozen apps.** Whatever comes in (a Slack DM, a review request, an email) reaches you already researched and in context. It's read as data, so it can't instruct your agent. *(In progress: Kingdom streams the events; Foundry doesn't receive or sort them yet.)*
- **Set up your repo.** Foundry reads your repo and builds the docs and Wardens. *(In progress.)*

[GitHub](https://github.com/inixiative/foundry) · [npm](https://www.npmjs.com/package/@inixiative/foundry)

### Kingdom

Everything you're doing, in one place. Your apps, data, AI and agents across every team you're in. Decide who and what can reach each piece, for how long, and take it back. Bring your agent.

[kingdom.inixiative.com](https://kingdom.inixiative.com)

### Signet

Pass-through execution with tracking and audit, built on the permissions library. The way anything uses an integration through Kingdom: a revocable pass on the owner's terms.

[GitHub](https://github.com/inixiative/signet) · [npm](https://www.npmjs.com/package/@inixiative/signet)

### KCP, the Kingdom CLI Proxy

Built so far: `kcp login`, `status` and `map`. Next: Kingdom runs your provider CLIs with its stored credentials and streams the output back, so none sit on your machine. *(In progress.)*

[GitHub](https://github.com/inixiative/kcp) · [npm](https://www.npmjs.com/package/@inixiative/kcp)

### Archive

Keeps every agent session. Local-first, and syncs to the archives you choose.

[GitHub](https://github.com/inixiative/archive) · [npm](https://www.npmjs.com/package/@inixiative/archive)

### Oracle

Measures whether a correction stuck, and tunes the docs and Wardens.

### agent-session

Drive Claude Code and Codex as persistent streaming sessions behind one interface.

[GitHub](https://github.com/inixiative/agent-session) · [npm](https://www.npmjs.com/package/@inixiative/agent-session)

### bench: Do I Still Have a Job?

Our own challenge set for coding agents, from staff to architect tiers. Results are from June 2026; more challenges are coming.

[bench.inixiative.com](https://bench.inixiative.com)

---

## Governance

The long-term goal is tools for people, and their agents, to cooperate. The whitepaper and the proposals are designs for cooperation at scale: how groups set rules, hold each other accountable, and invest together without a central referee.

**Planned: Inixiative.** A governance platform that runs over time, with ownership tracked in tokens. A group funds an initiative in phases: members put money into escrow, the operator shows a milestone was met, the members vote, and only then is that phase's money released. An ongoing venture votes each period on whether to reinvest or pay out. Each initiative sets its own rules; the marketplace it runs in can't change them, move its funds or override its votes. It started as governance software; a friend pointed me to managing money as the practical place to start, because it forces real cooperation. Nothing is built for users yet. [The write-up](https://www.inixiative.com/proposals/platform/).

**Planned: Livestock.** The first product meant to run on Inixiative: cattle trusts in Uruguay and Argentina, with ownership in tokens. Livestock handles the cattle side; Inixiative handles the funding, the milestones and the votes.

[The proposals](https://www.inixiative.com/proposals) · [whitepaper on GitHub](https://github.com/inixiative/whitepaper)

---

## Open source

MIT by default: the building blocks, Archive, agent-session, Signet, KCP, and Foundry's core engine and ACP adapter. Foundry's app is source-available under BSL 1.1: free to use, including at work, but not to resell; it becomes MIT in April 2030. Kingdom and Oracle are closed.

We open the pieces people build on, and keep the hosted service and the tuning closed so the project can sustain itself.

---

**Contributors:** Aron Greenspan.

[www.inixiative.com](https://www.inixiative.com) · [Substack](https://inixiative.substack.com) · [X](https://x.com/inixiative)
