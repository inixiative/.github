# Vocabulary

We build the inixiative packages to **"solve it once"**: each concern has one home, and new features use that home instead of working around it. This page is the glossary of every primitive. It covers the **shared spine** first (the cross-library kernel: rules, lens, sources, builders, and the layers that compose on them), then the **template** that consumes it (a production SaaS foundation), then the **agentic** layer (Foundry, Kingdom and the pieces around them). Each entry names a **concept** (what it solves / how it works); the methods on that concept sit beneath it in a dim API row.

```
rule logic ─▶ schema ─▶ lens (scoped view) ─▶ sources ─▶ check / compile
                                  │
                                  ▼
                         builders (headless UI)
                                  │
                                  ▼
                    transitions · permissions  (downstream)
                                  │
                                  ▼
                          the template (consumes all of it)
```

> Canonical source: [`inixiative/.github`](https://github.com/inixiative/.github/blob/main/VOCABULARY.md) · rendered at [www.inixiative.com/vocabulary.html](https://www.inixiative.com/vocabulary.html)

---

## Part I · The shared spine

### 1 · Rule logic: the atom

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

### 2 · Schema: the shape of data

#### FieldMap
**Solves:** describe data (models, fields, kinds, relations, allowed values) without coupling to any ORM.
**How:** `{ models: { Model: { fields: { name: FieldMapEntry } } }, enums? }`. A `FieldMapEntry` has `kind` (`scalar | object | enum | bridge`), `type`, optional `isList`, `fromFields`/`toFields` (FK columns), and `values`: an optional `readonly string[]` that solves "constrained value set" once (enums and picklists route through it; `validateRuleInLens` gates against it regardless of kind, so no enum-promotion hack). Projection output adds `options` (`{ value, label?, groups? }`) on every value-gated or sourced field.

#### Bridge
**Solves:** navigable relations *across* separate field maps (e.g. Prisma + Salesforce + CRM) without merging them.
**How:** a `Bridge` declares two endpoints (`{ fieldMap, model, on }`) and a cardinality (`oneToOne | oneToMany`). Rules read across it by the bridge key (`salesforce:Contact.industry`); `check` walks plain properties, so the caller embeds the foreign row under that key.
<small>API · `stitchFieldMaps`: inject a navigable `kind: 'bridge'` field on each endpoint model · `indexBridges(set, rawData)`: index foreign rows by their `on` key into a `BridgeDictionary`, ready to embed under bridge keys.</small>

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
<small>API · `toPrisma(condition,{map,mapName,model} or {lens})` → a plan `{ steps }` (with `groupBy` steps for count and aggregate operators) that `executePrismaPlan` resolves to a Prisma `where` · `toSql(condition,{map,model,alias} or {lens})` → `{ sql, params, joins }` (array predicates with conditions throw; run them through `toPrisma` or `check`) · `toWhereSql` / `toLensWhereSql`: a condition, or a lens's row filters, as an inline SQL predicate in the engine's `sql.dialect`.</small>

### 7 · Builders: headless UI on the lens (`@inixiative/rules-builder`)

#### Rule builder
**Solves:** build/edit a `Condition` **headlessly**: logic in, descriptors out, you render.
**How:** a hook owns the JSON value and emits a recursive `GroupNode`/`LeafNode`/`ArrayNode` descriptor tree; each node states *what controls exist* (field/operator/value, each `{ value, options, set }`, with per-control `valid`) plus bound actions (`addRule`, `remove`, …). It renders nothing; you wire your own components (two copy-paste reference renderers, plain and shadcn, are provided).
<small>API · `resolve(source,{sourceValues})`: the narrowed lens as a `LensView`, each visit resolved on demand · `useRuleBuilder(...)` → `{ value, root }` descriptor tree · `describeModelFields(lens, mapName, model)` → per-field `BuilderField` (operators per family, `options`/`enumValues`).</small>

#### lensValuePicker
**Solves:** "pick any value reachable through a lens": the shared atom behind a rule's `field` (LHS) and `path` (RHS reference), reused downstream (permissions, email).
**How:** enumerates leaf values across relations as dotted paths, with kind and allowed values. A `Json` column is flagged `acceptsSubPath` (it has no declared sub-fields, but the kernel resolves *and* validates a dotted JSON path in `check`/`toPrisma`/`toSql` + `validateRuleInLens`), so a renderer can offer freeform descent, e.g. `metadata` → `metadata.theme`.
<small>API · `lensValuePicker(lens,{maxDepth})` · `useLensValuePicker` · `LensValueOption.acceptsSubPath` (JSON freeform sub-path).</small>

### 8 · Downstream layers

#### @inixiative/transitions
**Solves:** declarative, serializable state-machine **guards + affordances**: "can this record move from A→B, and who may?"
**How:** a `TransitionMap` (resource → action → `Action`, the OR of its `from`/`to` edges); each side holds a `predicate` `Condition` and an optional `permission` `ActionRule`, and the `to` side an optional `Merge`. `from` is checked against the current record, `to` against the merged one. Permissions go through an injected `Authorize` seam (`createAuthorize` adapts the permissions library). Reuses `Condition`/`check` for the predicate instead of re-solving it.
<small>API · `checkTransition`: guard a proposed move · `available`: list the legal actions for a record · `eligible` (via `toPrisma`): every record currently eligible for an action.</small>

#### @inixiative/permissions
**Solves:** who may do what, decided on your own records: roles, relations and conditions in one engine, with delegation that never outlives whoever granted it, and no second store to sync.
**How:** a schema maps each resource's actions to an `ActionRule`, all plain JSON. A **permix** holds what the actor holds, its **permissions**: role grants plus **entitlements**, each backed by its grantor and asked again at every use. `check` decides on a loaded record. The **core** (about a thousand lines) never imports the **helpers** (delegation, hydration). 1.0, by the end of 2026, adds **reflection**: the same rules compiled to Prisma and SQL, so "which records can I read" and "who can read this" are one query each.
<small>API · `createRebacCheck` · `createPermissions` · `createCanGrantEntitlement`: may this writer hand that entitlement on · `validateActionRule`: check a stored rule before saving · helpers: `delegate` / `capAtRole` / `installPermissions` / `permixOf` / `createHydrator`</small>

#### @inixiative/prisma-map
**Solves:** the schema reflection the lens reasons over: an ORM-agnostic runtime model map.
**How:** extracts models, fields, relations, FK direction, and enums from a generated Prisma client (v6 & v7). The `FieldMap` the rest of the stack is authored against.

#### atlas
**Solves:** navigate the codebase by **concept**, not folders.
**How:** `@atlas` annotations + a concept graph + `MAP.md`; query by `kind` / `partOf` / `uses` / `path`, or a json-rules predicate.

### The one-line version

`Condition` is the atom; `FieldMap` is the shape; a **lens** scopes the shape; **sources** turn data into options; `validateRuleInLens` enforces; `toPrisma`/`toSql` push to the store; **builders** edit it headlessly; **transitions** and **permissions** compose it without re-solving it; permissions 1.0 reflects its rules to the store the same way.

---

## Part II · The template

A production SaaS foundation (`@inixiative/template`) that consumes the spine. Teams building a SaaS product rebuild the same concerns on every product: auth, tenancy, permissions, side effects, background work, caching, real-time, audit. The template solves each *once*, names it, and reuses it. Each section opens with the problem that whole layer addresses; each entry is a **concept**, with its methods in a dim `API ·` row.

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

- **prismaMap (`@inixiative/prisma-map`)**: generated runtime model/field/relation map; powers hooks, factories, lenses.
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
- **contact + ContactType (~22)**: one polymorphic-owner model for phone/email/social handles; per-type registry def; deliverability + `acceptedKinds`.
- **customerRef**: false-polymorphic customer ↔ provider link; each side a User, Organization or Space.
- **webhookSubscription / webhookEvent**: owner-polymorphic outbound subscriptions; per-attempt delivery ledger (RSA-signed).
- **auditLog**: immutable who/what/before/after with actor + tenant + polymorphic subject.
- **cronJob**: DB-defined scheduled jobs (pattern/handler/payload/enabled, retry config).
- **token / authProvider / account / session / verification**: API keys / org auth config / external credentials / BetterAuth's session schema (handshake sessions live in Redis) / verification values.
- **tag / tagCategory / tagAttachment**: owner-scoped labels grouped by category, attached to resources.
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

### Atlas: the code map

*Why: a large codebase is unnavigable by folders alone. Atlas lets you find code by concept/role/dependency and enforces that the map stays true.*

- **atlas**: navigate by concept not folders: `@atlas` annotations + `.atlas/` registry + generated `MAP.md`. <small>CLI · `graph` (reverse indexes) · `query` (by kind/partOf/uses/path/json-rules) · `generate` (write `MAP.md`) · `stamp` (fill derivable axes) · `check`/`coverage` (CI) · `hook read` / `hook grep` (agent hooks)</small>
- **`@atlas` block**: per-file `@kind` (role) / `@partOf` (concepts) / `@uses` (the dependencies it relies on) / `@constructs`.
- **`.atlas/` config**: `config.ts` (stamp/include rules), `kinds.ts` (role vocab), `concepts.ts` (concept registry).

---

## Part III · Agentic

The tools that carry what you've already decided into every agent session, and the hosted hub they connect through. Product lines are in the [org profile](https://github.com/inixiative/.github/blob/main/profile/README.md); this part names the words.

### Foundry and the chorus

*Why: one big prompt loses instructions in the sauce. Many narrow experts, each holding one part of how the codebase works, stay sharp and point the agent at what already exists.*

- **Foundry**: the workspace around your agent: Claude Code, Codex and Muse as workers, ACP apps (Paseo, T3 Code, Zed) as clients. It runs locally on your own subscriptions and wraps each thread with Wardens. The app is source-available (BSL 1.1); its core engine (`@inixiative/foundry-core`) and ACP adapter (`@inixiative/foundry-acp`) are MIT.
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
- **KCP** *(in progress)*: the Kingdom CLI Proxy. Your provider CLIs run through Kingdom with no credentials on your machine; Kingdom runs the command and streams the output back. MIT.

### Sessions and the loop

*Why: corrections only compile into docs if the sessions they happened in are kept, and only matter if they stop recurring.*

- **Archive**: keeps every agent session, local-first, and syncs to the archives you choose. MIT.
- **agent-session**: drives Claude Code, Codex and Muse as persistent streaming sessions behind one interface. MIT.
- **Oracle**: measures whether a correction stuck, and tunes the docs and Wardens. Closed.
