# archmap — Backend Architecture Visualizer & Explainer (Design)

Date: 2026-10-10
Status: Draft, awaiting review
Name: `archmap` is a placeholder; final product name decided later.

## 1. Purpose

A tool that reads a backend codebase and **explains and visualizes** how it works: what the system is made of, how a request flows through it, what data it stores, and how modules depend on each other.

Audiences (all in scope):

- **Onboarding** — a developer new to a codebase understands the backend quickly.
- **Own projects** — visualize/explain backends the author builds (docs, portfolio).
- **Teams** — architecture docs generated from code, shareable.
- **Teaching** — explain backend concepts using real code.

### Success criteria for v1

1. Correctly maps the author's own Node/TS backends (e.g. Pingward, Safeship): routes, DB tables, external services, request flows match reality on manual review.
2. Published free on the VS Code Marketplace.
3. Polished enough to be a portfolio piece.

## 2. Key decisions

| Decision | Choice | Why |
|---|---|---|
| Privacy model | **Local-first**: analysis runs on the user's machine; source never leaves it | Safest; works on private code without repo access |
| First surface | **VS Code extension** | Developers already work there; enables click-to-code |
| Sharing | **Export** self-contained HTML / JSON containing the model only (no source) | Shareable without a backend; hosted links later |
| v1 language | **Node/TypeScript only**; Python next | Author's projects are TS; best tooling; detector interface keeps it pluggable |
| Analysis engine | **TypeScript compiler via ts-morph** + per-framework detectors | Type info resolves real call chains → accurate request flows |
| Explanations | **Deterministic templates** (free, offline) + **optional AI** | Works without keys; AI enriches |
| AI providers | **Pluggable: Claude, OpenAI, Ollama**; bring-your-own key | Ollama enables fully local AI |
| Accounts/backend | None in v1 | Not needed for local-first + export |

## 3. Architecture

pnpm + TypeScript monorepo:

```
packages/
  core-model/        Architecture Model types + JSON schema (no deps)
  analyzer-ts/       ts-morph project loader + detectors  -> Model
  explainer/         template explanations + AI adapter interface/impls
  viewer/            React app rendering the 4 views (webview + standalone export)
  vscode-extension/  commands, webview host, click-to-code, SecretStorage
  cli/               `archmap scan --out model.json` (CI, tests)
fixtures/            small sample backends + expected model snapshots
```

**The Architecture Model is the contract.** Analyzers produce it; viewer and explainer consume it. A future `analyzer-py` emits the same model; nothing downstream changes.

### 3.1 core-model

Versioned JSON validated by a JSON schema generated from these TypeScript types. **Never contains source code** — only names, workspace-relative paths, line numbers, and structural metadata.

```ts
type Confidence = 'certain' | 'inferred' | 'unresolved';
type Location = { file: string /* workspace-relative, '/' separators */; line: number };

interface ArchModel {
  schemaVersion: string;          // semver, see "Versioning" below
  generatedAt: string;            // ISO timestamp
  nodes: Node[];
  edges: Edge[];
  flows: Flow[];
  report: ScanReport;
}

type NodeKind =
  | 'service' | 'library'         // library = workspace package with no detected entrypoint/routes
  | 'route' | 'middleware' | 'handler' | 'function'
  | 'module'                      // folder (granularity per config) or NestJS @Module
  | 'db-table' | 'external-api' | 'queue' | 'cache';

interface Node {
  id: string;                     // see "ID grammar"
  kind: NodeKind;
  name: string;
  serviceId: string | null;       // null for global nodes (external-api, shared db)
  location?: Location;
  meta: NodeMeta;                 // discriminated by kind, below
}

// Kind-specific meta
type RouteMeta = { method: string; fullPath: string; pathConfidence: Confidence; framework: string };
type MiddlewareMeta = { scope: 'global' | 'router' | 'route'; role?: 'auth' | 'validation' | 'rate-limit' | 'cors' | 'logging' | 'error' | 'other' };
type TableMeta = {
  source: 'prisma' | 'drizzle';
  columns: { name: string; type: string; nullable: boolean; isPrimaryKey: boolean; isUnique: boolean; default?: string; isGenerated?: boolean }[];
  enums?: { name: string; values: string[] }[];
  indexes?: { columns: string[]; unique: boolean }[];
};
type ExternalApiMeta = { provider?: string /* 'stripe' */; host?: string; matchedServiceId?: string /* cross-service call */ };
type QueueMeta = { queueName: string; nameConfidence: Confidence };
type CacheMeta = { client: 'ioredis' | 'redis'; connectionLabel: string };  // one node per client instance, not per key

type EdgeKind = 'calls' | 'imports' | 'reads' | 'writes' | 'relates-to' | 'publishes' | 'consumes' | 'handles' | 'uses-middleware';

interface Edge {
  id: string;
  from: string; to: string;
  kind: EdgeKind;
  confidence: Confidence;
  location?: Location;            // call site
  meta?: { fromColumns?: string[]; toColumns?: string[]; cardinality?: '1-1' | '1-n' | 'n-m'; onDelete?: string }; // relates-to
}

// Request flow = a call tree, not a flat list
interface Flow {
  routeId: string;
  rootStepId: string;
  steps: Step[];
}
interface Step {
  id: string;
  parentId: string | null;
  nodeId: string;                 // middleware, handler, function, or sink (db-table/external-api/queue/cache)
  edgeKind: EdgeKind;             // how the parent reached this node
  phase: 'middleware' | 'handler';
  order: number;                  // source order among siblings
  callSite?: Location;
  confidence: Confidence;
  branch?: { kind: 'cond' | 'try' | 'catch' | 'loop'; label?: string };
  truncated?: 'depth' | 'cycle' | 'node_modules';
}

interface ScanReport {
  services: { id: string; role: 'service' | 'library'; durationMs: number }[];
  detectors: { name: string; status: 'ok' | 'skipped' | 'error'; error?: string; counts: Record<string, number> }[];
  unsupportedFrameworks: string[];  // from package.json allowlist, e.g. koa, hapi, trpc, graphql, typeorm, sequelize, mongoose
  limitsHit: { kind: 'depth' | 'budget-time' | 'budget-memory' | 'max-nodes'; detail: string }[];
}
```

**Flow contents.** A flow keeps the **full call tree pruned to subtrees that reach a sink** (db-table, external-api, queue, cache), so the intermediate business functions stay visible. Middleware steps come first, in registration order. Depth counts function hops from the handler (middleware are depth 0).

**ID grammar** (stable across re-scans as long as the symbol isn't renamed or moved):

| Kind | ID |
|---|---|
| service / library | `svc:{relPackageDir}` |
| route | `route:{serviceId}:{METHOD} {fullPath}` |
| handler / function / middleware | `fn:{serviceId}:{relFile}#{symbolPath}` (e.g. `OrdersService.create`) |
| anonymous function | `fn:{serviceId}:{relFile}#anon@{line}:{col}` — explicitly unstable |
| module | `mod:{serviceId}:{relDir}` or `mod:{serviceId}:nest:{ModuleClass}` |
| db-table | `table:{datasourceLabel}:{tableName}` |
| external-api | `ext:{provider}` or `ext:host:{host}` |
| queue | `queue:{queueName}` (dynamic name → `queue:unresolved@{relFile}:{line}`) |
| cache | `cache:{serviceId}:{connectionLabel}` |
| edge | `{kind}:{from}->{to}` (+ `@{line}` when multiple call sites) |

**Versioning.** `schemaVersion` is semver. Minor = additive fields; major = breaking. The viewer ships migrations so newer viewers read older exports; an export with a newer *major* than the viewer shows a "made with a newer version" message instead of rendering.

### 3.2 analyzer-ts

#### Service discovery

- Every `package.json` in the workspace (honoring `exclude` globs; `node_modules` always excluded) is a **package**.
- A package is a **service** only if a framework detector finds an entrypoint or route in it; otherwise it is a **library**.
- Explicit `services` roots in config override discovery.
- Each package uses its `tsconfig.json`, or `config.tsconfig` if set (e.g. `tsconfig.build.json`), or `allowJs` defaults if it has none. tsconfig `paths` and project `references` are honored.
- Workspace packages (`workspace:*`) resolve to their **source**, not `dist/`, through path mapping, so calls into shared libraries are followed.
- **One ts-morph `Project` per workspace**, not one per package, so shared code is loaded once.

#### Detector contract (two phases)

```ts
interface Detector {
  name: string;
  owns: NodeKind[];                                   // e.g. prisma owns 'db-table'
  appliesTo(pkg: PackageInfo): boolean;               // dependency check in package.json
  discover(ctx: AnalysisContext): { nodes: Node[]; edges: Edge[] };   // phase 1
  classifyCall?(call: CallExpression, checker: TypeChecker): SinkMatch | WrapperMatch | null; // phase 2
}
type SinkMatch = { kind: 'sink'; nodeId: string; edgeKind: 'reads' | 'writes' | 'calls' | 'publishes'; confidence: Confidence };
type WrapperMatch = { kind: 'wrapper'; unwrapArg: number };   // e.g. asyncHandler(fn) -> follow arg 0
```

- **Phase 1 (`discover`)**: every detector runs independently and in isolation. Order does not matter.
- **Phase 2 (call-chain resolution)**: the resolver asks every detector's `classifyCall` about each call expression it visits.
- **Merge rules**:
  - Same node `id` from two detectors → `meta` is unioned and the owning detector wins on conflicts.
  - Same edge `id` → the highest confidence is kept.
  - Nodes of a kind are created only by the detector that `owns` it; others reference them by id.

#### v1 detectors

**Frameworks.** Each must compute the **full path** through mount/prefix composition. A non-literal prefix gives `pathConfidence: 'unresolved'`.

- `express`: `app.METHOD`, `router.METHOD`, `router.route()`, nested `app.use('/prefix', router)`, and middleware (`app.use(mw)` global, `router.use` router-scoped, inline route middleware).
- `fastify`: `fastify.METHOD`, `route()`, `register(plugin, { prefix })` composed recursively, and the hooks `onRequest`/`preHandler` as middleware.
- `nestjs`:
  - Routes: `@Controller(prefix)` + method decorators, `setGlobalPrefix`, URI versioning.
  - Middleware: guards, interceptors and pipes, and `consumer.apply().forRoutes()`.
  - Dependency injection: a provider map is built from `@Module` metadata (`providers`, `useClass`, `useFactory`, `useExisting`, `@Inject(TOKEN)`), so injected tokens resolve to concrete classes. Unmapped tokens are `unresolved`.
  - Nest `@Module`s become `module` nodes.
- `nextjs-api`:
  - App router `app/**/route.ts` (exported `GET`/`POST`/…) and pages router `pages/api/**`.
  - Path rules:
    - `(group)` segments are dropped
    - `[id]` becomes `:id`
    - `[...slug]` becomes `:slug*`
    - `[[...slug]]` becomes `:slug*?`
    - `@slot` and `_private` folders are ignored
    - `basePath` from `next.config.*` applies when it is a literal
  - `middleware.ts` is global middleware, with its `matcher` recorded.

**Middleware roles.** `role` is detected from known packages: auth (passport, jsonwebtoken, next-auth, clerk, `@nestjs/passport` guards), validation (zod, joi, class-validator, ValidationPipe), rate limiting, cors, helmet, and logging (morgan, pino-http). Anything else is `other`.

**Data.**

- `prisma`:
  - `schema.prisma` is parsed with `@mrleebo/prisma-ast` into tables, columns, enums, indexes, and relations with cardinality and `onDelete`.
  - Calls to `prisma.<model>.find*|count|aggregate|groupBy` are `reads`. `create*|update*|upsert|delete*` are `writes`. `$queryRaw`/`$executeRaw` produce an `unresolved` edge to the datasource.
- `drizzle`:
  - `pgTable`/`mysqlTable`/`sqliteTable` definitions are read through ts-morph.
  - Foreign keys come from `.references()`. `relations()` adds cardinality.
  - If the two disagree, `.references()` wins and the relation edge is `inferred`.
  - `db.select().from(t)` is a read. `insert`/`update`/`delete` are writes.

**Outbound.**

- `http-clients` (fetch, axios, got, ky) and a data table of known SDKs (v1: Stripe, Resend, Supabase, AWS S3, OpenAI, Anthropic) produce `external-api` nodes.
- A literal URL becomes `ext:host:{host}`, with credentials and query strings stripped. A non-literal URL is `unresolved`.
- **Cross-service matching:** when a literal host or path matches another detected service's route (localhost ports, a docker-compose service name, or `config.serviceHosts`), `matchedServiceId` is set and the system map draws a service→service edge.

**Async.**

- `bullmq`: `new Queue(name)` is a queue node, `queue.add` publishes, and `new Worker(name, fn)` consumes. The worker fn is analyzed as a flow root. Dynamic names are `unresolved`.
- `redis`: one cache node per client instance (`new Redis(...)`/`createClient`), labeled by variable name.

**Structure.**

- `imports` builds the module graph at folder granularity set by `config.moduleDepth` (default: 2 levels under `src/`).
- Cycles are detected at **file level** and shown rolled up to folders. A folder edge is marked cyclic only if a file-level cycle crosses it.

**Unsupported frameworks.** If `package.json` lists koa, hapi, @trpc/server, graphql/apollo-server, typeorm, sequelize, mongoose or knex, the scan report shows "detected, not yet supported".

#### Call-chain resolution

- **Roots:** each route handler (with its middleware chain prepended) and each BullMQ worker fn.
- **Per-function summaries, memoized:** each function is analyzed once into `{ callees in source order (with branch context), sink matches }`. Flows are composed from these summaries, so shared service and repository layers aren't walked again for every route.
- **Resolution:**
  - Calls go through the type checker to the declaration.
  - A call to an interface or abstract method goes to its single implementation if there is exactly one in the workspace (`inferred`). With more than one, the edge is `unresolved` and lists the candidates.
  - NestJS injections go through the provider map.
  - Known wrappers (detector `WrapperMatch`: `asyncHandler`, `catchAsync`, `validate(schema)(fn)`, Nest `@UseGuards`) are unwrapped.
  - Callbacks passed as arguments are followed when they are function literals or identifiers that resolve.
- **Stops:**
  - Depth `config.maxDepth` (default 8) → `truncated: 'depth'`.
  - A cycle (function already on the stack) → `truncated: 'cycle'`.
  - Any declaration in `node_modules` or a `.d.ts` file (unless a detector classifies it as a sink) → `truncated: 'node_modules'`.
  - Dynamic dispatch or computed members → an `unresolved` step.
- **Budget:** ≤ 60 s and ≤ 1.5 GB RSS for a 1,000-file service on a typical laptop. The analyzer runs in a **child process** with `--max-old-space-size` set from config (default 2048). If the budget is exceeded, it stops resolving and returns a partial model with `limitsHit` recorded.

#### Output

- The model is validated against the schema and written to the extension's `workspaceState` storage (`context.storageUri`), not into the repo.
- The CLI writes it to `--out`.
- `.env*` files are never read.

#### Re-scan (v1: not truly incremental)

- On save (debounced 2 s), the **affected service** is re-analyzed in full, reusing the cached ts-morph `Project`, which is refreshed for the changed files. The new result replaces that service's slice of the model.
- Fine-grained incremental patching (file ownership plus reverse-dependency invalidation) is deferred until after v1.

### 3.3 explainer

- **Templates** generate plain-English text from model slices, e.g. "`POST /orders` is handled by `createOrder` in `orders.controller.ts`. It validates input, writes to the `orders` table, and calls Stripe." Available for every node kind and every flow. Deterministic and offline.
- **AI adapter interface**: `explain(request: { modelSlice, snippets, question? }) => AsyncIterable<string>`.
  - Implementations: Claude, OpenAI, Ollama.
  - Keys stored in VS Code SecretStorage; Ollama needs only a base URL.
- **Payload builder**: assembles the model slice and the minimum relevant code snippets; redacts secret-looking strings (high-entropy tokens, known key prefixes). The user sees an exact **preview** and confirms before anything is sent.
- AI features: richer narration of a node/flow, and free-form Q&A about the architecture.

### 3.4 viewer

React app, one build used in two hosts: VS Code webview and the exported standalone HTML.

Views:

1. **System map** — services, DBs, caches, queues, external APIs, and edges between them.
2. **Request flow** — pick a route; sequence-style diagram of handler → services → DB/external/queue.
3. **Data model (ERD)** — tables, columns, relations from Prisma/Drizzle.
4. **Module dependency graph** — folders/modules and import edges; cycles highlighted.

- Graphs: React Flow with ELK auto-layout. Request flow: custom sequence renderer.
- Edge styling by confidence (dashed = unresolved). "Unknowns" panel lists unresolved edges.
- Large graphs (> ~300 nodes) collapse by service/folder; expand on click.
- Side panel: selected node's template explanation + "Ask AI" (extension host only).
- Host bridge abstraction: `openLocation(file, line)`, `askAi(...)` — implemented by the extension; no-ops / hidden in standalone export.

### 3.5 vscode-extension

- Commands: `Scan workspace`, `Open architecture map`, `Export (HTML/JSON)`, `Configure AI provider`.
- Runs analysis in a worker thread with progress notification and cancellation.
- Click on node → opens file at line.
- Re-scan on save (debounced, incremental).
- Export: single self-contained `architecture.html` (viewer + inlined model) or `model.json`; option to strip file paths.

### 3.6 cli

- `archmap scan [dir] --out model.json` — reuses analyzer-ts. Used for CI smoke tests and fixtures; usable by anyone.

## 4. Data flow

1. User runs **Scan workspace**.
2. Extension discovers services and starts the analyzer in a worker.
3. Detectors run per service: frameworks → data → outbound → async → imports.
4. Call-chain resolution builds request flows.
5. Partial models are merged, validated against the schema, written to `.archmap/model.json`.
6. Webview receives the model and renders the views.
7. Node click → extension opens source location.
8. Node select → template explanation; optional AI via preview-and-confirm.
9. Export writes the standalone HTML or JSON.

## 5. Error handling and limits

- **Confidence on every edge**; unresolved shown honestly, never guessed.
- **Detector isolation**: a throwing detector is logged and skipped; other results still render. Scan report shows per-detector status.
- **Big repos**: worker thread, progress, cancel; collapsed graphs above ~300 nodes.
- **Secrets**: `.env*` never read; AI payload redaction; explicit preview before send.
- **AI failures** (bad key, rate limit, Ollama down): inline error; template explanation remains.
- **Unsupported stack**: still produce module graph; report lists unsupported frameworks detected.
- **Schema validation failure** on merge: surface as scan error with details, do not render a corrupt model.

## 6. Testing

- **Detector fixtures** (TDD): small sample apps per detector (Express+Prisma, NestJS, Next.js+Drizzle, Fastify+BullMQ, ...) with expected `model.json` snapshots.
- **Real-repo smoke** (CI): CLI runs on 2–3 pinned public OSS repos; assert completion, node/edge count ranges, and time budget.
- **Viewer**: component tests rendering fixture models; one Playwright test against an exported HTML.
- **Extension**: `@vscode/test-electron` integration test — scan, open map, click-to-code.
- **Explainer**: template snapshot tests; AI adapters tested with mocked HTTP; redaction unit tests.

## 7. Scope

### In v1

- 4 views; Express, Fastify, NestJS, Next.js API; Prisma, Drizzle; fetch/axios/known SDKs; BullMQ, Redis; imports graph
- Template explanations; optional AI (Claude, OpenAI, Ollama)
- HTML/JSON export; CLI; VS Code Marketplace release

### Later (not v1)

- v1.1: **Guided tours / teaching mode** — step-by-step walkthroughs of a request flow
- Python analyzer (FastAPI, Django, Flask, SQLAlchemy), then Go, Java
- Hosted share links, accounts, team spaces
- Architecture diff between commits/PRs
- JetBrains plugin, desktop app, mobile viewer
