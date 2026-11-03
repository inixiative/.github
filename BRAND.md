# inixiative copy brief

Aron's decisions, 2026-11-02. Every marketing surface uses these words: inixiative.com, the GitHub org profile, repo descriptions, READMEs and the Kingdom landing. When in doubt, use these exact lines. Don't invent new slogans. Each product fact matches what is built; anything not built yet is marked in progress.

Why it lives here: our copy rotted once already. We left old names in repo descriptions and READMEs after the products changed. This file is the one source. The marketing owner's weekly pass compares every public surface against it, and `@inixiative/config check` fails on the retired words below. Change the copy here first, by PR.

## Voice

- Write in plain language about what the user sees and gets, in short, direct sentences.
- No em-dash asides, no "not X, but Y" constructions, no hype adjectives, and never "novel" or "revolutionary".
- State the mechanism and the bet plainly, without hedging. Marketing is not the paper.
- Don't invent numbers. Show why with specifics instead of superlatives.
- Aron's chosen pitches stay exactly as written: "Technology for cooperation." · "An agent that works like you do." · "Giving your work a voice." · "Stop repeating yourself to Claude." · "Stop checking a dozen apps." · "Stop being a meat proxy." · "Everything you're doing, in one place." · "Bring your agent." · "The voice of the codebase." · "Stop rediscovering what you need to build." · "like shadcn, but for the whole SaaS app" · the `<iXi>` mark. The sentences around them follow the rules below.

The rules below are the short form of the no-slop skill; the skill holds the register anchors and the full detail (Aron, 2026-11-02: "Please use the no-slop skill on all of our copy."). If you have the skill (`~/.claude/skills/no-slop-writing`), load it and its two reference files before drafting. Why: Aron wants every public line to read as written by the people who built the product, with a fact in every sentence.

### Genre

Write product pages, READMEs, profile copy and repo descriptions as a briefing from the person who built the thing to a peer who will use it. Each section says what the thing does, who it is for, what it does not do yet, and how to start. Headings state the finding. Telegraphic lists are fine where every item carries a fact. Every sentence carries a fact (a name, a number, a decision, a state such as "in progress") or connects two that do; if there is nothing concrete to say, cut the sentence.

### Agency

People make decisions, so name them. A thing can be the subject only when the verb is what it mechanically does ("Kingdom runs the command", "Archive keeps every session"). An object never takes a verb that stands for a human choice or its consequence: surfaces, emerges, creeps in, falls through the cracks, keeps adding, ends up as, drives (a decision), nobody owns, nobody chose. Past decisions go in past tense with a named actor; current conditions go in present tense; don't slide between them within a sentence. Write that a team adopted, selected or brought in a tool, not that it bought one. Test: ask "who decided this?" of every sentence about a choice, and name that person or team.

### Never

- Em dashes, in any function, and en dashes used as dashes. Use a period, comma or colon.
- "Not X, it's Y" or "not X, but Y" reframes, unless X is something the reader believed and Y corrects it.
- Rhetorical questions as transitions ("So what does this mean?").
- One-line emphasis fragments ("That matters.", "This is the shift.").
- Colons that build suspense. A colon delivers information.
- Stacked synonym adjectives. Pick the one accurate word.
- Affirmation openers ("Absolutely", "Great question"), "it's worth noting", "it's important to understand".
- Truth-intensifiers: actually, genuinely, really, truly, in reality, the reality is. Never in a heading or subject line.
- Tacked-on codas: ", which is why…", ", which is where…", ", which is what…".
- Rule-of-three padding. Two real points stay two.
- Insight-cosplay metaphors: load-bearing, says the quiet part out loud, where it lands, the move, the through-line, the tell, at its core, table stakes, the sharpest version of, the honest answer.
- Filler openers: Here's the thing, That said, At the end of the day, In today's landscape, Make no mistake, Simply put.
- Claims nobody has measured, and superlatives.
- Corporate-polish words: leverage, robust, seamless, navigate (figurative), underscore, foster, empower, unlock, elevate, streamline, delve, dive into, ecosystem, landscape, realm, tapestry, nuanced, crucial, pivotal, vital, compelling, myriad, plethora, game-changer, cutting-edge, best-in-class, holistic, synergistic, pioneering; also intuitive, data-driven as decoration, "deliver personalized experiences", "amplifying the spectrum", and stacked verbs (craft, integrate, maximize, optimize). Names that contain one (a package or a command such as `train/ecosystem-sync`) stay.

### Review as a rejector

Don't smooth prose line by line: a reviewer who rewrites as they go becomes a second author writing in the default register. For each sentence, keep it untouched or reject it, name the rule above that it breaks, and write that one sentence again inside the genre. Before publishing, search the diff for `—`, ` – ` and every word in the polish list, ask "who decided this?" of each sentence about a choice, and delete any opening line that clears its throat. Each sentence should fit inside one of the skill's register anchors.

Vary paragraph length. Put the most important item in a list where it reads as most important. Match the weight of the language to the weight of the fact. When you don't know, say so instead of filling the gap.

## inixiative

- Line: **Technology for cooperation.**
- Three sections, in this order: **Building blocks** · **Agentic** · **Governance** (Aron, 2026-11-02): the building blocks are the base, the agentic tools help people and their agents use them, and governance (tools for cooperation) is the long-term goal.
- **The story, in this order (Aron, 2026-11-02).** Every surface that introduces inixiative tells it, in two sentences or in a page. Aron speaks in the first person singular where the copy is his.
  1. **The building blocks are a decade of lessons, packaged.** Aron: "It's a decade of lessons learned and making my work portable, and not having to figure things out every time." Template and the libraries are those answers, so you start from them instead of rediscovering them. Six of these packages (json-rules, transitions, prisma-map, rules-builder, atlas, gloss) run in production at the company where Aron works. Say that; don't name the company.
  2. **Coding agents ignore the answers, so Foundry exists.** Agents build parallel versions of what the codebase already has (on 2026-10-18 an agent built a second auth path in Kingdom beside the existing one). Aron: "If you were able to use the template, fine. I wouldn't need to have built Foundry." Foundry's Wardens point the agent at what exists. Kingdom, Archive, Signet, KCP and Oracle come from the same problem; keep them to a line each.
  3. **Governance is where it goes:** tools for people, and their agents, to cooperate. "Technology for cooperation."
  Why: on 2026-11-02 Aron found the public copy read like an internal plan (three doors, a layer diagram) and didn't say why any of it exists.
- **No internal jargon in public copy (Aron, 2026-11-02).** Never "door(s)", "domain", "palette", "data-domain", "Layer 1 / Layer 2", and never "regions" or "chorus" without a plain explanation (say "Wardens are built from your docs, one per topic"). The theme notes below (`data-domain`, colours) are for builders, not readers. Use the plain section names: Building blocks, Agentic, Governance.
- People: **Contributors: Aron Greenspan.** Aron Greenspan only, for now (Aron, 2026-11-02: "No one else has really contributed."). Don't use "founder" or "co-founder" labels. Don't write "available for work", a hiring pitch or employer names.
- Mark: keep the current `<iXi>` exactly. The only allowed changes are outlining it to paths and a tighter favicon.

## Building blocks

Theme: `data-domain="building-blocks"`, amber.

- Line: **Stop rediscovering what you need to build.**
- Template is "like shadcn, but for the whole SaaS app": you start from it and own the code. It is the whole app (API, web, admin and superadmin apps, the UI package, jobs, auth, permissions), never "a backend" (Aron, 2026-11-02: "This thing is more than a backend."). If a coding agent writes its own versions instead, Foundry points it back at the blocks.
- Show why each block is better, with specifics:
  - **json-rules:** one rule AST compiles to in-memory checks, Prisma queries and SQL WHERE. Rules are serializable, with type-safe operators, paths and binding.
  - **permissions:** RBAC + ABAC + ReBAC with entitlements and delegates, decided on your own records with no separate store to sync. The core is about 1k lines.
  - **transitions:** declarative lifecycles on json-rules.
  - **rules-builder:** a headless builder that offers only what the engine can run.
  - **prisma-map**.
- Two groups, with template on top consuming them (Aron, 2026-11-02):
  - **The libraries**, in dependency order: json-rules → permissions → transitions → prisma-map → rules-builder.
  - **Developer tooling:** atlas (a map of your code), gloss (explanations kept out of the source), config (the shared toolchain and theme).
  - **atlas:** a codebase map from `@atlas` annotations; `atlas check` fails CI when they drift.
  - **gloss:** comment-free source with sidecar explanations.
  - **config:** the shared toolchain and theme.

## Agentic

Theme: `data-domain="agentic"`, lime.

- Lines: **"An agent that works like you do."** and **"Giving your work a voice."**
- Framing (Aron's Substack, 2026-03-21, "The Real Problem With AI Coding Isn't Capability. It's Alignment."):
  - Every session starts half-amnesiac, and you pay a re-alignment tax: re-explaining conventions, decisions and taste.
  - The model and the harness are outside your control. Your corpus (docs, rules, CLAUDE.md) is the only layer you control, and Foundry gives you a tool to control it.
  - The agent does what you want when it's told the right thing. The bet is that your corrections compile into docs that tell it automatically.
  - In early runs, we gave the agent Aron's corrections before it planned, and the plans came out right. Cite them as "early runs" with no invented stats. Oracle runs the tuning loop.
- **Foundry** (source-available, BSL 1.1) is the workspace around your agent: Claude Code, Codex and Muse, plus ACP apps. Its pitches, in order:
  1. **Stop repeating yourself to Claude.** What you've already decided reaches every session. Wardens notice what your request touches and point at what your codebase already has and where it's written down. They never block; your agent decides. (Built; off by default.)
  2. **Every agent you pay for, in one place.** Claude Code, Codex and Muse on your own subscriptions; a worker per thread; every session captured.
  3. **Stop being a meat proxy.** Your agent, their agent and the people in one thread, hashing out the work. (Host side built; joiner view in progress.)
  4. **Stop checking a dozen apps.** Whatever comes in (a Slack DM, a review request, an email) reaches you already researched and in context. It's read as data, so it can't instruct your agent. (In progress: Kingdom streams the events; Foundry doesn't receive or sort them yet.)
  5. **Set up your repo.** Foundry reads your repo and builds the docs and Wardens. (In progress.)

  Subhead option: "The voice of the codebase."
- **Kingdom** (closed, hosted at kingdom.inixiative.com): "Everything you're doing, in one place." Your apps, data, AI and agents across every team you're in. Decide who and what can reach each piece, for how long, and take it back. "Bring your agent." The Kingdom landing's eyebrow is always "Kingdom".
- **Signet** (open, MIT): pass-through execution with tracking and audit, built on the permissions library. It is how anything uses an integration through Kingdom: a revocable pass on the owner's terms.
- **KCP, the Kingdom CLI Proxy** (open, MIT; **in progress**): built so far: `kcp login`, `status` and `map`. Next: Kingdom runs your provider CLIs with its stored credentials and streams the output back, so none sit on your machine. Don't claim it works end to end yet.
- **Archive** (open, MIT): keeps every agent session, local-first, and syncs to the archives you choose.
- **Oracle** (closed): measures whether a correction stuck and tunes the docs and Wardens (the loop). Give it one line, and never describe its method, fixtures or scoring.
- **agent-session** (open, MIT): drive Claude Code, Codex and Muse as persistent streaming sessions behind one interface.
- **bench** ("Do I Still Have a Job?"): our own challenge set for coding agents, staff to architect tiers. It is its own thing, not proof for Foundry. Its results are from June: show the date, and say more challenges are coming.

Prior work: Foundry is not the first system to turn corrections into lasting guidance. Trace (arXiv 2606.13174) is the nearest prior work. State what is different; never claim "nobody else".

## Governance

Theme: `data-domain="governance"`, violet.

- The whitepaper and proposals. There is no tagline yet; use a plain description: "Designs for cooperation at scale."
- Long-form pages read in the light theme.

## Open source

- **MIT by default:** the building blocks, Archive, agent-session, Signet, KCP, and Foundry's core engine and ACP adapter.
- **Source-available (BSL 1.1):** Foundry's app. It is free to use, including at work, but not to resell, and it becomes MIT in April 2030.
- **Closed:** Kingdom and Oracle.
- **Why, in one plain sentence (draft, pending Aron's wording):** we open the pieces people build on, and keep the hosted service and the tuning closed so the project can sustain itself.

## Retired words (never publish)

- Kastle / Castle (as a product)
- Connection(s) as a Kingdom model
- Cartographer, Librarian, Artificer and Steward as Foundry roles
- "foundry-oracle"
- medical or healthcare demos (a doctor's visit as a storytelling example is fine)
- Cursor as a Foundry runtime
- "your own LLM keys"
- token savings as Foundry's value
- "principal" for the permission holder: use actor and anchor (the permissions README dropped it in #25).

Where a word can be checked mechanically, `@inixiative/config check` fails on it (its retired-terms rule, with per-repo allow lists and baselines). The rest are checked by the weekly pass.
