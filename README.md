<p align="center">
  <img src="https://raw.githubusercontent.com/inixiative/.github/main/icon-400x400.png" alt="inixiative" width="120" />
</p>

<p align="center"><strong>technology for cooperation</strong></p>

---

We're building a cooperation platform — identity, governance, and investment infrastructure that lets strangers collaborate on complex, long-horizon endeavors with the same trust they'd extend to a neighbor.

### Open Source

- **[template](https://github.com/inixiative/template)** *(in progress)* — Reusable SaaS monorepo foundation. Bun, Hono, Prisma 7, React/TanStack, BullMQ, Redis — auth, permissions, jobs, and multi-tenant patterns out of the box.
- **[json-rules](https://github.com/inixiative/json-rules)** — Type-safe rules engine with 38 operators. Declarative business logic for permissions, pricing, eligibility, and form visibility.
- **[prisma-map](https://github.com/inixiative/prisma-map)** — Model map extracted from a Prisma client: fields, relations, FK info. The schema reflection the rules engine reasons over; supports Prisma v6 and v7.
- **[permissions](https://github.com/inixiative/permissions)** — Relationship-based access control as json-rules predicates. Relationship-walking checks, per-row grants, and an app-injected resolver so the engine stays generic.
- **[transitions](https://github.com/inixiative/transitions)** — State machines whose guards and affordances are json-rules predicates, so workflow rules and permission rules speak one language.
- **[rules-builder](https://github.com/inixiative/rules-builder)** — Visual rule builder for json-rules. Headless core with a bring-your-own-components contract; drag-and-drop condition authoring, no JSON by hand.
- **[atlas](https://github.com/inixiative/atlas)** — A concept map of your codebase: `@atlas` annotations plus a queryable graph, so humans and agents navigate by concept instead of crawling folders.
- **[hivemind](https://github.com/inixiative/hivemind)** — Multi-agent coordination library. Message bus with role-scoped channels for AI-powered workflows.
- **[foundry](https://github.com/inixiative/foundry)** *(in progress)* — Context-layered agent orchestration harness. Primitives for sessions, middleware, tracing, and multi-provider LLM integration on Bun.
- **[whitepaper](https://github.com/inixiative/whitepaper)** *(in progress)* — Adaptive governance infrastructure for cooperative societies.
- **[config](https://github.com/inixiative/config)** — Shared toolchain for the ecosystem. Blessed TypeScript/Biome/Bun versions, tsconfig/biome/tsup presets, and a CLI that keeps every repo in sync and cascades releases downstream.

### Coming Soon

- **foundry-oracle** — Corpus optimization engine. Automated recursion loop that captures agent interactions, generates test fixtures, and improves AI context through multi-agent evaluation.
- **inixiative platform** — Identity, permissions, escrow, and governance. The shared infrastructure layer for cooperation at scale.

### Stack

`Bun` · `Hono` · `PostgreSQL` · `Prisma 7` · `React` · `TanStack` · `Tailwind v4` · `Redis` · `BullMQ`

### Get In Touch

We're open to advisory engagements and collaboration.

- **Substack:** [@inixiative](https://inixiative.substack.com)
- **Twitter/X:** [@inixiative](https://x.com/inixiative)
