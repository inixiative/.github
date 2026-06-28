<p align="center">
  <img src="https://raw.githubusercontent.com/inixiative/.github/main/icon-400x400.png" alt="inixiative" width="120" />
</p>

<p align="center"><strong>technology for cooperation</strong></p>

---

We're building the infrastructure that lets strangers collaborate on complex, long-horizon endeavors with the same trust they'd extend a neighbor — identity, governance, and investment, at scale.

Most platforms re-solve the same problems badly, five times over: who can see what, who can do what, how automation and forms and state stay safe as tenants nest inside tenants. **Our open source is a bet that these aren't separate problems — they're one.** A single, serializable, *enforceable model of authority over data*, expressed in one small predicate language, that governs all of them. The libraries are deliberately small; the leverage is in how they compose.

Two things are built here at once, on purpose: the **cooperation platform**, and the **AI-engineering infrastructure we use to build it** — and we dogfood the latter into the former.

---

## The spine — one predicate language, one authority model

The core bet: a serializable predicate language and a composable, enforceable model of authority over data. Everything else is a consumer of it.

**→ [Full vocabulary](https://www.inixiative.com/vocabulary.html)** — every primitive across the ecosystem and the template, with what it solves and how it works. ([source](https://github.com/inixiative/.github/blob/main/VOCABULARY.md))

### json-rules

One serializable `Condition` AST, three backends: evaluate in memory (`check`), compile to a Prisma query (`toPrisma`), or to a PostgreSQL `WHERE` (`toSql`). 38 operators across scalars, dates, windowing, arrays, and multi-source graphs.

On top of the AST sits the **lens**: a composable, enforceable model of *what a party can see and what it can do*, delegated down trust boundaries (org → space → subtenant → client) — narrow-only, with `where` clauses as grants applied server-side, enforced at author-time *and* execution-time. It's the authority/visibility spine the rest of the stack composes on, not a filter helper. Author a predicate once; run it wherever it has to run, against whatever source.

[GitHub](https://github.com/inixiative/json-rules) · [npm](https://www.npmjs.com/package/@inixiative/json-rules)

### prisma-map

An ORM-agnostic, runtime-friendly model map extracted from a Prisma client — the schema reflection the lens reasons over, so authority can be expressed over your real data graph without coupling to Prisma internals. Works with Prisma v6 and v7.

[GitHub](https://github.com/inixiative/prisma-map) · [npm](https://www.npmjs.com/package/@inixiative/prisma-map)

### permissions

Relationship-based access control as json-rules predicates: relationship-walking, per-row grants, and an app-injected resolver so the engine stays generic. Permissions stop being bespoke `if`-checks scattered through controllers and become *rules authored over your authorized view* — the same primitive, pointed at "can this actor do X."

[GitHub](https://github.com/inixiative/permissions) · [npm](https://www.npmjs.com/package/@inixiative/permissions)

### transitions

State machines whose guards and affordances are json-rules predicates. The same authority model, applied to a different question — "which move is legal now, for this actor, given this state" — so your workflow rules and your permission rules speak one language.

[GitHub](https://github.com/inixiative/transitions)

### rules-builder

The visual editor for everything above: compose conditions against a lens surface, drag-and-drop, no JSON by hand. Headless core plus a bring-your-own-components contract, so it drops into any design system. The authoring front-end for permissions, email automation, feature flags, and forms alike.

[GitHub](https://github.com/inixiative/rules-builder)

### conditional-form

Dynamic forms whose show/hide and validation logic *is* the same predicate language. Forms that reason exactly like the backend does — define the rule once and it governs both the UI and the submission.

[GitHub](https://github.com/inixiative/conditional-form) · [npm](https://www.npmjs.com/package/@inixiative/conditional-form)

---

## The platform

What the spine is for.

### template

The SaaS foundation the platform stands on: Bun, Hono, Prisma 7, React/TanStack, BullMQ, Redis — with auth, multi-tenancy, permissions, jobs, an app-events bus, audit logging, field-level encryption, and the whole rules/lens spine wired in. Fork it and the hard, security-critical parts are already done and tested. *(in progress)*

[GitHub](https://github.com/inixiative/template)

### whitepaper

Adaptive governance infrastructure for cooperative societies — the thesis the platform implements: how groups of strangers set rules, hold one another accountable, and invest together without a central referee. *(in progress)*

[GitHub](https://github.com/inixiative/whitepaper)

### inixiative

The product: identity, permissions, escrow, and governance — the shared infrastructure layer for cooperation at scale, built on the template and the spine. *(in progress)*

[GitHub](https://github.com/inixiative/inixiative)

---

## The AI engineering layer — build the tools, then build with them

The infrastructure we use to ship the platform faster and more correctly — dogfooded daily.

### foundry

Agent orchestration: wraps Claude Code sessions with persistent memory, project conventions, multi-thread awareness, and correctness checking — a system designed to get sharper after every interaction instead of starting cold each time. *(in progress)*

[GitHub](https://github.com/inixiative/foundry) · [Site](https://foundry-marketing.vercel.app)

### foundry-oracle

Corpus optimization: mines your merged PRs into evaluation fixtures, scores your codebase against them, and opens PRs with measurable improvements — turning your own git history into a feedback loop. *(closed beta)*

[GitHub](https://github.com/inixiative/foundry-oracle) · [Waitlist](https://foundry-marketing.vercel.app/oracle)

### atlas

A concept map of your codebase: lightweight `@atlas` annotations plus a queryable graph (`atlas graph` / `atlas query`), so humans and agents navigate by *concept* — feature, primitive, infrastructure — instead of crawling folders.

[GitHub](https://github.com/inixiative/atlas) · [npm](https://www.npmjs.com/package/@inixiative/atlas)

### bench

An agentic-engineering benchmark with two tracks — build-from-scratch and work-within-an-existing-codebase — scored not just on passing tests but on *cruft vs elegance*: does the agent leave the codebase better, or just barely working.

[GitHub](https://github.com/inixiative/bench)

### hivemind

Multi-agent coordination: a message bus with role-scoped channels, so fleets of agents collaborate on a workflow without stepping on each other.

[GitHub](https://github.com/inixiative/hivemind) · [npm](https://www.npmjs.com/package/@inixiative/hivemind)

---

## Stack

`Bun` · `Hono` · `PostgreSQL` · `Prisma 7` · `React` · `TanStack` · `Tailwind v4` · `Redis` · `BullMQ`

---

## Get In Touch

We're open to advisory engagements and collaboration.

- **Substack:** [@inixiative](https://inixiative.substack.com)
- **Twitter/X:** [@inixiative](https://x.com/inixiative)
