# Vocabulary

We build the inixiative packages to **"solve it once"**: each concern has one home, and new features use that home instead of working around it. This page is the glossary of every primitive, in four parts. The **building blocks** come first, in the order they stack: prisma-map supplies the field map, json-rules writes rules and lenses against it, permissions decides access with those rules, transitions guards state changes with json-rules and permissions, and rules-builder edits rules in your UI. **Developer experience** (atlas, gloss, config) helps the people and agents working in the code without changing what the code does. The **template** is a production SaaS app built on the building blocks. The **agentic** part covers Foundry, Kingdom and the pieces around them. Each entry names a **concept** (what it solves, how it works); the methods on that concept sit beneath it in a dim API row.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="brand/vocabulary-map.svg">
  <img src="brand/vocabulary-map-light.svg" width="100%" alt="How the libraries fit. prisma-map is the base: json-rules and permissions read its map. permissions is built on json-rules; transitions on json-rules and permissions; rules-builder on json-rules. template sits on top and uses prisma-map, json-rules, permissions, rules-builder, atlas and config, and moves onto transitions and gloss next. Developer experience beside them: atlas (uses json-rules), gloss, config (the toolchain under every package), and the worktrees, init and CI rules that come with template.">
</picture>

> Canonical source: [`inixiative/.github`](https://github.com/inixiative/.github/blob/main/VOCABULARY.md) · rendered at [www.inixiative.com/vocabulary](https://www.inixiative.com/vocabulary/)

---

## Part I · The building blocks

Five MIT packages, in the order they build on each other: §1 is `@inixiative/prisma-map`, §2 to §6 are `@inixiative/json-rules`, §7 is `@inixiative/permissions`, §8 is `@inixiative/transitions` and §9 is `@inixiative/rules-builder`.

### 1 · Schema: the shape of data

#### @inixiative/prisma-map
**Solves:** one runtime map of your Prisma schema that every other building block reads, without adding a Prisma generator.
**How:** reads the client you already generate (Prisma v6 or v7) and returns its models, fields, field kinds, relations with their foreign-key direction (`fromFields`/`toFields`), enum values and defaults, keyed by Prisma name, with the database name (`dbName`) wherever `@map`/`@@map` changes it. json-rules reads it as a `FieldMap` (`{ models: map }`), and permissions takes that field map as its `SchemaMap`'s `fieldMap`.
<small>API · `buildPrismaMapV7` / `buildPrismaMapV6`: build the map from the generated client · `getColumns(map, model)`: a model's scalar and enum field names, in schema order · `pickColumns(map, model, row)`: a row cut to its columns, dropping included relations · `getRelations` / `relationForeignKey`: a model's relations and their keys · `tableName` / `columnName` / `storedValue`: the table, column and stored enum value the database uses, for code that writes SQL · `hasColumn` / `isScalarField` / `isEnumField`: narrow a field to the kinds that have a column.</small>

#### FieldMap
**Solves:** describe data (models, fields, kinds, relations, allowed values) in one shape the rest of json-rules is written against, with no ORM import.
**How:** `{ models: { Model: { fields: { name: FieldMapEntry } } }, enums? }`. A `FieldMapEntry` has `kind` (`scalar | object | enum | bridge`), `type`, optional `isList`, `fromFields`/`toFields` (FK columns), and `values`: an optional `readonly string[]` that solves "constrained value set" once (enums and picklists route through it; `validateRuleInLens` gates against it regardless of kind, so no enum-promotion hack). Projection output adds `options` (`{ value, label?, groups? }`) on every value-gated or sourced field. A prisma-map map is one, under `models`.

#### Bridge
**Solves:** navigable relations *across* separate field maps (e.g. Prisma + Salesforce + CRM) without merging them.
**How:** a `Bridge` declares two endpoints (`{ fieldMap, model, on }`) and a cardinality (`oneToOne | oneToMany`). Rules read across it by the bridge key (`salesforce:Contact.industry`); `check` walks plain properties, so the caller embeds the foreign row under that key.
<small>API · `stitchFieldMaps`: inject a navigable `kind: 'bridge'` field on each endpoint model · `indexBridges(set, rawData)`: index foreign rows by their `on` key into a `BridgeDictionary`, ready to embed under bridge keys.</small>

### 2 · Rule logic: the atom

#### Condition
**Solves:** one serializable way to express "does this hold?": the single predicate language used everywhere a yes/no over data is needed.
**How:** a recursive JSON tree of `all`/`any` (boolean), field operators (`equals`, `in`, `contains`, `between`…), array operators (`any`/`all`/`none`/`atLeast`…), date operators, aggregates, and `if`/`then`/`else`. No code, no `eval`: pure data you can store, ship, and diff.
<small>API · `check(condition, data)`: evaluate against a record → `true` or a failure string; handles what the compilers can't (array predicates, aggregates, windowing). It is the "fetch results, then filter" backstop.</small>

#### Operators
**Solves:** a closed, introspectable catalog of what comparisons exist and what input each needs, so a UI renders the right control and every target honors the same set.
**How:** `Operator` · `ArrayOperator` · `DateOperator` const catalogs; each operator carries the targets it compiles to and a `ValueShape` (`none | scalar | ordered | array | string | pattern | range | dateValue | dateRange | dateWindow | dayList | count | predicate`).
<small>API · `getValueShape(op, family)`: the input shape a control should render · `getOperatorsForKind`: the operators a field kind takes, narrowed to one target when given.</small>

#### RuleTarget
**Solves:** "where will this rule run?", so a rule can be restricted to operators all chosen targets support.
**How:** a tag (`check` in-memory · `toPrisma` · `toSql`); operator availability is intersected across the selected targets.

#### Bindings
**Solves:** a stored rule that names a per-request value (the caller's id, the tenant) without being rewritten for each request.
**How:** a `{ bind }` token stands where a value goes; `bindOptional` marks one that may stay unfilled. `check(rule, row, { bindings })` fills them while checking, and `bindRule` fills them before compiling. A required bind left unfilled throws `UsageError`.
<small>API · `bindRule(rule, bindings)`: substitute the binds the map covers and leave the rest in place · `listBindings(rule, { required })`: the names a bindings map must cover · `clockUnit(rule)`: the finest clock unit the rule's answer can change by while its data stays the same.</small>

#### Rule validation
**Solves:** reject a malformed rule, or one that would pass on missing data, before it is stored.
**How:** `validateRule` reports every issue in a rule's shape. With `{ positive: true }` it also refuses every shape that can hold on a missing value (`if`/`then`/`else`, array and aggregate rules, the negations, `isEmpty`, `notExists`), so a rule checked that way passes only on what the row carries.
<small>API · `validateRule(rule, options)` → `{ ok, errors }` · `assertValidRule`: the same check, thrown.</small>

#### Engine globals
**Solves:** process-wide defaults a caller sets once at boot instead of passing on every call.
**How:** typed dotted paths into one state object: `string.caseInsensitive`, `string.fuzzy`, `prismaOptions.datasource.provider`, `sql.dialect` (`SqlDialect.postgres` by default, or `SqlDialect.mysql`) and `clock.finestUnit`. A rule's own flag wins over its default.
<small>API · `engineGlobals.set` / `get` / `reset` · `engineGlobals.with(values, fn)`: run one synchronous call under other defaults.</small>

### 3 · Lens: one schema, many scoped views

#### Lens
**Solves:** one scoped, enforceable view of a schema, authored once and narrowed monotonically down trust boundaries (org → space → subtenant → client), never widened; enforced at author-time *and* execution-time. The authority/visibility spine the rest of the stack composes on.
**How:** three forms of one thing:

| Form | What | Used for |
|---|---|---|
| **Reference** | `Lens` + a serializable `LensNarrowing` chain, stored one `StoredLens` record per layer | author, store, ship |
| **Visit** | `lensVisit`: one visit resolved on demand (shown fields with values and options, sources, labels) | the computed view, narrowing applied |
| **Public** | `projectLens` (per path, or `{ by: 'model' }` for a flattened lens) | the leak-safe surface a UI/consumer sees |

A `LensNarrowing` layer has a `root` (path-specific, from the anchor) and `mapDefaults` (wherever a model appears). Each node carries `picks`/`omits` (columns), `enumPicks`/`enumOmits` (value sets), `where` (a row-filter `Condition`), `sources` (data-backed options), and `relations` (recursive). Relations are off until the first narrowing turns them on; later layers only narrow. Composes filter-first, AND-only.
<small>API · `createLens`: anchor at a model in a map (stitches bridges) · `validateNarrowing`: reject a chain that widens or names missing fields · `projectLens(lens,{sourceValues})`: project the public surface, attaching fetched options to each field as `options` · `lensVisit` / `walkLensPath` / `readLensValue`: one visit, one path, one value off a row, as the lens shows it · `clampLens`: AND clamps into the first layer · `bindLens` / `listLensBindings`: fill a narrowing's `bind` tokens per request · `storeLens` / `composeLens`: store a chain one record per layer and compose it back · `toLensSelect` / `projectRows`: fetch the rows a lens shows and cut them to it · `getLensRoot`: the base lens of a chain.</small>

### 4 · Sources: a column's contents as an option set

#### sources
**Solves:** turn the live values of a column into a field's selectable options (a "pseudo-enum"), scoped by the lens. The engine compiles the query; the app executes it.
**How:** `sources: { field: Condition | SourceSpec }` on a narrowing node: a bare eligibility `where`, or `{ where?, label?, groupBy? }`, or `{ from: 'mapDefaults' }` to offer the model's own source. The `where` composes like `where` (AND-only). The app runs the compiled DISTINCT query and gets back `SourceValues = { path, mapName, model, field, options }`, which `projectLens(lens, { sourceValues })` attaches to each field as `options` (§3). Wire format: `{ lens, sourceValues }`: both serializable.
<small>API · `toSourceQueries(lens)`: one `SELECT DISTINCT` per sourced field, emitted as both Prisma and SQL (`sql` is `null` with an `error` where SQL can't express the where; a source read across a bridge over-fetches and carries a `recheck`) · `materializeSourceQuery(query, rows)`: one query's fetched rows as `SourceValues` · `materializeSources(lens, rows)`: every source from rows already fetched under the lens.</small>

### 5 · Enforcing & describing rules against a lens

#### Lens enforcement
**Solves:** make a lens constrain rules: decide whether a rule is expressible and allowed here, report what it touches, and carry the lens's row-filters into it.
**How:** static analysis + injection over a rule against the projected lens.
<small>API · `validateRuleInLens(rule, lens)`: author-time gate (hidden fields, relations the lens doesn't turn on, out-of-set values) → `{ ok, errors }` · `describeRule(rule, lens)`: classify (maps read, bridge-crossing, supported targets) · `describeRuleSources(rule, lens)`: the source values a rule names · `narrowRule(rule, lens)`: AND the narrowing's `where`s into the rule at their anchors (into the window `filter` under `all`) · `coerceRule(rule, lens)`: stamp each field rule's `coerceType` from the lens · passing `lens` to `toPrisma` / `toSql` gates, narrows and compiles in one call.</small>

### 6 · Compile targets: push the predicate to the store

#### Compile targets
**Solves:** run a `Condition` where the data lives, as a prefilter or the full filter where expressible.
**How:** the same AST, two compilers.
<small>API · `toPrisma(condition,{map,mapName,model} or {lens})` → a plan `{ steps }` (with `groupBy` steps for count and aggregate operators) that `executePrismaPlan` resolves to a Prisma `where` · `toSql(condition,{map,model,alias} or {lens})` → `{ sql, params, joins }` for PostgreSQL (array predicates with conditions throw; run them through `toPrisma` or `check`).</small>

#### Inline predicates
**Solves:** a rule, or a lens's row filters, as one SQL fragment for a caller that names its own columns and writes its own joins.
**How:** `toWhereSql` and `toLensWhereSql` write the engine's `sql.dialect`: PostgreSQL by default, MySQL when `engineGlobals` sets `SqlDialect.mysql`. They take `equals`, `notEquals`, `in`, `notIn`, `exists` and `notExists` inside `all`/`any`, and `any`/`all`/`none`/`empty`/`notEmpty` over a to-many relation, written as an `EXISTS` subquery (or read off the caller's own join with `toMany: 'join'`). Any other operator throws and names itself. `toSql` and `toPrisma` are unchanged.
<small>API · `toWhereSql(condition, { columns, tables?, map, mapName, model })` → one predicate string · `toLensWhereSql(lens, { columns, tables?, toMany? })` → `{ root, relations, mapDefaults }`: the root path's `where`s, each relation path's own, and one predicate per model default.</small>

### 7 · Permissions: who may do what (`@inixiative/permissions`)

#### @inixiative/permissions
**Solves:** who may do what, decided on your own records: roles, relations and conditions in one engine, with delegation that never outlives whoever granted it, and no second store to sync.
**How:** a schema maps each resource's actions to an `ActionRule`, all plain JSON, and the checks walk your records through a `SchemaMap`: the resources, the field map (a prisma-map map, as `fieldMap: { models: prismaMap }`) and the polymorphic models' kinds. A **permix** holds what the actor holds, its **permissions**: role grants plus **entitlements**, each backed by its grantor and asked again at every use. `check` decides on a loaded record. The **core** (about a thousand lines) never imports the **helpers** (delegation, hydration). 1.0, by the end of 2026, adds **reflection**: the same rules compiled to Prisma and SQL, so "which records can I read" and "who can read this" are one query each.
<small>API · `createRebacCheck` · `createPermissions` · `createCanGrantEntitlement`: may this writer hand that entitlement on · `validateActionRule`: check a stored rule before saving · helpers: `delegate` / `capAtRole` / `installPermissions` / `permixOf` / `createHydrator`</small>

### 8 · Transitions: guarded state changes (`@inixiative/transitions`)

#### @inixiative/transitions
**Solves:** declarative, serializable state-machine **guards + affordances**: "can this record move from A→B, and who may?"
**How:** a `TransitionMap` (resource → action → `Action`, the OR of its `from`/`to` edges); each side holds a `predicate` `Condition` and an optional `permission` `ActionRule`, and the `to` side an optional `Merge`. `from` is checked against the current record, `to` against the merged one. Predicates are json-rules `Condition`s run by `check`; permissions go through an injected `Authorize` seam, which `createAuthorize` fills from the permissions library. The template has not adopted it yet: template ticket FEAT-018 has the library done and the move of its first consumer, the inquiry status lifecycle, still open.
<small>API · `checkTransition`: guard a proposed move · `available`: list the legal actions for a record · `eligible` (via `toPrisma`): every record currently eligible for an action · `validateTransition`: check one transition before saving it · `applyMerge` / `isSerializableMerge` · `createAuthorize`.</small>

### 9 · Builders: headless UI on the lens (`@inixiative/rules-builder`)

#### Rule builder
**Solves:** build/edit a `Condition` **headlessly**: logic in, descriptors out, you render.
**How:** a hook owns the JSON value and emits a recursive `GroupNode`/`LeafNode`/`ArrayNode` descriptor tree; each node states *what controls exist* (field/operator/value, each `{ value, options, set }`, with per-control `valid`) plus bound actions (`addRule`, `remove`, …). It renders nothing; you wire your own components (two copy-paste reference renderers, plain and shadcn, are provided).
<small>API · `resolve(source,{sourceValues})`: the narrowed lens as a `LensView`, each visit resolved on demand · `useRuleBuilder(...)` → `{ value, root }` descriptor tree · `describeModelFields(lens, mapName, model)` → per-field `BuilderField` (operators per family, `options`/`enumValues`).</small>

#### lensValuePicker
**Solves:** "pick any value reachable through a lens": the shared atom behind a rule's `field` (LHS) and `path` (RHS reference), reused downstream (permissions, email).
**How:** enumerates leaf values across relations as dotted paths, with kind and allowed values. A `Json` column is flagged `acceptsSubPath` (it has no declared sub-fields, but the kernel resolves *and* validates a dotted JSON path in `check`/`toPrisma`/`toSql` + `validateRuleInLens`), so a renderer can offer freeform descent, e.g. `metadata` → `metadata.theme`.
<small>API · `lensValuePicker(lens,{maxDepth})` · `useLensValuePicker` · `LensValueOption.acceptsSubPath` (JSON freeform sub-path).</small>

### The one-line version

prisma-map supplies the **field map**; `Condition` is the atom written against it; a **lens** scopes the map; **sources** turn data into options; `validateRuleInLens` enforces; `toPrisma`/`toSql`/`toWhereSql` push to the store; **permissions** decides who may act with the same rules; **transitions** guards state changes with both; **builders** edit rules headlessly. Permissions 1.0 reflects its rules to the store the same way.

---

## Part II · Developer experience

*Why: tools for the people and agents working in the code. They change how you find, read, check and release code, and none of them ships in your product. Template's worktrees, init and CI rules belong here too; they are listed with the template in Part III.*

#### atlas (`@inixiative/atlas`)
**Solves:** find code by concept, role or dependency instead of by folder, with a map that CI keeps true.
**How:** each file carries an `@atlas` block: `@kind` (its role), `@partOf` (its concepts), `@uses` (the dependencies it relies on) and `@constructs`. Blocks are checked against the repo's `.atlas/` registry (`config.ts` for stamp and include rules, `kinds.ts` for the role vocabulary, `concepts.ts` for the concepts), and `MAP.md` is generated from them. Agent hooks print a file's concept neighbourhood on every Read and Grep.
<small>CLI · `init` · `graph` (reverse indexes) · `query` (by kind / partOf / uses / path / json-rules) · `check` and `coverage` (CI) · `generate` (write `MAP.md`) · `report` (`COVERAGE.md` and `atlas.html`) · `stamp` (fill derivable axes) · `hook read` / `hook grep` (agent hooks) · `setup`</small>

#### gloss (`@inixiative/gloss`)
**Solves:** source files free of comments, with the explanations still beside the code for when the code leaves you confused.
**How:** commentary lives in a mirrored sidecar under `.gloss/` (`src/foo.ts` → `.gloss/src/foo.ts.md`), one section per symbol, committed and reviewed like docs. Source keeps only daggers that mark a note exists (`// gloss`, `// gloss:file`), machine directives and the rare one-line `// why:`. The harvester moves every other comment into the sidecar at commit, and `gloss read` dates each note against its symbol's git history, so you can see when a note is older than the code.
<small>CLI · `lint` · `check` (dagger and section audit, blocking in CI) · `fix` · `harvest --staged` (pre-commit) · `watch` · `read` · `history` · `setup` · `hook read` / `hook grep` (agent hooks)</small>

#### config (`@inixiative/config`)
**Solves:** one toolchain, one blessed set of versions, one set of local ports and one look across every inixiative repo.
**How:** TypeScript, Biome, tsup and lefthook presets; a version manifest (`versions.json`) naming the blessed version of each package per lane; the port registry (`ports.json`: a service's default port plus 100 per project); and the theme. `check` fails CI on drift, retired terms in public copy included, and `sync` fixes it. `train` is the release cascade: the `primitives` lane ships first, then `agentic`; each new version is recorded as a candidate, adopted by the repos that depend on it, then blessed.
<small>Theme · `theme/public.css` (the brand: base, `--app-*` slots, status, UI and type tokens, fonts) · `theme/marketing.css` (fluid type scale, spacing, section rhythm, motion) · `theme/domains.css` (four palettes on one structure, set by `data-domain`: agentic, building-blocks, governance, inixiative) · `theme/status.css` (thread and work status and badge layers, for apps that show agent work)</small>
<small>CLI · `check` · `sync` · `scan` · `train --lane=primitives|agentic` · `ports [project]`</small>

---

## Part III · The template

A production SaaS foundation (`@inixiative/template`) built on the building blocks in Part I. Teams building a SaaS product rebuild the same concerns on every product: auth, tenancy, permissions, side effects, background work, caching, real-time, audit. The template solves each *once*, names it, and reuses it. Each section opens with the problem that whole layer addresses; each entry is a **concept**, with its methods in a dim `API ·` row.

### Core platform & build

*Why: a monorepo of four apps + shared packages only stays fast and consistent if builds, environments, and local services are uniform and cached. This layer makes "clone and run" and "change shared code, everything reloads" true.*

- **Four apps**: one platform, four perspectives: `web` (participant), `admin` (operator), `superadmin` (platform ops), `api` (Hono backend).
- **Bun workspaces monorepo**: share code without publishing; `apps/*` + `packages/*` (`db`, `shared`, `ui`, `permissions`, `email`, `sdk`).
- **Turborepo**: task graph with caching + dependency-aware watch. <small>CLI · `turbo watch local` runs all apps · `#<app>` filters one</small>
- **Generation chain**: `db:generate → generate:openapi → generate:sdk → generate:routes`; `.gen.` files are template-owned, git/lint/search-ignored.
- **Path aliases**: `#/` app-internal, `@template/*` cross-package; `../` relative imports forbidden outside barrels.
- **Barrel files (top-level only)**: only package/module `index.ts` re-exports; everything else imports source directly.
- **Environment names**: `local | test | pr | staging | prod`; `isLocal/isTest/…` flags; `.env.<env>` files (local/test) + Infisical (cloud).
- **with-env**: run any command with the right env loaded. <small>CLI · `bun run with <env> <app> <cmd>`</small>
- **PR environments**: ephemeral per-PR stack (DB branch, previews, isolated secrets), destroyed on close.
- **Docker Compose dev services**: local Postgres/Redis/MinIO; pinned project name so worktrees share one stack.
- **init vs setup**: `init` = one-time provisioning (Ink TUI, or headless `init:agent`: secrets/services); `setup` = per-dev bring-up (deps/Docker/Prisma/seed).
- **Worktree tooling**: isolated parallel checkouts sharing one container stack; each slot gets its own databases, buckets, Redis DB and ports. <small>CLI · `bun run worktree:create` / `worktree:list` / `worktree:destroy`</small>

### API & routing

*Why: every endpoint should look the same, validate itself, and document itself, so you write business logic instead of boilerplate, and the frontend gets a generated typed client.*

- **File-based route auto-registration**: drop route+controller in a module; wired on startup. <small>API · `autoRegisterRoutes()`</small>
- **Route templates**: consistent OpenAPI 3.1 + ~70% less boilerplate. <small>API · `createRoute` · `readRoute` · `updateRoute` · `deleteRoute` · `actionRoute`</small>
- **makeController**: type-safe handler bound to a route; only the route's declared statuses are respondable (wrong-shape responses caught at compile time). <small>API · `respond.ok`/`respond.created`/`respond.noContent` → the uniform `{ data }` envelope</small>
- **Errors**: one error contract `{ error, message, guidance, fieldErrors?, requestId }`; a global `onError` normalizes Zod/Prisma/HTTPException/raw and stamps `requestId`; drift-free labels per status code. <small>API · `makeError`/`AppError` · `HTTP_ERROR_MAP` · `errorHandlerMiddleware`</small>
- **paginate()**: list-endpoint orchestrator: applies search + filter + ordering + pagination from the request; returns `{ data, pagination }`.
- **Bracket-notation query language (`bracketQuery`)**: one URL syntax for **filtering and ordering**: `filter[user.email][gte]=…` and `?orderBy[]=field:dir` (GraphQL-like, URL-based, cacheable); the `[:]` marker carries real `null`/`true`/`false` through all-string params (allowlisted, no eval); `dialect` handles provider-specific JSON-path filtering. <small>API · `parseBracketNotation` → `BracketQueryRecord` (the `bracketQuery` ctx var) · `buildWhereClause` (filters → Prisma `where`, lens-validated, the lens's root wheres ANDed in) · `buildOrderBy` (sort) · `@template/shared/bracketQuery` (symbols/operators) · FE: `buildFilterQuery`/`serializeBracketQuery`</small>
- **Batch API**: many requests in one call; strategies `transactionAll`/`transactionPerRound`/`allowFailures`/`failOnRound`; `<<roundIndex.requestIndex.path>>` interpolation.
- **OpenAPI 3.1 generation**: full 3.1.0 spec served as JSON at `/openapi/docs` (via `app.doc31`); drives the SDK.
- **Generated SDK (`@template/sdk`)**: hey-api typed client + TanStack Query hooks + query keys from the OpenAPI spec.
- **Module structure**: allowlisted folders per `modules/<name>/`: routes/controllers/services/schemas/queries/tests/…; shared code → `lib/`.
- **Router exports / Modules constant**: `<resource>Router`/`admin<Resource>Router` exported from each module's `index.ts`, mounted in `routes/api.ts` / `routes/admin.ts`; type-safe model-name registry (`Modules.organization`, not `'organizations'`).

### Request context & errors

*Why: multi-tenant code must never leak across requests, so identity, tenant, db client, and permissions are isolated per request and flow implicitly, not threaded by hand.*

- **prepareRequest**: first middleware after CORS; sets db/requestId/permix, nulls auth, parses `bracketQuery`, wraps in `db.scope`.
- **requestContext (AppEnv)**: typed per-request Hono Variables (db, user, token, permix, resource, filterLens, bracketQuery, …).
- **permix.getActor**: the actor the rules see, `{ model, record }`: the user for a JWT or spoof, the token's own columns for an API token. <small>API · `GET /api/v1/identify` answers it for any credential</small>
- **getResource / resourceContextMiddleware**: auto-load the `:id` resource (404 on 0, 409 on >1) before authorization.
- **Per-request scope (bind tokens)**: a route's static `filterLens` carries `bind` tokens for tenant/ownership scope; the controller passes their values to `paginate`, which refuses a lens with a token missing.
- **Database scope (AsyncLocalStorage)**: request/job correlation id. <small>API · `db.getScopeId()` · `db.isInTxn()`</small>
- **getValidatedBody / getValidatedQuery**: typed validated input outside route-typed context.

### Adapters

*Why: external services (storage, email, error reporting) change per environment and shouldn't leak SDK details into app code. One interface, swappable providers, chosen per environment.*

- **Adapter pattern**: couple to swappable external SDKs via a role interface + providers + env wiring. <small>API · `makeAdapterRouter` (one provider per env) · `makeAdapterRegistry` (pick by name) · `makeBroadcastRegistry` (fan out via `Promise.allSettled`)</small>
- **storage adapter (s3)**: provider-agnostic blob storage (presigned POSTs); MinIO locally.
- **errorReporter adapter**: console locally, Sentry in prod, behind the router.
- **email client adapter**: `EmailClient` with Resend (prod) / Console (dev) providers, per-tenant keys.

### Database & ORM

*Why: correctness, type safety and consistency are decided in the database layer. These primitives make Prisma transactional by default and introspectable at runtime, and they reject or disable the writes that would skip hooks.*

- **prismaMap (`@inixiative/prisma-map`, Part I §1)**: the runtime model/field/relation map built from the generated client; hooks, factories, lenses and the permission checks read it.
- **Model name utilities**: `ModelName` ↔ `AccessorName` conversion. <small>API · `isModelName` · `isAccessorName`</small>
- **UUIDv7 IDs**: time-sortable PKs without coordination; better index locality than v4.
- **db client (txn-aware proxy)**: routes to the active transaction when present, else raw. <small>API · `db.scope` · `db.txn` (auto-merging) · `db.onCommit` (after-commit side effects) · `db.findForUpdate` (row lock, txn-only) · `db.raw.*` (bypass)</small>
- **assertNoNestedWrites**: reject nested writes that would skip lifecycle hooks (validation/audit).
- **Disabled createMany/updateMany**: they return no rows (break hooks); use `*AndReturn`.
- **pass-the-delegate**: pass a concrete delegate so TS infers exact types for dynamic model access. <small>API · `query.findMany(db.user, …)` · `RuntimeDelegate`/`AnyCrudDelegate` (runtime-only access)</small>
- **hydrate**: load a record's owning to-one relations along `schemaMap`, each parent read through `fetchOne` under its own cache key; the permissions library's `createHydrator`. <small>API · `hydrate(accessor, record)` · `fetchOne`</small>
- **Lens helpers**: json-rules lenses over the Prisma schema, plus narrowing layers and a fetch through them. <small>API · `lensFor` (base lens for a model) · `redactLens` (omit redact fields) · `omitForeignKeys` · `fetchLens` (`toLensSelect` + `projectRows`) · `searchablePaths` / `orderablePaths` · conditions `live` / `boundAndLive` / `platformOrBound` / `through`</small>
- **Path notation**: safe dot-notation nested queries (charset/depth guarded). <small>API · `buildNestedPath` · `validatePathNotation`</small>
- **Seed (SeedFile / `--prime`)**: FK-ordered idempotent upserts by UUIDv7; `prime` records are dev-only.
- **Split schema**: one `.prisma` file per model, combined at generation (fewer merge conflicts).
- **Zod schema generation**: `<Model>ScalarSchema`/input schemas derived from Prisma.

### Mutation lifecycle & hooks

*Why: cross-cutting concerns (audit, validation, cache, webhooks, ordering) must fire on every mutation, inside its transaction, without sprinkling calls everywhere. One extension runs them declaratively.*

- **mutationLifeCycle extension**: Prisma client extension wrapping create/update/delete/upsert with auto-txn + hooks + previous-state fetch. <small>API · `registerDbHook` / hook registry · `registerHooks()` at boot · hook payload `{model, action, args, result, previous}` at `before`/`after`</small>
- **auditLog hook**: immutable before/after/changes diff + actor context for `AUDIT_ENABLED_MODELS`.
- **immutableFields hook**: strip `id`/`createdAt`/FKs (and overrides) on update; silently, never throws.
- **rules hook + RulesRegistry**: declarative per-model field validation via json-rules `Condition`.
- **shadowMerge**: merge `previous` with update data so atomic ops (`increment`/`push`) validate against final state.
- **cache hook + `cacheReference`**: invalidate cache keys a mutated record is reachable by, post-commit (per-model key map in `hooks/cache/constants/cacheReference.ts`).
- **webhooks hook + circuit breaker**: enqueue signed `sendWebhook` post-commit; auto-disable a subscription after 5 fails.
- **orderedList hook**: dense `[1..N]` position columns per scope, bulk-safe via SQL CTEs; soft-deletes get negative sentinels.
- **contactRules hook + ContactRegistry**: per-`ContactType` parse/canonicalize/validate/uniqueness.
- **False polymorphism + PolymorphismRegistry**: type enum + separate FK columns (not a real polymorphic FK); one source of truth for constraints/rules/immutability.
- **preventHardDelete / soft delete**: `deletedAt`; `SOFT_DELETE_MODELS`; `null→timestamp` recorded as a delete.
- **HOOK_IGNORE_FIELDS / isNoOpUpdate**: skip audit/cache/webhooks when only noise fields changed.
- **auditActorContext**: AsyncLocalStorage actor (user/token/spoof/job/integration/IP/inquiry) auto-attached to mutations/events.

### Auth & sessions

*Why: support every auth shape (a user JWT from email/password or OAuth, an API token, impersonation) behind one resolved actor, so handlers never care how the caller authenticated.*

- **BetterAuth handshake**: BetterAuth runs sign-up, sign-in, email verification and OAuth; the session it creates lives at most 10 minutes, and `jwtHandoff` swaps it for a user JWT before the response leaves the API. <small>API · `jwtHandoff` · `withThrowawaySession` (call BetterAuth's session-bound account endpoints server-side)</small>
- **User JWT**: ES256, 30 days, kept by the client and sent as `Authorization: Bearer`; bearer-only, so no cookie or session ever authenticates an API request. It carries `User.jwtVersion` as `ver`; incrementing the version signs the user out everywhere. <small>API · `jwtAuthMiddleware` · `rotateUserJwts`</small>
- **secondaryStorage (Redis)**: holds BetterAuth's handshake sessions and the OAuth one-time tokens; the Prisma `Session` model holds no rows.
- **AuthProvider (per-org OAuth/SAML)**: org-scoped (`organizationId` required) OAuth/SAML config with encrypted secrets; platform-wide providers are env-driven via `getPlatformProviders()`, not `AuthProvider` rows.
- **Token authentication**: `Authorization: Bearer`/`?token=`, SHA-256 hashed, Redis-cached; `keyPrefix` identifies without exposing.
- **Token owner / hierarchical scope**: polymorphic owner (User/Org/OrgUser/Space/SpaceUser/Integration); the owner determines reach.
- **Auth middleware chain**: `prepareRequest → jwtAuthMiddleware → tokenAuthMiddleware → spoofMiddleware → auditActorMiddleware`; a JWT-shaped bearer is the user's JWT, any other bearer an API token.
- **setUserContext / refreshUserContext**: populate user/org/space + permissions; rebuild mid-request after changes.
- **Spoofing (impersonation)**: superadmin `x-spoof-user-email`; records `spoofedBy`; response headers + UI badge for auditability.
- **Auth gates**: there is no "is there a user" gate: every route names its permission action, and user-only acts are rules on the actor (`actorIsUser`). <small>API · `validateSuperadmin` (the admin router) · `basicAuthMiddleware` (HTTP Basic for utilities, e.g. BullBoard)</small>

### Authorization (RBAC / ABAC / ReBAC)

*Why: access decisions in a SaaS app use roles, attributes and relationships at once. One engine expresses all three as data, so access rules are rows a tenant can own, not scattered `if` checks or a policy file shipped on deploy.*

- **ActionRule**: the algebra: a string (another action on the same resource) | `{ rel, action }` (walk a relation, ask the related record) | `{ rule }` (a json-rules condition over the record and `__actor`) | `any` / `all` | `true` / `false` / `null`. Positive only: you narrow by composing, there is no deny.
- **rebacSchema**: the template's actions per model. <small>API · `ownerActions()` (across owner relations) · `memberRowActions` (member rows) · routes declare checks via `validatePermission` / `validateSubmodelPermission`</small>
- **Role + StandardAction**: `owner > admin > member > viewer`, mapping to `own > manage > operate > read`. <small>API · `roleToStandardAction`</small>
- **PlatformRole / superadmin bypass**: `user | superadmin`; superadmin short-circuits every check.
- **Permix**: the actor's installed permissions for one request. <small>API · `createPermissions()` · `setup` · `setActor` · `setSuperadmin` · `installPermissionsFor`</small>
- **Permissions**: what the actor holds: role grants plus entitlements, and for a delegate, `full`. <small>API · `permissionsOfUser` · `ownerPermissions` · `delegatedPermissions`</small>
- **Entitlement**: an action on a resource, from an anchor record, narrowed by an optional `where`, backed by whoever granted it. Every check asks the grantor again, up the chain; demote anyone and what they handed on stops. <small>API · `changeEntitlements` (add / remove) · the library's grant check decides who may hand one on</small>
- **Delegate**: a token or integration acting for its owner: `full` (the owner whole, in sync), a role (the owner capped at that role, `capAtRole`) or entitlements. Never more than its owner. Setting `full` takes the person-only `delegateFull` action.
- **Member self-grant**: every member row's entitlement to `operate` on itself, backed by its user; written on create and kept by a rule.
- **Row-level rules (`permissionRules`)**: a rule a tenant writes on one record (`Contact` for `read` today), validated before saving and ORed with the schema, so it can only widen access. Ad hoc sharing without a schema change.

### Encryption

*Why: sensitive fields must be encrypted at rest with rotation and tamper-detection, but adding a field shouldn't require touching hooks/rotation/validation. A registry makes it one line, and the hooks discover it.*

- **AES-256-GCM field encryption**: authenticated per-field encryption, random IV, auth tag. <small>API · `createEncryption(keyring)` → `{ encrypt, decrypt }` · `EncryptedFieldData` = `{ciphertext, version, iv, authTag}` (base64)</small>
- **AAD (buildAAD)**: bind ciphertext to immutable record fields; prevents transplant to another record.
- **ENCRYPTED_MODELS registry**: single source of which fields encrypt; hooks/rotation/validation auto-discover.
- **Column-triplet convention**: `encrypted{Field}` + `…Metadata` + `…KeyVersion`. <small>API · `encryptField`/`decryptField` generics</small>
- **EncryptionKeyring / key versions**: current + previous key for dual-key zero-downtime rotation.
- **rotateEncryptionKeys (singleton job)**: idempotent re-encrypt via `updateManyAndReturn where version=N`.
- **`validateEncryptionVersions`**: rejects key-version gaps/downgrades/mixed versions (called from its tests only).

### App events & jobs

*Why: business logic must never call email/SMS/analytics directly. It emits an event; bridges and background jobs do the side effects reliably, after commit, off the request path.*

- **emitAppEvent**: fire a business event decoupled from side effects; auto-enriches actor. <small>API · `makeAppEvent` (define one event's fan-out) · `appEventHandlers` (typed name→handler map; `onCommit` deferral in txns)</small>
- **handoffs**: email (`EmailHandoff` → `sendEmail` planner job), websocket (`WSHandoff`: a channel refetch via `sendToChannel`, or a stream append built by `streamAppend`), callbacks (`cb`), and observe (the envelope to the log and an `AppEvent` row via `observeRegistry`), each independent via `db.parallel(…, { resolution: 'allSettled' })`, failures thrown after all settle.
- **enqueueJob**: typed central job dispatch with overflow protection + dedupe. <small>API · `makeJob` (basic) · `makeSingletonJob` (one-at-a-time, lock) · `makeSupersedingJob` (newest cancels older, lanes)</small>
- **jobHandlers / ctx.log**: typed name→payload→handler registry; dual-write logging (stdout + BullBoard).
- **Worker / BullMQ**: concurrency 10 by default (`JOBS_WORKER_CONCURRENCY`), retries, graceful shutdown; own Redis connection.
- **BullBoard**: queue/job monitoring UI; receives worker logs via broadcast.
- **Job overflow buffer (JobOutbox)**: adhoc jobs spill to DB rows past the queue depth budget (`JOBS_MAX_QUEUE_DEPTH`). <small>API · `accumulator` (batches the writes) · `drain` loop (re-admit FIFO under a lock) · `quarantine` (poison rows after N attempts) · overflow flag / `queueDepth` (set-once trip flag + cached waiting+active probe)</small>
- **Cron jobs / JobType**: DB-defined schedules registered at startup; `cron|cronTrigger|adhoc`.
- **messaging**: non-email multi-channel dispatch. <small>API · `messageUser`/`messageContact` · `messageProviderRegistry` (pluggable per-`ContactType` adapters)</small>

### Concurrency

*Why: a horizontally-scaled app needs cross-process mutual exclusion, supersession, serialization, and bounded parallelism. Small primitives keep batches from exhausting pools or interleaving.*

- **Distributed lock**: single-node cross-process mutual exclusion (heartbeat/TTL, `onLockLost`). <small>API · `createLock` · `@template/db`</small>
- **Supersede lane**: cross-process "last enqueue wins" baton for superseding jobs; own `lane:` Redis namespace. <small>API · `claimLane` · `watchLane` · `laneKey` · `@template/db`</small>
- **Serialized writes**: serialize concurrent async writes to one resource (in-process promise chain). <small>API · `createSerializedQueue`</small>
- **Bounded parallelism**: parallelism caps per resource class; bounded `Promise.all`. <small>API · `getConcurrency` · `resolveAll`</small>
- **Recurring timer**: non-overlapping recurring async task with a clean `stop()` (the primitive under lock lease-renewal and lane usurp-polling). <small>API · `heartbeat`</small>

### Redis & caching

*Why: caching is only safe if invalidation is automatic and keys never collide. A namespaced key scheme plus mutation-driven invalidation makes reads fast without staleness bugs.*

- **Redis connection strategy**: main shared client + dedicated subscriber + BullMQ's own. <small>API · `createRedisConnection` · `getRedisClient`/`getRedisSub`</small>
- **redisNamespace**: central key-prefix map (`bull`/`cache`/`cacheWritten`/`job`/`ws`/`otp`/`session`/`limit`/`lock`/`lane`/`rollingCache`).
- **Cache**: get-or-set memoization with TTL; concurrent misses on one key compute once (a single-flight lock) and null results get a short TTL; canonical `cache:{accessor}:{field}:{value}[:tags][:*]` keys; a reverse-reference map clears the right keys on mutation. <small>API · `cache` · `checkCache` · `upsertCache` · `cacheKey` (a key containing `undefined` is refused on read and write) · `clearKey` (exact/wildcard) · `flushRedis` (tests, which run on an in-memory ioredis-mock)</small>
- **Rolling cache**: a value computed from stored data only to serve it fast (a count, a rollup, a picker's vocabulary read on every load) is a rolling cache, not a table. It recomputes on a fixed interval, and its entries outlive the longest a refresh can take, so a read always hits and there is no invalidation to write. One definition, in the module that owns the data, declares its `targets` (a model's own Prisma `findMany` args; each row is one refresh), its results and its interval, and is registered in `apps/api/src/lib/rollingCaches.ts` (CI rule `rolling-cache-registered`). A read never computes. <small>API · `makeRollingCache` · `checkRollingCache` (the api read: on a miss it queues that target's refresh and returns the channel to wait on) · `read` / `refresh` on a cache · jobs `coordinateRollingCaches` (root, every minute) → `refreshRollingCache` (branch, one per cache) → `refreshRollingCacheTarget` (leaf, one per target, starts spread over `spreadSeconds`)</small>

### Realtime / websockets

*Why: clients need instant "your data changed" signals across load-balanced servers, without leaking data or hand-managing subscriptions. Refetch triggers over Redis pub/sub do it.*

- **WebSocket server**: native Bun WS on the Hono port; anonymous at the upgrade, then the credential (a user JWT or an API token) arrives in an in-band `authenticate` frame and is resolved through `/api/v1/identify`.
- **Connection lifecycle / registry**: track by id/user/channel/stream; identity via `authenticate` frames (spoof and unspoof re-send one) and `logout`; changing the user or credential drops every channel and stream.
- **channelKey / WSEvent**: query identity key; a channel carries a data-free `{category:'query', action:'refetch', key}` hint, and a subscription is authorized by probing the channel's own HTTP route. Data streams carry rows instead, as a `snapshot` then sequenced `append` frames.
- **Pub/sub fan-out**: publish to the `ws:broadcast` Redis channel → every instance delivers locally. <small>API · `sendToChannel` · `appendToStream` · `broadcast` · `initWebSocketPubSub`</small>
- **LIVE_QUERIES**: operationIds with a BE producer, derived from the `WS_CHANNELS` registry; the FE subscribes only to these.
- **createApiWebsocket / client pipe**: channel refcount, reconnect, send-queue; queryCache subscriber → auto subscribe/invalidate.
- **WS heartbeat / stale sweep**: ping/pong liveness; close idle connections; FE force-reconnect.

### Communications / email

*Why: transactional email must be tenant-overridable, idempotent, consent-aware, and reconstructable. A DB-driven template system + a planner/deliver pipeline gives at-most-once sends you can audit.*

- **EmailTemplate / EmailComponent**: tenant-overridable MJML templates + reusable blocks (`{{#component:slug}}`).
- **Render pipeline**: fetch + recursively expand components + substitute `{{sender/recipient/data/system.*}}`. <small>API · `composeTemplate` / `composeComponent` · `expand` · `interpolate` · `evaluateConditions` (`{{#if rule=…}}` via json-rules; a degraded or throwing rule renders nothing)</small>
- **Cascade resolution (lookupCascade)**: owner chain Space→Org→default (and the user chain: SpaceUser, OrgUser, User, default) picks the right-tier template.
- **EmailOwnerModel / inheritToSpaces**: false-polymorphic ownership: default/admin/Org/Space/User/OrgUser/SpaceUser.
- **Render issue policy (`render.onIssue`)**: set on the template's registry entry: `platform` (default: re-render from the platform tier's row of the same slug) / `fail` (retry→DLQ) / `degrade` (send with failing blocks and tokens empty, issues recorded) / `substitute` (render another template instead).
- **sendEmail planner → deliverEmail**: two-stage fan-out: planner finds-or-creates one log per recipient, then one deliver job each.
- **idempotencyKey**: event-anchored hash-last key collapses duplicate sends but re-sends on payload change.
- **CommunicationLog**: per-recipient delivery ledger + at-most-once fence; stores the settled MJML, variables and render issues.
- **canDeliver (acceptedKinds)**: per-`Contact`/`CustomerRef` consent by `CommunicationKind`.
- **Unsubscribe (HMAC link)**: signed one-click unsubscribe (RFC 8058 headers, POST-only) for every non-`system` email; one kind for one contact.
- **Deliverability cache**: bouncer verdict cached on `Contact.deliverability` (TTL 30 days); skip undeliverable pre-send.
- **Email versioning / recompose**: audit-log version graph reconstructs exactly what was sent.
- **Email clients (Resend/Console)**: pluggable `EmailClient` per environment.

### Frontend architecture & state

*Why: three apps share one state model and one API contract. A single composable store + generated SDK keep client state, tenancy, and server data coherent without per-app forks.*

- **useAppStore**: one shared Zustand store composing 6 slices; consumed via `@template/ui/store` (all three apps). <small>API · `create<X>Slice` composition → `AppStore` = their intersection</small>
- **Store slices**: `AuthSlice` / `TenantSlice` / `PermissionsSlice` / `NavigationSlice` / `ClientSlice` / `UISlice` (identity / tenant context / FE ReBAC / nav config / QueryClient+WS / theme); every app composes the same six.
- **File-based routing (TanStack Router)**: `createFileRoute` under `app/routes/`; nesting via `<Outlet/>`; layout routes `_authenticated` (guard+AppShell) / `_public` / `_fullscreen`.
- **Guards**: redirect in `beforeLoad`, preserve `redirectTo`/context. <small>API · `requireAuth` · `requirePublic`</small>
- **useAuthenticatedRouting**: sync tenant context + spoof between URL and store; route permission check.
- **Navigation config (`NavConfig`/`NavItem`)**: declarative, permission-filtered nav: context-keyed (`user`/`organization`/`space`/`public`) recursive `NavItem` trees.
- **apiQuery / apiMutation**: typed SDK wrappers (unwrap `data.data` / keep full response); auth+spoof headers.
- **Theme system (three-tier CSS vars)**: `--app-*` → `--space-*` → `--primary`. <small>API · `useDarkMode` · `useSpaceTheme`</small>

### Frontend layout & UI

*Why: a consistent app frame and a set of conditional-aware primitives let every page look and behave the same with minimal code: permission-filtered, theme-aware and responsive by default.*

- **AppShell**: master authenticated layout: Sidebar (ContextSelector + nav + UserMenu) beside `<Outlet/>`.
- **ContextSelector**: switch Personal/Org/Space (supports `lockedContext` for white-label).
- **Sidebar / Header / UserMenu / Breadcrumbs**: permission-filtered nav / page header / account menu / location trail.
- **MasterDetailLayout / DetailPanel / ResponsiveDrawer**: list+detail; slide-out detail; drawer-desktop/modal-mobile.
- **Modal / FullscreenLayout / Toaster / ErrorBoundary**: overlay dialog / chrome-less protected page / toasts / render-error fallback.
- **UI primitives**: Button (CVA variants), Input, PasswordInput, SlugInput, Select, Card, Badge, Table, Avatar, EmptyState, DropdownMenu, Pagination, ThemeToggle, Label.
- **Conditional props (`show`/`disabled`/`disabledText`)**: uniform conditional render/disable across components.
- **SettingsLayout + context-aware tabs**: nested-route settings sidebar adapting to personal/org/space.

### Frontend data & hooks

*Why: list UIs all need the same search/filter/sort/pagination/scroll behavior, derived from the API's own schema, so tables and filters stay in sync with the backend automatically.*

- **usePaginatedData / useInfiniteData**: page-number / infinite-scroll controllers with URL + scroll persistence.
- **Data config**: derive searchable/orderable/enum fields from the OpenAPI spec; shared filter state. <small>API · `makeDataConfig` · `useDataFilters` · `useQueryMetadata` · `buildFilterQuery`/`serializeBracketQuery` (serialize to bracket-notation)</small>
- **Scroll/section hooks**: `useInfiniteScrollTrigger` (IntersectionObserver load-more) · `useScrollState` (restore) · `useSectionHash` (hash↔section deep-linking).
- **useOptimisticMutation**: optimistic cache updates with rollback. <small>API · `createOptimisticListTarget` (list create/update/delete)</small>
- **usePermission / useInquiryPermission**: turn a permission check into `{show, disable, disabledText}`.
- **Utility hooks**: `useAuthProviders` · `useValidateUniqueness` · `useDebounce` · `useMediaQuery` · `usePageMeta` · `useBreadcrumbs` · `useLanguage`.
- **makeContextQueries**: per-feature, context-scoped (public/user/org/space) query+mutation registries.

### Features (domain)

*Why: the recurring SaaS nouns (tenants, members, approvals, contacts, webhooks, audit) ship as first-class, consistent primitives instead of being re-modeled per product.*

- **organization / space**: top-level tenant; org-scoped sub-workspace with branding.
- **organizationUser / spaceUser**: membership joins with `Role` + entitlements; space membership requires org membership.
- **user / me**: platform identity; self-service `/me` surface (profile, contacts, tokens, webhooks, redact).
- **redact (GDPR)**: anonymize user PII while preserving referential integrity.
- **inquiry**: generic request/approval/audit primitive; polymorphic source/target, per-type handler, status state machine.
- **InquiryType / handler**: `inviteOrganizationUser`/`createSpace`/`updateSpace`/`transferSpace`; a handler = content+resolution schemas + `handleApprove`/`autoApprove`/`unique`/expiry.
- **transferSpace**: an inquiry from a Space to an Organization. Approving it moves the space and everything downstream into the target org in one transaction: members who are also in the target org keep their space membership and everything they own there, every other member is soft-deleted with their rows, and one Space update re-keys the rest. The old org keeps nothing of the space.
- **contact + ContactType (~22)**: one polymorphic-owner model for phone/email/social handles; per-type registry def; deliverability + `acceptedKinds`.
- **customerRef**: false-polymorphic customer ↔ provider link; each side a User, Organization or Space.
- **webhookSubscription / webhookEvent**: owner-polymorphic outbound subscriptions; per-attempt delivery ledger (RSA-signed).
- **auditLog**: immutable who/what/before/after with actor + tenant + polymorphic subject.
- **cronJob**: DB-defined scheduled jobs (pattern/handler/payload/enabled, retry config).
- **token / authProvider / account / session / verification**: API keys / org auth config / external credentials / BetterAuth's session schema (handshake sessions live in Redis) / verification values.
- **tag / tagCategory / tagAttachment**: owner-scoped labels grouped by category, attached to resources.
- **project**: a taggable unit of work (a name, an optional description) owned by any of the five owners (`ProjectOwnerModel`: User, OrganizationUser, Organization, Space, SpaceUser), its name unique per owner. Each owner kind has its own create and list (`meCreateProject` … `spaceUserReadManyProjects`).
- **Owner lists**: one list per owner kind returns exactly that owner's rows, filtered by the same placement the permission check used (`placeSubmodel` → `submodelOwnerWhere`), so an organization's list leaves out its members' rows. Members get their own token and integration lists (`/organizationUser/{id}/tokens`, `/spaceUser/{id}/integrations`, …), read by the member and the organization's owner; `/me/tokens` lists every token that is yours.
- **batch**: multi-request execution with strategies, interpolation, and `BatchStatus`.
- **entitlements / invitations**: entitlements are action grants beyond a role (see Authorization); invitation = the `inviteOrganizationUser` inquiry.
- **admin / superadmin surfaces**: privileged cross-tenant listing/management; `superadmin` app + permission bypass.

### Testing & quality

*Why: data factories that mirror the schema make tests fast to write and isolated, and CI gates enforce the repo's own conventions mechanically.*

- **Test factories (`build*`/`create*`)**: in-memory (`build*`) vs persisted (`create*`) records; auto-infer FK deps. <small>API · `createFactory(model, config)` · `getNextSeq()` (unique values) · `packages/db/src/test/factories`</small>
- **`__serialize()`**: convert `Date`→ISO to assert against API/SDK string-timestamp shapes (CI-enforced in UI tests).
- **createTestApp**: Hono app with a mocked user, memberships or token whose permissions install as in a real request; `mount` the routers under test.
- **request helpers**: concise HTTP calls + `{data, pagination}` parsing (`get`/`post`/…/`json`).
- **cleanupTouchedTables / `--max-concurrency=1`**: DB test isolation; serial test runs.
- **happy-dom**: browser-less DOM testing via `bunfig.toml` preload.
- **CI rules**: repo-specific rules beyond Biome. <small>API · `run-ci-rules.sh` (`--test` self-tests against fixtures) · `no-jest`/`no-vitest`/`no-radix`/`no-lens-spread`/`no-module-mocks`/`ui-serialized-factories`/`spy-must-restore` · `check-import-aliases.sh` + stale-generated-file checks</small>
- **`bun run check`**: canonical gate: lint + typecheck + backend tests + FE tests + CI rules.

### Observability & ops

*Why: in production you need to trace one request/job across every log line and catch errors without coupling app code to a vendor: scoped structured logging + optional otel/sentry.*

- **logger**: structured logging over a swappable adapter (pino deployed / consola local); server-only. <small>API · `LogScope`/`logScope()` (nesting tags, request/job `requestId`) · `addLogBroadcast` (extra sinks, e.g. BullBoard)</small>
- **frontendLogger**: browser-safe logger, never crosses with the server logger. <small>API · `createFrontendLogger`</small>
- **OpenTelemetry / Sentry**: logs, traces and metrics to any OTLP destination (exported in hosted environments when an endpoint is configured; local and test only with `OTEL_ENABLED=true`); Sentry error reporting behind the errorReporter adapter.

---

## Part IV · Agentic

The tools that carry what you've already decided into every agent session, and the hosted hub they connect through. Product lines are in the [org profile](https://github.com/inixiative/.github/blob/main/profile/README.md); this part names the words.

### Foundry and the chorus

*Why: one big prompt loses instructions in the sauce. Many narrow experts, each holding one part of how the codebase works, stay sharp and point the agent at what already exists.*

- **Foundry**: the workspace around your agent: Claude Code and Codex as workers, ACP apps (Paseo, T3 Code, Zed) as clients. It runs locally on your own subscriptions and wraps each thread with Wardens. The app is source-available (BSL 1.1); its core engine (`@inixiative/foundry-core`) and ACP adapter (`@inixiative/foundry-acp`) are MIT.
- **Warden**: an expert on one region of the codebase, built from that region's docs. On each turn it answers whether the moment touches its region and, if so, points at what already exists and where it's written. It comments; it never blocks. A Warden that times out stays silent for that turn, and the thread shows it.
- **Chorus**: a project's set of region Wardens, asked every turn. Each project has its own.
- **Layer 1 / Layer 2**: the chorus's two steps. Layer 1 (Haiku) detects: is this moment in my region, with a one-line why. On each yes, Layer 2 (Sonnet) confirms and picks the thing and where it's written. Layer 2 doesn't see Layer 1's why.
- **Herald** *(planned)*: carries live data where Wardens carry static knowledge: what other agents and teammates are working on, in Linear, GitHub and Slack. Not built yet.

### Kingdom and access

*Why: agents need access to your apps and data without you logging them in, and every grant needs an owner who can take it back.*

- **Kingdom**: the hosted hub at [kingdom.inixiative.com](https://kingdom.inixiative.com): identity, owners, integrations and Signets. Foundries, Archives and Oracle connect through it. Closed.
- **Owner**: who a thing belongs to, for tenancy: a user, an organization user (a person's membership in an org), an organization, a space, or a space user.
- **Project**: what an owner organizes work around: a name, an optional description, tags. A Signet names the projects it may be used in, and a Foundry project links to one Kingdom project.
- **Integration**: a provider connected to an owner (GitHub, Slack, a Claude or Codex subscription, a Foundry, an Archive). Its **Credential** is what Kingdom uses to call the provider: encrypted, outbound, never handed out. A Kingdom **Token** is a different thing: a key for Kingdom's own API, with a role.
- **Signet**: the only grant for using an integration: which resources and operations, through which lens, within which limits, until it's revoked. Its authority comes from its owner. It's presented with an enrolled key, and what's done through it is recorded as usage. `@inixiative/signet` (MIT) is pass-through execution with tracking and audit, built on the permissions library.
- **Signet grant**: one row per resource on a Signet's revision (`SignetResourceGrant`, for an integration resource or a project). Each row holds an operation selection (picks and omits over the integration's operation map: each model, then its operations and its events) and a json-rules lens narrowing over the integration's base lens for that model; no lens means the base. Every scope change is a new revision, and each revision applies at once. Recipients are never restricted: a grantor keeps an operation from one by leaving it out.
- **KCP** *(in progress)*: the Kingdom CLI Proxy. Your provider CLIs run through Kingdom with no credentials on your machine; Kingdom runs the command and streams the output back. MIT.
- **Subscription login and capacity pool**: Kingdom holds your Claude and Codex logins, each an integration on a personal owner. A project gets a pool (`CapacityPool`); for each run Kingdom picks a login from it (`RunBinding`) and releases its token, through a Signet (`subscription.use`), to a Foundry of the same owner. The machine runs the real harness with it, so inference never passes through Kingdom.
- **Integration events**: the integration owns the provider subscription (a webhook, or a poll where the provider has none), so nothing is set up on your machine. Each event reaches every Signet whose grant selects it, cut to the event's base lens and, where the resource takes one, filtered by the grant's own lens. It arrives as a live stream the holder opens over a WebSocket. With no holder connected it goes nowhere: there is no backlog. Built for GitHub (organization webhooks and a person's notifications poll), Slack (Events API) and Linear (webhooks Kingdom creates).
- **Artifactory** *(planned)*: one shared place to publish AI artifacts (guides, skill files, generated pages), signing in with Kingdom, which also decides who may see what. No code yet.

### Sessions and the loop

*Why: corrections only compile into docs if the sessions they happened in are kept, and only matter if they stop recurring.*

- **Archive**: keeps every agent session, local-first, and syncs to the archives you choose. MIT.
- **agent-session**: drives Claude Code and Codex as persistent streaming sessions behind one interface. MIT.
- **Oracle**: measures whether a correction stuck, and tunes the docs and Wardens. Closed.
