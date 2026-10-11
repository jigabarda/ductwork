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

- Discovers services: each `package.json` in the workspace (excluding `node_modules`) is a candidate service; it is analyzed with its `tsconfig.json`, or with `allowJs` defaults if it has none (monorepo support).
- Loads each service with ts-morph.
- **Detector interface**: `detect(ctx: AnalysisContext) => Partial<Model>`; detectors are independent and isolated.
- v1 detectors:
  - Frameworks: `express`, `fastify`, `nestjs`, `nextjs-api` (app & pages router route handlers)
  - Data: `prisma` (schema.prisma → tables/relations; `prisma.x.find*/create*/...` → reads/writes), `drizzle` (table definitions; query builder calls)
  - Outbound: `http-clients` (fetch, axios) and known SDKs (v1 list: Stripe, Resend, Supabase, AWS S3, OpenAI, Anthropic; list is a data table, easy to extend) → `external-api` nodes; unknown fetch/axios targets become `external-api` nodes named by host when the URL is a literal, otherwise `unresolved`
  - Async: `bullmq` (queues, producers, workers), `redis` (ioredis/redis clients → cache)
  - Structure: `imports` (module graph by folder)
- **Call-chain resolution**: from each route handler, follow calls via the type checker across files up to depth 8; record paths reaching db/external/queue/cache nodes into `flows`. Unresolvable calls (dynamic dispatch, computed members) produce `unresolved` edges rather than guesses.
- Output merged and written to `.archmap/model.json` in the workspace.
- Incremental: on file save, re-analyze affected files and patch the model.
- Never reads `.env*` files.

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
