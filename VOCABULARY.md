# Vocabulary

The inixiative ecosystem is built **"solve it once"**: each concern has one canonical home, and new features resolve *to* it rather than around it. This page is the glossary of every primitive — first the **ecosystem spine** (the cross-library kernel: rules, lens, sources, builders, and the layers that compose on them), then the **template** that consumes it (a production SaaS foundation). Each entry gives **what it solves** (the problem it exists for) and **how it works** (the mechanism).

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

## Part I · The ecosystem spine

### 1 · Rule logic — the atom

#### Condition
**Solves:** one serializable way to express "does this hold?" — the single predicate language used everywhere a yes/no over data is needed.
**How:** a recursive JSON tree — `all`/`any` (boolean), field operators (`equals`, `in`, `contains`, `between`…), array operators (`any`/`all`/`none`/`atLeast`…), date operators, aggregates, and `if`/`then`/`else`. No code, no `eval` — pure data you can store, ship, and diff.
<small>API · `check(condition, data)` — evaluate against a record → `true` or a failure string; handles what the compilers can't (array predicates, aggregates, windowing) — the "fetch results, then filter" backstop.</small>

#### Operators
**Solves:** a closed, introspectable catalog of what comparisons exist and what input each needs — so a UI renders the right control and every target honors the same set.
**How:** `Operator` · `ArrayOperator` · `DateOperator` enums, each carrying a `ValueShape` (`scalar | range | array | dateWindow | predicate | none`).
<small>API · `getValueShape(op)` — the input shape a control should render.</small>

#### RuleTarget
**Solves:** "where will this rule run?" — so a rule can be restricted to operators all chosen targets support.
**How:** a tag (`check` in-memory · `toPrisma` · `toSql`); operator availability is intersected across the selected targets.

### 2 · Schema — the shape of data

#### FieldMap
**Solves:** describe data (models, fields, kinds, relations, allowed values) without coupling to any ORM.
**How:** `{ models: { Model: { fields: { name: FieldMapEntry } } }, enums? }`. A `FieldMapEntry` has `kind` (`scalar | object | enum | bridge`), `type`, optional `isList`, `fromFields`/`toFields` (FK columns), and `values` — an optional `readonly string[]` that solves "constrained value set" once (enums, picklists, and hydrated sources all route through it; `checkRuleAgainstLens` gates against it regardless of kind, so no enum-promotion hack).

#### Bridge
**Solves:** navigable relations *across* separate field maps (e.g. Prisma + Salesforce + CRM) without merging them.
**How:** a `Bridge` declares two endpoints (`fieldMap:model.on`) and a cardinality.
<small>API · `stitchFieldMaps` — inject a navigable `map:model` field on each side · `buildBridgeDictionary` — materialize the cross-map links.</small>

### 3 · Lens — one schema, many scoped views

#### Lens
**Solves:** one scoped, enforceable view of a schema — authored once, narrowed monotonically down trust boundaries (org → space → subtenant → client), never widened; enforced at author-time *and* execution-time. The authority/visibility spine the rest of the stack composes on.
**How:** three forms of one thing —

| Form | What | Used for |
|---|---|---|
| **Reference** | `Lens` + a serializable `LensNarrowing` chain | author, store, ship |
| **Full** | `resolveVisit` → a `VisitEffect` per node | the computed view, narrowing applied |
| **Public** | `exposedSurface` (per model) · `projectByPath` (per path) | the leak-safe surface a UI/consumer sees |

A `LensNarrowing` node carries `picks`/`omits` (field visibility), `enumPicks`/`enumOmits` (value sets), `where` (a row-filter `Condition`), `sources` (data-backed options), and `relations` (recursive). Composes filter-first, AND-only.
<small>API · `createLens` — anchor at a model in a map · `resolvePolicy`/`resolveVisit` — compute the `VisitEffect` (visible fields, `enumValuesByField`, composed `where`, `sources`) · `exposedSurface(lens,{sourceValues})` / `projectByPath(lens,{sourceValues})` — project the public surface, folding narrowed + fetched `values` onto `field.values` · `validateNarrowing` — reject a chain that widens or names missing fields.</small>

### 4 · Sources — a column's contents as an option set

#### sources
**Solves:** turn the live values of a column into a field's selectable options (a "pseudo-enum"), scoped by the lens — *engine compiles the query, app executes it*.
**How:** `sources: { field: Condition }` on a model in the narrowing — a per-field eligibility `where` that composes like `where` (AND-only). The app runs the compiled DISTINCT query and ships back `SourceValues = { path, mapName, model, field, values }`, which fold onto `field.values` inside the projection (§3). Wire format: `{ lens, sourceValues }` — both serializable.
<small>API · `sourceQueries(lens)` — a `SELECT DISTINCT field WHERE <narrowing ∧ source>` per `(path, field)`, emitted as both Prisma and SQL (SQL degrades gracefully on array predicates).</small>

### 5 · Enforcing & describing rules against a lens

#### Lens enforcement
**Solves:** make a lens actually constrain rules — is this rule expressible and allowed here, what does it touch, and carry the lens's row-filters into it.
**How:** static analysis + injection over a rule against the projected lens.
<small>API · `checkRuleAgainstLens(rule, lens)` — author-time gate (flags hidden fields + out-of-set values) → `{ ok, violations }` · `describeRule(rule, lens)` — classify (maps touched, bridge-crossing, supported targets) · `applyLens(rule, lens)` — AND the narrowing's `where` into the rule (through array conditions too).</small>

### 6 · Compile targets — push the predicate to the store

#### Compile targets
**Solves:** run a `Condition` where the data lives — a prefilter, or the full filter where expressible.
**How:** the same AST, two compilers.
<small>API · `toPrisma(condition,{map,mapName,model})` → a Prisma `where` (+ `groupBy` for count operators) · `toSql(condition,{map,model,alias})` → `{ sql, params, joins }` (degrades gracefully on array-condition predicates — fall back to `check`).</small>

### 7 · Builders — headless UI on the lens (`@inixiative/rules-builder`)

#### Rule builder
**Solves:** build/edit a `Condition` **headlessly** — logic in, descriptors out, you render.
**How:** a hook owns the JSON value and emits a recursive `GroupNode`/`LeafNode` descriptor tree; each node states *what controls exist* (field/operator/value, each `{ value, options, set }`) plus bound actions (`addRule`, `remove`, …). Renders nothing — wire your own components (a copy-paste reference renderer + shadcn drop-in are provided).
<small>API · `resolve(source,{sourceValues})` — assemble the public surface in one call · `useRuleBuilder(...)` → `{ value, root }` descriptor tree · `describeModelFields(lens, map, model)` → per-field `BuilderField` (operators by target, value shape, `enumValues`).</small>

#### lensValuePicker
**Solves:** "pick any value reachable through a lens" — the shared atom behind a rule's `field` (LHS) and `path` (RHS reference), reused downstream (permissions, email).
**How:** enumerates leaf values across relations as dotted paths, with kind and allowed values.
<small>API · `lensValuePicker(lens,{maxDepth})` · `useLensValuePicker`.</small>

### 8 · Downstream layers

#### @inixiative/transitions
**Solves:** declarative, serializable state-machine **guards + affordances** — "can this record move from A→B, and who may?"
**How:** a `TransitionMap` (model → action → `from`/`to`, guard `ActionRule`, optional `Merge`), evaluated through an injected `Authorize` seam. Reuses `Condition`/`check` for the predicate — it doesn't re-solve reasoning.
<small>API · `checkTransition` — guard a proposed move · `available` — list the legal actions for a record · `eligible` (via `toPrisma`) — every record currently eligible for an action.</small>

#### @inixiative/permissions
**Solves:** rebac/abac/rbac in one ORM-agnostic core.
**How:** an `ActionRule` is a `RuleCheck` (a `Condition` over the record = **ABAC**), a `RelationCheck` (walk a relation, then check = **ReBAC**), or a `SelfCheck` (record field == actor id). A permix wrapper + relationship-walking engine, ORM-agnostic via an injected `ResolveModel`.

#### @inixiative/prisma-map
**Solves:** the schema reflection the lens reasons over — an ORM-agnostic runtime model map.
**How:** extracts models, fields, relations, FK direction, and enums from a generated Prisma client (v6 & v7). The `FieldMap` the rest of the stack is authored against.

#### atlas
**Solves:** navigate the codebase by **concept**, not folders.
**How:** `@atlas` annotations + a concept graph + `MAP.md`; query by `kind` / `partOf` / `uses`.

### The one-line version

`Condition` is the atom; `FieldMap` is the shape; a **lens** scopes the shape; **sources** turn data into options; `checkRuleAgainstLens` enforces; `toPrisma`/`toSql` push to the store; **builders** edit it headlessly; **transitions** and **permissions** compose it — never re-solving it.

---

## Part II · The template

A production SaaS foundation (`@inixiative/template`) that consumes the spine. A SaaS backend accretes the same load-bearing concerns over and over — auth, tenancy, permissions, side effects, background work, caching, real-time, audit. The template solves each *once*, names it, and reuses it, so a new feature resolves *to* existing primitives instead of re-inventing them. Each `###` section opens with the problem that whole layer addresses.

### Core platform & build

*Why: a monorepo of four apps + shared packages only stays fast and consistent if builds, environments, and local services are uniform and cached — this layer makes "clone and run" and "change shared code, everything reloads" true.*

- **Four apps** — one platform, four perspectives: `web` (participant), `admin` (operator), `superadmin` (platform ops), `api` (Hono backend).
- **Bun workspaces monorepo** — share code without publishing; `apps/*` + `packages/*` (`db`, `shared`, `ui`, `permissions`, `email`, `sdk`).
- **Turborepo** — task graph with caching + dependency-aware watch; `turbo watch local` runs all apps, `#` filters one.
- **Generation chain** — `db:generate → generate:openapi → generate:sdk → generate:routes`; `.gen.` files are template-owned, git/lint/search-ignored.
- **Path aliases** — `#/` app-internal, `@template/*` cross-package; `../` relative imports forbidden outside barrels.
- **Barrel files (top-level only)** — only package/module `index.ts` re-exports; everything else imports source directly.
- **Environment names** — `local | test | pr | staging | prod`; `isLocal/isTest/...` flags; `.env.<env>` files (local/test) + Infisical (cloud).
- **with-env** — run any command with the right env loaded (`bun run with <env> <app> <cmd>`).
- **PR environments** — ephemeral per-PR stack (DB branch, previews, isolated secrets), destroyed on close.
- **Docker Compose dev services** — local Postgres/Redis/MinIO; pinned project name so worktrees share one stack.
- **`bun run init` vs `setup`** — `init` = one-time provisioning (Ink TUI: secrets/services); `setup` = per-dev bring-up (deps/Docker/Prisma/seed).
- **Worktree tooling** — isolated parallel checkouts sharing one container stack, per-slot DB names + bucket prefixes.

### API & routing

*Why: every endpoint should look the same, validate itself, and document itself — so you write business logic, not boilerplate, and the frontend gets a typed client for free.*

- **File-based route auto-registration** — drop route+controller in a module, `autoRegisterRoutes()` wires them on startup.
- **Route templates** — `createRoute/readRoute/updateRoute/deleteRoute/actionRoute` emit consistent OpenAPI 3.1 + cut ~70% of boilerplate.
- **makeController** — type-safe handler bound to a route; only the route's declared statuses are respondable (wrong-shape responses caught at compile time).
- **respond (ok/created/noContent)** — standard status + `{ data }` envelope.
- **`{ data }` envelope** — uniform response shape across single/list endpoints, so the frontend handles every response the same way.
- **makeError / AppError** — throwable `HTTPException` with `{ error, message, guidance, fieldErrors?, requestId }`.
- **HTTP_ERROR_MAP** — drift-free default labels/guidance per status code.
- **errorHandlerMiddleware** — global `onError` normalizing Zod/Prisma/HTTPException/raw errors, stamps `requestId`.
- **paginate()** — list-endpoint orchestrator: applies search + filter + ordering + pagination from the request; returns `{ data, pagination }`.
- **Bracket-notation query language (`bracketQuery`)** — one URL syntax for **filtering and ordering**: `filter[user.email][gte]=…` and `?orderBy[]=field:dir` (GraphQL-like, URL-based, cacheable). `parseBracketNotation` parses the request into a `BracketQueryRecord` (the `bracketQuery` context var); `buildWhereClause` compiles filters into a Prisma `where` (merged with server scope, lens-validated) and `buildOrderBy` the sort (path-notation guarded against injection); the `[:]` marker carries real `null`/`true`/`false` through all-string params (allowlisted, no eval); `dialect` handles provider-specific JSON-path filtering (Postgres array vs MySQL JSONPath); symbols/operators live in `@template/shared/bracketQuery`. FE mirror: `buildFilterQuery`/`serializeBracketQuery`.
- **Batch API** — many requests in one call; strategies `transactionAll/transactionPerRound/allowFailures/failOnRound`; `<<round.req.path>>` interpolation.
- **OpenAPI 3.1 generation** — full 3.1.0 spec served as JSON at `/openapi/docs` (via `app.doc31`); drives the SDK.
- **Generated SDK (`@template/sdk`)** — hey-api typed client + TanStack Query hooks + query keys from the OpenAPI spec.
- **Module structure** — allowlisted folders per `modules/<name>/`: routes/controllers/services/schemas/queries/tests/...; shared code → `lib/`.
- **Router exports** — `<resource>Router` / `admin<Resource>Router`, mounted in `index.ts`.
- **Modules constant** — type-safe model-name registry; `Modules.organization` not `'organizations'`.

### Request context & errors

*Why: multi-tenant code must never leak across requests — so identity, tenant, db client, and permissions are isolated per request and flow implicitly, not threaded by hand.*

- **prepareRequest** — first middleware; sets db/requestId/permix, nulls auth, wraps in `db.scope`.
- **requestContext (AppEnv)** — typed per-request Hono Variables (db, user, token, permix, resource, filterLens, …).
- **getActor** — resolve the effective actor uniformly across session and org/space tokens.
- **getResource / resourceContextMiddleware** — auto-load the `:id` resource (404 on 0, 409 on >1) before authorization.
- **scopeNarrowing** — merge per-request tenant/ownership `where` scope into the filter lens.
- **Database scope (AsyncLocalStorage)** — request/job correlation id; `getScopeId`/`isInTxn`.
- **getValidatedBody / getValidatedQuery** — typed validated input outside route-typed context.

### Adapters

*Why: external services (storage, email, error reporting) change per environment and shouldn't leak SDK details into app code — one interface, swappable providers, env-selected.*

- **Adapter pattern** — couple to swappable external SDKs via a role interface + providers + env wiring.
- **makeAdapterRouter** — pick one provider per environment (prod/staging/pr/default).
- **makeAdapterRegistry** — pick a provider explicitly by name.
- **makeBroadcastRegistry** — fan one operation out to many providers via `Promise.allSettled`.
- **storage adapter (s3)** — provider-agnostic blob storage (presigned POSTs); MinIO locally.
- **errorReporter adapter** — console locally, Sentry in prod, behind the router.
- **email client adapter** — `EmailClient` with Resend (prod) / Console (dev) providers, per-tenant keys.

### Database & ORM

*Why: the DB is where correctness, type-safety, and consistency are won or lost — these primitives make Prisma transactional-by-default, runtime-introspectable, and impossible to misuse.*

- **prismaMap (`@inixiative/prisma-map`)** — generated runtime model/field/relation map; powers hooks, factories, lenses.
- **Typed model IDs** — phantom-branded `UserId`/`OrganizationId` prevent cross-model ID mixups at compile time.
- **Model name utilities** — `ModelName`↔`AccessorName` conversion + `isModelName`/`isAccessorName` guards.
- **UUIDv7 IDs** — time-sortable PKs without coordination; better index locality than v4.
- **db client (txn-aware proxy)** — routes to the active transaction when present, else raw; `db.raw.*` bypasses.
- **db.scope / db.txn / db.onCommit** — scoped state; auto-merging transactions; after-commit side effects.
- **db.findForUpdate** — row-level pessimistic lock (transaction-only).
- **assertNoNestedWrites** — reject nested writes that would skip lifecycle hooks (validation/audit).
- **Disabled createMany/updateMany** — they return no rows (break hooks); use `*AndReturn`.
- **`query` / pass-the-delegate** — pass a concrete delegate (`query.findMany(db.user, …)`) so TS infers exact types; `RuntimeDelegate`/`AnyCrudDelegate` cover runtime-only model access.
- **hydrate** — batch-resolve a record's FK relations (no N+1) from `prismaMap`.
- **Lens helpers (lensFor/includeFromLens/prune/redactLens/fetchLens)** — json-rules lens over the schema → Prisma include/projection/redaction.
- **Path notation (buildNestedPath/validatePathNotation)** — safe dot-notation nested queries (charset/depth guarded).
- **Seed (SeedFile / `--prime`)** — FK-ordered idempotent upserts by UUIDv7; `prime` records are dev-only.
- **Split schema** — one `.prisma` file per model, combined at generation (fewer merge conflicts).
- **Zod schema generation** — `<Model>ScalarSchema`/input schemas derived from Prisma.

### Mutation lifecycle & hooks

*Why: cross-cutting concerns (audit, validation, cache, webhooks, ordering) must fire on every mutation, inside its transaction, without sprinkling calls everywhere — one extension runs them declaratively.*

- **mutationLifeCycle extension** — Prisma client extension wrapping create/update/delete/upsert with auto-txn + hooks + previous-state fetch.
- **Mutation lifecycle hooks** — `{model, action, args, result, previous}` callbacks at `before`/`after` timing.
- **registerDbHook / hook registry** — central registration + dispatcher run by the lifecycle extension; `registerHooks()` at boot.
- **auditLog hook** — immutable before/after/changes diff + actor context for `AUDIT_ENABLED_MODELS`.
- **immutableFields hook** — strip `id`/`createdAt`/FKs (and overrides) on update; silently, never throws.
- **rules hook + RulesRegistry** — declarative per-model field validation via json-rules `Condition`.
- **shadowMerge** — merge `previous` with update data so atomic ops (`increment`/`push`) validate against final state.
- **cache hook + `cacheReference`** — invalidate cache keys a mutated record is reachable by, post-commit (per-model key map in `hooks/cache/constants/cacheReference.ts`).
- **webhooks hook + circuit breaker** — enqueue signed `sendWebhook` post-commit; auto-disable a subscription after 5 fails.
- **orderedList hook** — dense `[1..N]` position columns per scope, bulk-safe via SQL CTEs; soft-deletes get negative sentinels.
- **contactRules hook + ContactRegistry** — per-`ContactType` parse/canonicalize/validate/uniqueness.
- **False polymorphism + PolymorphismRegistry** — type enum + separate FK columns (not a real polymorphic FK); one source of truth for constraints/rules/immutability.
- **preventHardDelete / soft delete** — `deletedAt`; `SOFT_DELETE_MODELS`; `null→timestamp` recorded as a delete.
- **HOOK_IGNORE_FIELDS / isNoOpUpdate** — skip audit/cache/webhooks when only noise fields changed.
- **auditActorContext** — AsyncLocalStorage actor (user/token/spoof/IP/inquiry) auto-attached to mutations/events.
- **Slow mutation logging** — warn when a mutation exceeds 5s.

### Auth & sessions

*Why: support every auth shape (cookie session, OAuth bearer, API token, impersonation) behind one resolved actor — so handlers never care how the caller authenticated.*

- **BetterAuth** — session-based browser auth (signup/signin/OAuth) issuing a stateless JWT.
- **Stateless JWT session** — JWT in HTTP-only cookie (email/pw) or localStorage Bearer (OAuth); `cookieCache` skips DB validation for a window.
- **secondaryStorage (Redis)** — fast permission/org caches; not the primary session store.
- **AuthProvider (per-org OAuth/SAML)** — org-scoped (`organizationId` required) OAuth/SAML config with encrypted secrets; platform-wide providers are env-driven via `getPlatformProviders()`, not `AuthProvider` rows.
- **Token authentication** — `Authorization: Bearer`/`?token=`, SHA-256 hashed, Redis-cached; `keyPrefix` identifies without exposing.
- **Token owner / hierarchical scope** — polymorphic owner (User/Org/OrgUser/Space/SpaceUser); scope determines reach.
- **Auth middleware chain** — `prepareRequest → tokenAuth → authMiddleware → spoof`; token wins over cookie.
- **setUserContext / refreshUserContext** — populate user/org/space + permissions; rebuild mid-request after changes.
- **Spoofing (impersonation)** — superadmin `x-spoof-user-email`; records `spoofedBy`; response headers + UI badge for auditability.
- **validateUser / validateActor / validateNotToken / validateSuperadmin** — auth-method/role gates.
- **basicAuthMiddleware** — HTTP Basic gate for protected utilities (e.g. BullBoard).

### Authorization (RBAC / ABAC / ReBAC)

*Why: real permissions are roles AND attributes AND relationships at once — one engine expresses all three declaratively, so access rules are data, not scattered `if` checks.*

- **Permix** — per-request permission checker (`createPermissions()`), `check/setup/setSuperadmin`.
- **PlatformRole / superadmin bypass** — `user|superadmin`; superadmin short-circuits `check()` to true (with audit).
- **Role (RBAC) + roleHierarchy** — `owner>admin>member>viewer`; `lesserRole`/`greaterRole`, `rolesAtOrAbove`.
- **StandardAction** — `read|operate|manage|own`; `roleToStandardAction` maps role→action.
- **Entitlements (ABAC)** — per-membership/token boolean grants beyond role; `intersectEntitlements` constrains tokens.
- **ReBAC engine** — schema-driven action resolution walking relations/self/conditions, with cycle detection.
- **rebacSchema / ModelPermission** — per-model actions → `ActionRule`s.
- **ActionRule** — algebra: string-inherit | `RelationCheck` | `SelfCheck` | `RuleCheck` | `any`/`all` | null.
- **RelationCheck `{rel, action}`** — delegate to a related resource's action (single-hop or dot-path multi-hop).
- **SelfCheck `{self: field}`** — grant when actor id == `record[field]`.
- **RuleCheck `{rule}`** — json-rules `Condition` over the record (ABAC predicate).
- **ResolveModel** — app seam mapping a relation field → model (via `prismaMap`); injected into the engine.
- **ownerActions()** — spreadable action block fanning out across owner relations for owner-polymorphic models.
- **Row-level overrides (`permissionRules`)** — per-row additive grants OR'd with the schema rule (sharing without schema change).
- **Token permission restriction** — token can't exceed owner (`lesserRole` + entitlement intersection).

### Encryption

*Why: sensitive fields must be encrypted at rest with rotation and tamper-detection, but adding a field shouldn't require touching hooks/rotation/validation — a registry makes it one-line, auto-discovered.*

- **AES-256-GCM field encryption** — authenticated per-field encryption, random IV, auth tag.
- **createEncryption / EncryptedFieldData** — `createEncryption(keyring)` → `{ encrypt, decrypt }`; `EncryptedFieldData` = `{ciphertext, version, iv, authTag}` (base64).
- **AAD (buildAAD)** — bind ciphertext to immutable record fields; prevents transplant to another record.
- **ENCRYPTED_MODELS registry** — single source of which fields encrypt; hooks/rotation/validation auto-discover.
- **Column-triplet convention** — `encrypted{Field}` + `…Metadata` + `…KeyVersion`; `encryptField`/`decryptField` generics.
- **EncryptionKeyring / key versions** — current + previous key for dual-key zero-downtime rotation.
- **rotateEncryptionKeys (singleton job)** — idempotent re-encrypt via `updateManyAndReturn where version=N`.
- **`validateEncryptionVersions`** — rejects key-version gaps/downgrades/mixed versions (the deploy-time guard; currently exercised in tests).

### App events & jobs

*Why: business logic must never call email/SMS/analytics directly — it emits an event; bridges and background jobs do the side effects reliably, after commit, off the request path.*

- **emitAppEvent** — fire a business event decoupled from side effects; auto-enriches actor.
- **makeAppEvent** — define one event's fan-out across bridges (email/websocket/observe/cb) via `Promise.allSettled`.
- **appEventHandlers** — centralized typed name→handler map; `onCommit` deferral inside txns.
- **bridges** — email (`EmailHandoff` → sendEmail job), websocket (`WSEvent` refetch triggers), observe (→ `recordAppEvent`), raw callbacks.
- **enqueueJob** — typed central job dispatch with overflow protection + dedupe.
- **makeJob / makeSingletonJob / makeSupersedingJob** — basic / one-at-a-time (lock) / newest-cancels-older (lanes).
- **jobHandlers / ctx.log** — typed name→payload→handler registry; dual-write logging (stdout + BullBoard).
- **Worker / BullMQ** — concurrency 10, retries, graceful shutdown; own Redis connection.
- **BullBoard** — queue/job monitoring UI; receives worker logs via broadcast.
- **Job overflow buffer (JobOutbox)** — adhoc jobs spill to DB rows past `maxQueueDepth`; `accumulator` batches the writes.
- **drain loop** — re-admit buffered jobs FIFO as room frees, under a lock; `quarantine` poison rows after N attempts.
- **overflow flag / queueDepth** — Redis set-once trip flag; cached `waiting+active` probe.
- **Cron jobs / JobType** — DB-defined schedules registered at startup; `cron|cronTrigger|adhoc`.
- **messaging (messageUser/messageContact + messageProviderRegistry)** — non-email multi-channel dispatch via pluggable per-`ContactType` adapters.

### Concurrency

*Why: a horizontally-scaled app needs cross-process mutual exclusion, supersession, serialization, and bounded parallelism — small primitives so batches don't exhaust pools or interleave.*

- **createLock** — distributed Redis lock with heartbeat/TTL/`onLockLost`; single-node mutual exclusion. In `@template/db`.
- **lanes (claimLane/watchLane)** — cross-process "last enqueue wins" baton for superseding jobs; own `lane:` Redis namespace. In `@template/db`.
- **createSerializedQueue** — serialize concurrent async writes to one resource (in-process promise chain).
- **getConcurrency / resolveAll** — parallelism caps per resource class; bounded `Promise.all`.
- **heartbeat** — non-overlapping recurring async timer with clean `stop()`.

### Redis & caching

*Why: caching is only safe if invalidation is automatic and keys never collide — a namespaced key scheme plus mutation-driven invalidation makes reads fast without staleness bugs.*

- **Redis connection strategy** — main shared client + dedicated subscriber + BullMQ's own; `createRedisConnection`.
- **redisNamespace** — central key-prefix map (bull/cache/job/ws/otp/session/limit/lock/lane).
- **cacheKey / validateKey** — canonical `cache:{accessor}:{field}:{value}[:tags][:*]`; throws on `undefined`.
- **cache (get-or-set)** — memoize reads with TTL + negative-caching (stampede protection).
- **clearKey + automatic invalidation** — exact/wildcard delete; reverse-reference map clears the right keys on mutation.
- **RedisMock / flushRedis** — in-memory Redis for deterministic tests.

### Realtime / websockets

*Why: clients need instant "your data changed" signals across load-balanced servers, without leaking data or hand-managing subscriptions — refetch triggers over Redis pub/sub do it.*

- **WebSocket server** — native Bun WS on the Hono port; anonymous at handshake, then token auth via an in-band `authenticate` message.
- **Connection lifecycle / registry** — track by id/user/channel; identity via `authenticate`/`spoof`/`logout`.
- **channelKey / WSEvent** — query identity key; data-free `{category:'query', action:'refetch', key}` contract.
- **sendToChannel / sendToUser / broadcast** — publish to `ws:broadcast` Redis channel → fan out to all instances.
- **initWebSocketPubSub** — each instance subscribes to `ws:broadcast` and routes to local delivery.
- **LIVE_QUERIES** — operationIds with a BE producer; the FE subscribes only to these.
- **createApiWebsocket / client pipe** — channel refcount, reconnect, send-queue; queryCache subscriber → auto subscribe/invalidate.
- **WS heartbeat / stale sweep** — ping/pong liveness; close idle connections; FE force-reconnect.

### Communications / email

*Why: transactional email must be tenant-overridable, idempotent, consent-aware, and reconstructable — a DB-driven template system + a planner/deliver pipeline gives at-most-once sends you can audit.*

- **EmailTemplate / EmailComponent** — tenant-overridable MJML templates + reusable blocks (`{{component:slug}}`).
- **Render pipeline (compose/expand/interpolate)** — fetch + recursively expand components + substitute `{{sender/recipient/data.*}}`.
- **evaluateConditions** — `{{#if rule=…}}` blocks driven by json-rules; a throwing rule drops the branch.
- **Cascade resolution (lookupCascade)** — owner chain Space→Org→default (and user chain) picks the right-tier template.
- **EmailOwnerModel / inheritToSpaces** — false-polymorphic ownership: default/admin/Org/Space/User/...
- **EmailErrorPolicy** — `fail` (retry→DLQ) / `degrade` (drop bad blocks) / `fallback` (re-render one owner up).
- **sendEmail planner → deliverEmail** — two-stage fan-out: planner finds-or-creates one log per recipient, then one deliver job each.
- **idempotencyKey** — event-anchored hash-last key collapses duplicate sends but re-sends on payload change.
- **CommunicationLog** — per-recipient delivery ledger + at-most-once fence; re-renderable without storing the body.
- **canDeliver (acceptedKinds)** — per-`Contact`/`CustomerRef` consent by `CommunicationKind`.
- **Unsubscribe (HMAC link)** — signed one-click unsubscribe (RFC 8058 headers) for non-user recipients.
- **Deliverability cache** — bouncer verdict cached on `Contact` (TTL); skip undeliverable pre-send.
- **Email versioning / recompose** — audit-log version graph reconstructs exactly what was sent.
- **Email clients (Resend/Console)** — pluggable `EmailClient` per environment.

### Frontend architecture & state

*Why: three apps share one state model and one API contract — a single composable store + generated SDK keep client state, tenancy, and server data coherent without per-app forks.*

- **useAppStore** — one shared Zustand store composing 6 slices; consumed via `@template/ui/store` (all three apps).
- **Slice composition (createXSlice)** — modular `StateCreator` slices spread into one store; `AppStore` = their intersection.
- **AuthSlice / TenantSlice / PermissionsSlice / NavigationSlice / ClientSlice / UISlice** — identity / tenant context / FE ReBAC / nav config / QueryClient+WS / theme.
- **user-context slices (superadmin)** — same store, user-context-first, no forced tenant switching.
- **File-based routing (TanStack Router)** — `createFileRoute` under `app/routes/`; nesting via `<Outlet/>`.
- **`__root` / layout routes** — `_authenticated` (guard+AppShell), `_public`, `_fullscreen`.
- **Guards (requireAuth/requirePublic)** — redirect in `beforeLoad`, preserve `redirectTo`/context.
- **useAuthenticatedRouting** — sync tenant context + spoof between URL and store; route permission check.
- **Navigation config (`NavConfig`/`NavItem`)** — declarative, permission-filtered nav: context-keyed (`user`/`organization`/`space`/`public`) recursive `NavItem` trees.
- **apiQuery / apiMutation** — typed SDK wrappers (unwrap `data.data` / keep full response); auth+spoof headers.
- **Theme system (three-tier CSS vars)** — `--app-*` → `--space-*` → `--primary`; `useDarkMode`/`useSpaceTheme`.

### Frontend layout & UI

*Why: a consistent app frame and a set of conditional-aware primitives let every page look and behave the same with minimal code — permission-filtered, theme-aware, responsive by default.*

- **AppShell** — master authenticated layout: Sidebar (ContextSelector + nav + UserMenu) beside `<Outlet/>`.
- **ContextSelector** — switch Personal/Org/Space (supports lockedContext for white-label).
- **Sidebar / Header / UserMenu / Breadcrumbs** — permission-filtered nav / page header / account menu / location trail.
- **MasterDetailLayout / DetailPanel / ResponsiveDrawer** — list+detail; slide-out detail; drawer-desktop/modal-mobile.
- **Modal / FullscreenLayout / Toaster / ErrorBoundary** — overlay dialog / chrome-less protected page / toasts / render-error fallback.
- **UI primitives** — Button (CVA variants), Input, PasswordInput, SlugInput, Select, Card, Badge, Table, Avatar, EmptyState, DropdownMenu, Pagination, ThemeToggle, Label.
- **Conditional props (`show`/`disabled`/`disabledText`)** — uniform conditional render/disable across components.
- **SettingsLayout + context-aware tabs** — nested-route settings sidebar adapting to personal/org/space.

### Frontend data & hooks

*Why: list UIs all need the same search/filter/sort/pagination/scroll behavior, derived from the API's own schema — so tables and filters stay in sync with the backend automatically.*

- **usePaginatedData / useInfiniteData** — page-number / infinite-scroll controllers with URL + scroll persistence.
- **makeDataConfig / useDataFilters / useQueryMetadata** — derive searchable/orderable/enum fields from the OpenAPI spec; shared filter state.
- **buildFilterQuery / serializeBracketQuery** — serialize filter/sort state to bracket-notation query params.
- **useInfiniteScrollTrigger / useScrollState / useSectionHash** — IntersectionObserver load-more; scroll restore; hash↔section deep-linking.
- **useOptimisticMutation + createOptimisticListTarget** — optimistic cache updates with rollback; `createOptimisticListTarget` adapts it for list create/update/delete.
- **usePermission / useInquiryPermission** — turn a permission check into `{show, disable, disabledText}`.
- **useAuthProviders / useValidateUniqueness / useDebounce / useMediaQuery** — provider fetch / live availability / debounce / breakpoint.
- **usePageMeta / useBreadcrumbs / useLanguage** — document title+meta / crumb trail / locale.
- **makeContextQueries** — per-feature, context-scoped (public/user/org/space) query+mutation registries.

### Features (domain)

*Why: the recurring SaaS nouns (tenants, members, approvals, contacts, webhooks, audit) ship as first-class, consistent primitives instead of being re-modeled per product.*

- **organization / space** — top-level tenant; org-scoped sub-workspace with branding.
- **organizationUser / spaceUser** — membership joins with `Role` + entitlements; space membership requires org membership.
- **users (user) / me** — platform identity; self-service `/me` surface (profile, contacts, tokens, webhooks, redact).
- **redact (GDPR)** — anonymize user PII while preserving referential integrity.
- **inquiry** — generic request/approval/audit primitive; polymorphic source/target, per-type handler, status state machine.
- **InquiryType / handler** — `inviteOrganizationUser/createSpace/updateSpace/transferSpace`; handler = content+resolution schemas + `handleApprove`/`autoApprove`/`unique`/expiry.
- **contact + ContactType (~22)** — one polymorphic-owner model for phone/email/social handles; per-type registry def; deliverability + `acceptedKinds`.
- **customerRef** — false-polymorphic customer (User/Org/Space) ↔ provider (Space) link.
- **webhookSubscription / webhookEvent** — owner-polymorphic outbound subscriptions; per-attempt delivery ledger (RSA-signed).
- **auditLog** — immutable who/what/before/after with actor + tenant + polymorphic subject.
- **cronJob** — DB-defined scheduled jobs (pattern/handler/payload/enabled, retry config).
- **token / authProvider / account / session / verification** — API keys / org auth config / external credentials / sessions / verification values.
- **tag / tagCategory / tagAttachment** — owner-scoped labels grouped by category, attached to resources.
- **batch** — multi-request execution with strategies, interpolation, and `BatchStatus`.
- **entitlements / invitations** — feature grants beyond role; invitation = the `inviteOrganizationUser` inquiry.
- **admin / superadmin surfaces** — privileged cross-tenant listing/management; `superadmin` app + permission bypass.

### Testing & quality

*Why: confidence comes from data factories that mirror reality and gates that block drift — so tests are fast to write, isolated, and the repo's bespoke conventions are enforced mechanically.*

- **Test factories (`build*`/`create*`)** — in-memory (`build*`) vs persisted (`create*`) records; auto-infer FK deps; `createFactory(model, config)`; `getNextSeq()` for unique values (`packages/db/src/test/factories`).
- **`__serialize()`** — convert `Date`→ISO to assert against API/SDK string-timestamp shapes (CI-enforced in UI tests).
- **createTestApp** — Hono app with mocked auth/tenancy; auto superadmin bypass.
- **request helpers (get/post/.../json)** — concise HTTP calls + `{data, pagination}` parsing.
- **cleanupTouchedTables / `--max-concurrency=1`** — DB test isolation; serial test runs.
- **happy-dom** — browser-less DOM testing via `bunfig.toml` preload.
- **CI rule runner (`run-ci-rules.sh`)** — repo-specific rules beyond Biome; `--test` self-tests against pass/fail fixtures.
- **CI rules** — `no-jest`, `no-vitest`, `no-radix`, `no-lens-spread`, `no-module-mocks`, `ui-serialized-factories`, `spy-must-restore`, … (protected paths).
- **`bun run check`** — canonical gate: lint + typecheck + backend tests + FE tests + CI rules.
- **post-Biome checks** — import-alias (`check-import-aliases.sh`) + stale-generated-file checks.

### Observability & ops

*Why: in production you need to trace one request/job across every log line and catch errors without coupling app code to a vendor — scoped structured logging + optional otel/sentry.*

- **logger** — structured logging over a swappable adapter (pino deployed / consola local); server-only.
- **LogScope / logScope() / scope ids** — nesting context tags; request/job correlation (`requestId`).
- **addLogBroadcast** — pipe scoped logs to extra sinks (e.g. BullBoard).
- **frontendLogger** — browser-safe logger, never crosses with the server logger.
- **OpenTelemetry / Sentry** — optional tracing + error tracking, env-gated, skipped local/test.

### Atlas — the code map

*Why: a large codebase is unnavigable by folders alone — atlas lets you find code by concept/role/dependency and enforces that the map stays true.*

- **atlas** — navigate by concept not folders: `@atlas` annotations + `.atlas/` registry + generated `MAP.md`.
- **`@atlas` block** — per-file `@kind` (role) / `@partOf` (concepts) / `@uses` (load-bearing deps) / `@constructs`.
- **`.atlas/` config** — `config.ts` (stamp/include rules), `kinds.ts` (role vocab), `concepts.ts` (concept registry).
- **atlas commands** — `graph` (reverse indexes), `query` (by kind/partOf/json-rules), `stamp` (fill derivable axes), `check`/`coverage` (CI).
