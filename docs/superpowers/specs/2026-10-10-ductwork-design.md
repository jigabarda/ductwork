# Ductwork — Backend Architecture Visualizer & Explainer (Design)

Date: 2026-10-10
Status: Revised after review and re-ordered for startup validation (2026-10-11), awaiting user approval
Name: **Ductwork** (npm `ductwork` available; repo github.com/jigabarda/ductwork). VS Code Marketplace and domain availability to verify in M6.

## 1. Purpose

A tool that reads a backend codebase and **explains and visualizes** how it works: what the system is made of, how a request flows through it, what data it stores, and how modules depend on each other.

Audiences (all in scope):

- **Onboarding** — a developer new to a codebase understands the backend quickly.
- **Own projects** — visualize/explain backends the author builds (docs, portfolio).
- **Teams** — architecture docs generated from code, shareable.
- **Teaching** — explain backend concepts using real code.

### Success criteria

1. **Accuracy:** correctly maps the author's own backends, Pingward and Safeship (both Next.js + Prisma). This is measured by the golden-file targets in section 6.
2. **Demand validated (M2 gate):** see section 8 for the metrics. If the gate fails, we pivot or stop before building M3+.
3. **Revenue path:** at least 3 design-partner teams use PR architecture checks (M4) on a private repo.
4. **Distribution:** free CLI, MCP server and VS Code extension published (npm, Marketplace).

## 2. Key decisions

| Decision | Choice | Why |
|---|---|---|
| Privacy model | **Local-first**: analysis runs on the user's machine; source never leaves it | Safest; works on private code without repo access |
| First surface | **CLI + standalone HTML export** (validation build), then MCP server, PR checks, and the **VS Code extension** | The CLI is the fastest way to put maps of real repos in front of people; the extension is the daily-use surface |
| Business model | **Open core**: free MIT CLI, extension and MCP server; paid PR architecture checks for private repos and teams | The free tools bring developers in; teams pay for checks in their workflow, not for diagrams (see section 8) |
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
  cli/               `ductwork scan | diff | check | mcp`
  diff/              model-to-model diff + architecture rules engine (PR checks)
  mcp-server/        MCP server exposing the model to AI coding agents
  github-action/     runs `ductwork check` on PRs, posts a summary comment
fixtures/            small sample backends + expected model snapshots
scripts/golden.ts    accuracy script; golden expectations live OUTSIDE this public repo
                     (path passed via DUCTWORK_GOLDEN_DIR)
```

**The Architecture Model is the contract.** Analyzers produce it; the viewer, explainer, diff engine and MCP server consume it. A future `analyzer-py` emits the same model; nothing downstream changes.

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

- The model is validated against the schema and written to the extension's per-workspace storage folder (`context.storageUri`), not into the repo.
- The CLI writes it to `--out`.
- `.env*` files are never read. The redactor (see 3.3) runs on every literal string stored in the model (external hosts, queue names, meta).

#### Re-scan (v1: not truly incremental)

- On save (debounced 2 s), the **affected service** is re-analyzed in full, reusing the cached ts-morph `Project`, which is refreshed for the changed files. The new result replaces that service's slice of the model.
- Fine-grained incremental patching (file ownership plus reverse-dependency invalidation) is deferred until after v1.

### 3.3 explainer

**Templates**

- Templates turn model slices into plain-English text and **only state facts the model contains**.
- Example: "`POST /api/orders` passes through `requireAuth` (auth) and `validateBody` (validation), is handled by `OrdersController.create` in `orders.controller.ts`, writes to the `orders` table, and calls Stripe."
- Every node kind and every flow has a template. Templates are deterministic and work offline.
- **Teaching support in v1:** each node kind and middleware role links to a short built-in glossary entry ("What is middleware?", "What is a foreign key?", "Why use a queue?"). Full guided tours come in v1.1.

**AI adapters**

- Interface: `explain(request: { modelSlice, snippets, question? }) => AsyncIterable<string>`.
- Implementations: Claude, OpenAI, Ollama. The provider and model are chosen in settings.
- Keys are stored in VS Code SecretStorage.
- Ollama's base URL defaults to `http://localhost:11434`. A non-loopback host shows a warning that data will leave the machine.

**Payload builder**

- **Model slice:**
  - For a node or flow question: the node or flow plus neighbors one hop away.
  - For free-form Q&A: the service summary (routes, tables, externals) plus nodes whose names match the question's keywords.
  - The slice is capped at `config.ai.maxContextTokens` (default 12k).
- **Snippets (optional, on by default; `config.ai.includeSnippets`):**
  - For each function in the slice, the function body, capped at 80 lines each and 400 lines total.
  - Turning snippets off gives model-only AI.
- **Redaction (best-effort, stated in the UI):**
  - Strings that look like secrets are replaced with `[REDACTED]`: high-entropy tokens, known key prefixes (`sk-`, `pk_live_`, `AKIA`, `ghp_`, `xox`), JWTs, connection strings and URL credentials.
  - The same redactor runs on the **model** during analysis (see 3.2).
- **Preview:** shows the exact payload, the provider and the host, and an approximate token count. Nothing is sent until the user confirms.
- **Features:** richer narration of a node or flow, and free-form Q&A about the architecture.
- **Workspace Trust:** AI is disabled in untrusted workspaces.

### 3.4 viewer

React app, one build used in two hosts: VS Code webview and the exported standalone HTML.

Views:

1. **System map** — services, DBs, caches, queues, external APIs, and edges between them.
2. **Request flow** — pick a route; sequence-style diagram of handler → services → DB/external/queue.
3. **Data model (ERD)** — tables, columns, relations from Prisma/Drizzle.
4. **Module dependency graph** — folders/modules and import edges; cycles highlighted.

- **Rendering:** graphs use React Flow with ELK auto-layout. The request flow uses a custom sequence renderer driven by the `Flow` call tree; branches are drawn as alt/opt frames.
- **Confidence:** edges are styled by confidence (dashed = unresolved), and an "Unknowns" panel lists unresolved edges and truncated steps.
- **Large graphs:** graphs with more than `config.maxVisibleNodes` (default 300) nodes collapse by service or folder and expand on click.
- **Side panel:** shows the selected node's template explanation, glossary link and "Ask AI". "Ask AI" is available in the extension host only.
- **Host bridge:** `openLocation(file, line)` and `askAi(...)` are implemented by the extension. In the standalone export they are hidden.
- **Security:** all model strings are untrusted input from the analyzed repo. They are rendered as text only, never with `dangerouslySetInnerHTML` or as HTML labels.

#### Viewer build & CSP

- **Bundling:** Vite builds one JS bundle and one CSS bundle. ELK runs in a Web Worker built from `elkjs/lib/elk-worker.min.js`.
- **Webview:**
  - CSP: `default-src 'none'; script-src 'nonce-…'; style-src ${webview.cspSource} 'unsafe-inline'; worker-src blob:; img-src ${webview.cspSource} data:`
  - `localResourceRoots` is limited to the extension's `dist/viewer`.
  - The worker is loaded through a blob URL.
- **Export:**
  - JS, CSS and the worker source are inlined into the HTML; the worker is created from a blob.
  - The model is embedded as `<script type="application/json" id="model">`, with `<`, `>`, `&`, U+2028 and U+2029 escaped as `\uXXXX`.
  - Budget: ≤ 3 MB HTML for a 1,000-node model.
- **Theming:** the webview follows VS Code theme CSS variables. The export follows `prefers-color-scheme`, with a toggle.

### 3.5 vscode-extension

- **Commands:** `Scan workspace`, `Open architecture map`, `Export (HTML/JSON)`, `Configure AI provider`.
- **Analysis process:** analysis runs in a child process (see 3.2 budget), with a progress notification and cancellation (the process is killed).
- **Navigation:** clicking a node opens the file at that line.
- **Re-scan:** on save, the affected service is re-scanned (debounced, see 3.2).
- **Export:**
  - Output is either a single self-contained `architecture.html` (viewer plus inlined model) or `model.json`.
  - **File paths are stripped by default.** Including them is an opt-in checkbox.
  - Before writing, the export dialog lists what the file contains: service, route, table, column, middleware and external host names.
- **Activation:** `onCommand:*` only. There is no automatic scan when a workspace opens.
- **Workspace Trust:** `untrustedWorkspaces: { supported: 'limited' }`. Analysis is allowed because it never executes repo code; AI is disabled. Virtual workspaces are unsupported.
- **Packaging:**
  - esbuild bundles the extension host code, with `typescript` bundled so analysis never uses the repo's own version.
  - `vsce package --no-dependencies` produces the VSIX.
  - VSIX target ≤ 15 MB.
- **No telemetry.**

### 3.6 Configuration

VS Code settings (`ductwork.*`) and an optional `ductwork.config.json` at the workspace root share one schema, so the CLI and the extension behave the same. The file overrides settings.

| Key | Default | Purpose |
|---|---|---|
| `include` / `exclude` | `["**"]` / `["**/node_modules/**", "**/dist/**", "**/build/**", "**/.next/**", "**/*.test.*", "**/*.spec.*"]` | Files analyzed |
| `services` | auto | Explicit service roots and optional per-service `tsconfig` |
| `serviceHosts` | `{}` | Host → service id, for cross-service edges |
| `detectors` | all on | Enable or disable detectors by name |
| `maxDepth` | 8 | Call-chain depth |
| `moduleDepth` | 2 | Module-graph folder granularity |
| `maxVisibleNodes` | 300 | Collapse threshold |
| `analyzerMemoryMb` | 2048 | Child process heap |
| `export.includePaths` | false | Keep file paths in exports |
| `ai.provider` / `ai.model` / `ai.baseUrl` | none / provider default / localhost for Ollama | AI settings (keys live in SecretStorage, never in the file) |
| `ai.includeSnippets` | true | Send code snippets or the model only |
| `ai.maxContextTokens` | 12000 | Payload cap |

### 3.7 Privacy & data inventory

| Data | Read | Stored | In export | Sent to AI |
|---|---|---|---|---|
| Source code (non-`.env`) | Yes, in memory | No | No | Only snippets, only on confirm, only if `includeSnippets` |
| `.env*` files | Never | No | No | No |
| Names (services, routes, functions, tables, columns, hosts) | Yes | Model in extension storage | Yes | Yes (model slice) |
| File paths and lines | Yes | Model, workspace-relative | Only if `export.includePaths` | Yes (relative) |
| Secret-looking literals | Detected | Redacted | Redacted | Redacted |
| API keys | — | VS Code SecretStorage | No | Sent only as auth to the chosen provider |
| Telemetry | — | — | — | None collected |

"Source never leaves the machine" holds by default. The only exception is AI with snippets, which the user explicitly opts into for each request through the preview.

### 3.8 cli

Published to npm as `ductwork` (`npx ductwork …`).

- `ductwork scan [dir] [--out model.json] [--html map.html] [--include-paths]` runs analyzer-ts.
  - `--html` writes the standalone viewer export (same safety rules as 3.4; paths stripped unless `--include-paths`).
  - This is the **validation build's** main entry point.
- `ductwork diff <base.json> <head.json> [--format md|json]` prints the architecture diff (see 3.10).
- `ductwork check [--base <git-ref>] [--rules ductwork.config.json]`:
  - Scans HEAD, scans the base ref in a temporary `git worktree`, diffs the two and evaluates the rules.
  - Exits non-zero on an `error`-level violation.
- `ductwork mcp [dir]` starts the MCP server over stdio (see 3.9).
- Exit codes: 0 ok, 1 rule violation, 2 scan failure, 3 bad usage.

### 3.9 MCP server

- **Purpose:** gives AI coding agents (Claude Code, Cursor, and others) exact architecture facts instead of guesses.
- **Distribution:** listed as a one-line install for those tools.
- **Transport:** stdio. It is started by the agent with `npx ductwork mcp`.
- **Startup:** it loads the cached model (`.ductwork/cache`, or the extension storage when that path is configured) or runs a scan. It re-scans the affected service on file change (same rule as the extension re-scan).
- **Tools (read-only), all returning JSON slices of the model:**
  - `list_services()`
  - `list_routes(serviceId?)`
  - `get_flow(routeId | "METHOD /path")` returns the call tree with middleware
  - `get_table(name)` returns columns, relations, and the routes that read or write it
  - `find_usages(nodeId)` returns callers/routes reaching a function or sink
  - `get_external_services()`
  - `search(query)` matches node names
- **Privacy:** it only returns model data, never source code. The agent can read the source itself if it wants.

### 3.10 PR architecture checks (paid tier candidate)

**Diff (`packages/diff`)** compares two models by stable IDs:

- Nodes and edges are added or removed by ID.
- A node is changed when its `meta` changed.
- A flow is changed when the set of sink edges or the middleware changed.
- Anonymous-function IDs are excluded from "changed" noise.

**Built-in rules.** Each rule's level (`off | warn | error`) is set in config:

| Rule | Triggers when |
|---|---|
| `route-without-auth` | A new or changed route has no middleware step with `role: 'auth'` and doesn't match `publicRoutes` globs |
| `new-external-service` | A new `external-api` node appears |
| `new-table-writer` | A route or worker starts writing a table it didn't write before |
| `sensitive-table-access` | A new flow reaches a table listed in `sensitiveTables` (for PII and compliance) |
| `new-cycle` | A new module-level import cycle |
| `removed-route` | A route disappears (possible breaking change) |
| `unresolved-increase` | The count of unresolved edges grows by more than N |

**Output:**

- A markdown summary: routes, tables, externals and flows that changed, plus violations.
- An optional Mermaid snippet of changed flows, so diagrams appear in the PR comment.

**GitHub Action (`packages/github-action`):**

- Runs `ductwork check` on `pull_request` and updates one sticky comment.
- **Source stays in the customer's CI runner.** Only for a paid private-repo license check does a license key and repo id leave CI, never the model.

**Licensing:**

- Free for public repos.
- On private repos, the action requires a `DUCTWORK_LICENSE_KEY` (verified against a minimal license endpoint built in M8). Until M8, design partners get free keys that are validated offline (signed tokens).

## 4. Data flow

1. User runs **Scan workspace**.
2. Extension spawns the analyzer child process with the resolved config.
3. Analyzer discovers packages, classifies services and libraries, and loads one ts-morph `Project`.
4. Phase 1: every applicable detector runs `discover` independently. Results are merged by the merge rules.
5. Phase 2: call-chain resolution builds flows from route and worker roots, using memoized function summaries and detector `classifyCall`.
6. The model is redacted, validated against the schema and written to extension storage. The scan report is shown.
7. The webview receives the model and renders the views.
8. Clicking a node makes the extension open the source location.
9. Selecting a node shows the template explanation and glossary. AI is optional, through preview-and-confirm.
10. Export writes the standalone HTML or JSON (paths stripped by default).
11. On save, the affected service is re-scanned and its slice of the model is replaced.

## 5. Error handling and limits

- **Confidence:** every edge and step carries a confidence. Unresolved ones are shown honestly, never guessed.
- **Detector isolation:** a detector that throws in `discover` or `classifyCall` is logged and disabled for the rest of the scan. Other results still render, and the scan report shows per-detector status.
- **Big repos:**
  - Analysis runs in a child process with a memory cap, a progress bar and cancellation.
  - When the budget is exceeded, the analyzer returns a partial model with `limitsHit`, and the UI shows a banner.
  - Graphs over `maxVisibleNodes` collapse.
- **Analyzer crash or OOM:** the extension reports the failure, keeps the last good model, and suggests raising `analyzerMemoryMb` or narrowing `include`.
- **Secrets:** `.env*` is never read. The model and AI payloads are redacted (best-effort), and AI requires a preview first.
- **AI failures** (bad key, rate limit, Ollama down, non-loopback host): an inline error is shown and the template explanation stays.
- **Unsupported stack:** the module graph is still produced, and the report lists unsupported frameworks detected.
- **Schema validation failure:** shown as a scan error with details. A corrupt model is never rendered.
- **Export with a newer schema major version than the viewer:** shows a "made with a newer version" message.

## 6. Testing and acceptance

**Tests**

- **Detector fixtures (TDD):**
  - Each detector has small sample apps with expected `model.json` snapshots.
  - Each framework fixture **must include nested mounts/prefixes, global and router middleware, and a wrapper (`asyncHandler`)**.
  - NestJS fixtures include `useClass`/`@Inject(TOKEN)` providers.
  - Next.js fixtures cover every path-derivation rule.
- **Resolver tests:** cycles, depth truncation, interfaces with one or several implementations, callbacks, branches (if/try/loop), sibling order, `node_modules` stop.
- **Real-repo smoke (CI):** the CLI runs on 2–3 pinned public OSS repos. The test checks:
  - the scan completes
  - node and edge counts stay within ±10% of a recorded baseline
  - the run stays within the 3.2 budget (≤ 60 s per 1,000-file service on the CI runner)
- **Viewer:** component tests render fixture models. One Playwright test runs against an exported HTML and includes a model containing `</script>` and `<img onerror>` in names, to confirm they are not executed.
- **Privacy invariant:** a test confirms that exported HTML/JSON contain no function bodies or string literals from fixture source, apart from names and redacted hosts.
- **Extension:** a `@vscode/test-electron` integration test covers scan, open map and click-to-code.
- **Explainer:** template snapshot tests, AI adapters tested with mocked HTTP, and redaction unit tests.

**Acceptance for success criterion 1 ("correctly maps")**

Hand-written golden expectations for the author's repos (Pingward first, then Safeship) are checked in as a private test script. Targets:

- 100% route recall, with correct method and full path
- 100% of tables and relations
- 100% of external services
- ≥ 90% of sink edges present across 10 hand-picked key flows
- 0 false sinks

## 7. Scope

### In v1

- 4 views; Express, Fastify, NestJS and Next.js API, with middleware and route composition; Prisma and Drizzle; fetch/axios/known SDKs; BullMQ and Redis; imports graph
- Template explanations with glossary; optional AI (Claude, OpenAI, Ollama)
- HTML/JSON export; CLI; configuration; VS Code Marketplace release (MIT license)

### Milestones (each gets its own implementation plan)

The order tackles the riskiest part, call-chain accuracy, first.

1. **M1 Core + Express/Prisma:**
   - core-model types and schema, analyzer skeleton, service discovery, two-phase detector contract, call-chain resolver, CLI
   - `express` and `prisma` detectors
   - **Exit gate:** fixtures pass, and the golden accuracy check passes on Pingward (if Pingward isn't Express/Prisma, on whichever author repo is).
2. **M2 Viewer (standalone):** system map, request flow, ERD and module graph rendered from fixture models; standalone HTML export with safe embedding.
3. **M3 VS Code extension:** scan command and child process, webview with CSP, click-to-code, export, re-scan on save, config.
4. **M4 Remaining detectors:** fastify, nestjs (with DI), nextjs-api, drizzle, http/SDKs with cross-service matching, bullmq, redis, imports cycles. **Exit gate:** golden check on Safeship.
5. **M5 Explainer:** templates and glossary, then the AI adapters (Claude, OpenAI, Ollama) with payload builder, redaction and preview.
6. **M6 Release:** packaging, Workspace Trust, README with screenshots, Marketplace listing, final name check.

### Later (not v1)

- v1.1: **Guided tours / teaching mode**, with step-by-step walkthroughs of a request flow
- Fine-grained incremental analysis
- Python analyzer (FastAPI, Django, Flask, SQLAlchemy), then Go and Java
- tRPC, GraphQL, TypeORM and Mongoose detectors
- Hosted share links, accounts, team spaces
- Architecture diff between commits/PRs
- JetBrains plugin, desktop app, mobile viewer
