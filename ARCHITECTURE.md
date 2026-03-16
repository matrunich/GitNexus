# GitNexus — Architecture & Developer Guide

> Deep-dive reference for contributors, integrators, and anyone curious about how GitNexus works under the hood.  For user-facing documentation see [README.md](./README.md).

---

## Table of Contents

1. [High-level Overview](#1-high-level-overview)
2. [Repository Layout](#2-repository-layout)
3. [Data Flow](#3-data-flow)
   - [CLI / MCP path](#cli--mcp-path)
   - [Browser (client-side) path](#browser-client-side-path)
   - [Bridge mode](#bridge-mode)
4. [6-phase Ingestion Pipeline](#4-6-phase-ingestion-pipeline)
5. [Knowledge Graph Schema](#5-knowledge-graph-schema)
6. [Key Algorithms](#6-key-algorithms)
   - [BFS Process Detection](#bfs-process-detection)
   - [Leiden Community Detection](#leiden-community-detection)
   - [Hybrid Search (BM25 + Semantic + RRF)](#hybrid-search-bm25--semantic--rrf)
7. [Graph RAG Agent (Web UI)](#7-graph-rag-agent-web-ui)
8. [MCP Server (CLI)](#8-mcp-server-cli)
9. [LadybugDB Integration](#9-ladybugdb-integration)
10. [TypeScript vs Python — What Lives Where and Why](#10-typescript-vs-python--what-lives-where-and-why)
11. [Performance Considerations](#11-performance-considerations)
12. [Security & Privacy Model](#12-security--privacy-model)
13. [Developer Setup](#13-developer-setup)

---

## 1. High-level Overview

GitNexus builds a **queryable knowledge graph** of any codebase and exposes it through two surfaces:

| Surface | Who uses it | What they get |
|---------|-------------|---------------|
| **MCP server** (`gitnexus mcp`) | AI coding agents (Cursor, Claude Code, Windsurf, OpenCode) | 7 graph-aware tools via Model Context Protocol |
| **Web UI** ([gitnexus.vercel.app](https://gitnexus.vercel.app)) | Developers doing one-off exploration | Interactive graph visualiser + in-browser AI chat |

Both surfaces share the same underlying graph model and search stack.  The CLI/MCP path runs natively in Node.js; the browser path compiles the same TypeScript to WebAssembly-compatible code that runs entirely in the user's browser.

### User Workflow (CLI + MCP)

```
1.  cd my-repo && npx gitnexus analyze
    → 6-phase pipeline indexes the repo into .gitnexus/ (LadybugDB)

2.  npx gitnexus mcp          (or run automatically by the editor)
    → stdio MCP server starts, advertising 7 tools

3.  AI agent calls tools:
      query("auth validation")     → process-grouped BM25+semantic results
      context("validateUser")      → callers, callees, processes
      impact("AuthService")        → blast-radius depth analysis
      detect_changes()             → git-diff → affected symbols & flows
      rename("validateUser", ...)  → coordinated multi-file rename
      cypher("MATCH ...")          → raw graph queries
```

### User Workflow (Web UI)

```
1.  Open gitnexus.vercel.app

2.  Upload a .zip of the repo  (or connect to a local gitnexus serve)
    → Web Worker runs the full pipeline in-browser
    → LadybugDB WASM holds the graph in memory

3.  Explore the Sigma.js force graph (zoom, click, community view)

4.  Open chat → LangChain ReAct agent answers questions using
    the same search/cypher/read/grep tools, streaming back citations
```

---

## 2. Repository Layout

```
GitNexus/
├── gitnexus/                    ← npm package (CLI + MCP + HTTP server)
│   ├── src/
│   │   ├── cli/                 # Commander.js CLI entry point
│   │   ├── core/
│   │   │   ├── ingestion/       # 6-phase indexing pipeline
│   │   │   │   ├── pipeline.ts              # Orchestrator
│   │   │   │   ├── structure-processor.ts   # Phase 1 – file tree
│   │   │   │   ├── parsing-processor.ts     # Phase 2 – Tree-sitter AST
│   │   │   │   ├── import-processor.ts      # Phase 3 – import edges
│   │   │   │   ├── call-processor.ts        # Phase 4 – CALLS edges
│   │   │   │   ├── heritage-processor.ts    # Phase 5 – class hierarchy
│   │   │   │   ├── community-processor.ts   # Leiden clustering
│   │   │   │   ├── process-processor.ts     # BFS execution-flow tracing
│   │   │   │   ├── resolution-context.ts    # Cross-file symbol resolution
│   │   │   │   ├── symbol-table.ts          # Per-file symbol registry
│   │   │   │   ├── type-env.ts              # self/this receiver inference
│   │   │   │   ├── entry-point-scoring.ts   # Heuristics for entry points
│   │   │   │   ├── ast-cache.ts             # LRU cache for parsed ASTs
│   │   │   │   ├── framework-detection.ts   # @Controller, @GetMapping, etc.
│   │   │   │   ├── call-routing.ts          # Language-specific dispatch
│   │   │   │   └── resolvers/               # Language resolvers
│   │   │   │       ├── standard.ts          #   JS/TS/Python/Go/Rust
│   │   │   │       ├── jvm.ts               #   Java/Kotlin (MRO)
│   │   │   │       ├── csharp.ts            #   C#
│   │   │   │       ├── php.ts               #   PHP
│   │   │   │       └── ruby.ts              #   Ruby
│   │   │   ├── graph/           # Knowledge graph types (nodes + edges)
│   │   │   ├── search/          # BM25 + hybrid search
│   │   │   │   ├── bm25-index.ts            # FTS via LadybugDB
│   │   │   │   └── hybrid-search.ts         # RRF merge logic
│   │   │   ├── embeddings/      # Embedding generation (transformers.js)
│   │   │   ├── lbug/            # LadybugDB connection pool adapter
│   │   │   ├── tree-sitter/     # Parser loader + WASM init
│   │   │   ├── augmentation/    # Graph augmentation helpers
│   │   │   └── wiki/            # LLM-powered wiki generation
│   │   ├── mcp/                 # Model Context Protocol server
│   │   │   ├── server.ts        # MCP server factory
│   │   │   └── tools.ts         # 7 tool definitions
│   │   ├── server/              # Express HTTP API (for bridge mode)
│   │   ├── storage/             # Repo registry + git utilities
│   │   ├── config/              # Supported languages table
│   │   └── types/               # Shared TypeScript types
│   ├── test/                    # Vitest unit + integration tests
│   └── vendor/leiden/           # Vendored Leiden algorithm (CommonJS)
│
├── gitnexus-web/                ← React + Vite browser app
│   ├── src/
│   │   ├── App.tsx              # App shell + mode detection
│   │   ├── components/          # UI components
│   │   │   ├── GraphCanvas.tsx  # Sigma.js WebGL graph
│   │   │   ├── RightPanel.tsx   # Chat + tool results
│   │   │   └── ...
│   │   ├── core/
│   │   │   ├── ingestion/       # Pipeline (browser-compatible subset)
│   │   │   ├── llm/             # Graph RAG agent (LangChain)
│   │   │   │   ├── agent.ts         # createReactAgent factory
│   │   │   │   ├── tools.ts         # 7 tool definitions (browser)
│   │   │   │   └── context-builder.ts  # Dynamic system prompt
│   │   │   ├── embeddings/      # transformers.js (WebGPU / WASM)
│   │   │   ├── search/          # Hybrid search
│   │   │   ├── graph/           # Client-side graph ops
│   │   │   └── lbug/            # LadybugDB WASM adapter
│   │   ├── workers/
│   │   │   └── ingestion.worker.ts  # Web Worker – runs pipeline off main thread
│   │   ├── services/            # HTTP backend client, ZIP utils
│   │   └── hooks/               # useAppState, useGraph, …
│   └── vendor/                  # Prebuilt WASM binaries
│
├── gitnexus-claude-plugin/      # Claude Desktop plugin + skill files
├── gitnexus-cursor-integration/ # Cursor skill files
├── eval/                        # Python SWE-bench evaluation suite
│   ├── agents/gitnexus_agent.py # Python wrapper calling Node MCP server
│   ├── bridge/mcp_bridge.py     # MCP stdio bridge
│   └── run_eval.py              # Benchmark orchestrator
├── AGENTS.md                    # Agent integration guide
├── CLAUDE.md                    # Claude Code–specific instructions
└── ARCHITECTURE.md              ← (this file)
```

---

## 3. Data Flow

### CLI / MCP path

```
┌─────────────────────────────────────────────────┐
│  Developer Machine                               │
│                                                 │
│  $ npx gitnexus analyze                         │
│        │                                        │
│        ▼                                        │
│  ┌─────────────────┐                            │
│  │  6-phase         │  Tree-sitter AST           │
│  │  Ingestion       │  + Leiden + BFS            │
│  │  Pipeline        │  + LadybugDB load          │
│  └────────┬────────┘                            │
│           │ writes                              │
│           ▼                                     │
│  .gitnexus/                                     │
│    ├── kuzu/       ← LadybugDB (graph + FTS)    │
│    ├── meta.json   ← index metadata             │
│    └── …                                        │
│                                                 │
│  $ npx gitnexus mcp  (started by editor)        │
│        │                                        │
│        ▼                                        │
│  ┌─────────────────┐   JSON-RPC 2.0 / stdio     │
│  │  MCP Server     │ ◄──────────────────────────┤ AI Agent
│  │  (7 tools)      │ ────────────────────────── │ (Cursor / Claude /
│  └─────────────────┘                            │  Windsurf / OpenCode)
└─────────────────────────────────────────────────┘
```

### Browser (client-side) path

```
User's Browser
┌────────────────────────────────────────────────────────┐
│                                                        │
│  1. Upload .zip ──► DropZone component                 │
│                                                        │
│  2. Web Worker (ingestion.worker.ts via Comlink)       │
│     ├─ Unzip (JSZip)                                   │
│     ├─ Run 6-phase pipeline (Tree-sitter WASM)         │
│     └─ Return SerializablePipelineResult               │
│                                                        │
│  3. Main thread receives graph + file contents         │
│     ├─ LadybugDB WASM holds graph in memory            │
│     ├─ Sigma.js renders force-directed graph (WebGL)   │
│     └─ React state (useAppState) distributes data      │
│                                                        │
│  4. User asks question in chat                         │
│     ├─ createGraphRAGAgent() (LangChain ReAct)         │
│     ├─ Agent tool loop:                                │
│     │    search → cypher → read → grep → impact        │
│     └─ Streamed response with [[file:line]] citations  │
│                                                        │
│  All processing is local — no code leaves the browser  │
└────────────────────────────────────────────────────────┘
```

### Bridge mode

`gitnexus serve` starts an Express HTTP server on `127.0.0.1:4747`.  The web UI auto-detects it and delegates graph queries to the local server instead of running the pipeline in-browser.  This removes the browser memory cap and gives the UI access to all CLI-indexed repos without re-uploading.

```
Browser                HTTP (localhost only)         Node.js server
  │                                                       │
  │  GET /api/repos   ─────────────────────────────►      │
  │  GET /api/graph   ─────────────────────────────►  LadybugDB
  │  POST /api/query  ─────────────────────────────►  (native)
  │  POST /api/search ─────────────────────────────►      │
  │◄─────────────────────────────────────────────── JSON  │
```

---

## 4. 6-phase Ingestion Pipeline

`gitnexus/src/core/ingestion/pipeline.ts` — `runPipelineFromRepo(repoPath, onProgress)`

Each phase emits progress events consumed by the CLI progress bar and the web worker proxy.

| Phase | Processor | What it does |
|-------|-----------|--------------|
| **1 – Structure** | `structure-processor.ts` | Walks the file tree; creates `File` and `Folder` nodes with `CONTAINS` edges |
| **2 – Parsing** | `parsing-processor.ts` | Reads source files in 20 MB chunks; uses Tree-sitter (native or WASM) to produce ASTs; extracts `Function`, `Class`, `Method`, `Interface`, `Enum`, `Variable` nodes per file |
| **3 – Import Resolution** | `import-processor.ts` | Resolves `import`/`require`/`use`/`include` statements to target module nodes; creates `IMPORTS` edges with confidence scores |
| **4 – Call Tracing** | `call-processor.ts` + `call-routing.ts` | Maps function call sites to definitions; handles `self`/`this` receiver via `type-env.ts`; creates `CALLS` edges |
| **5 – Heritage** | `heritage-processor.ts` + `mro-processor.ts` | Extracts `EXTENDS`/`IMPLEMENTS` edges; computes Method Resolution Order for Java/Kotlin; creates `OVERRIDES` edges |
| **6 – Enrichment** | `community-processor.ts`, `process-processor.ts`, `export-detection.ts` | Leiden clustering → Community nodes; BFS tracing → Process nodes; export flag propagation |

After the six phases, the graph is loaded into LadybugDB (`lbug/`) and a full-text search (FTS) index is built.  Optionally, embedding vectors are generated for all symbols (controlled by `--embeddings` / `--skip-embeddings`).

**Memory budget:** The pipeline reads source files in chunks capped at `CHUNK_BYTE_BUDGET = 20 MB`.  Each chunk's source text, parsed ASTs, extracted records, and worker serialisation overhead all live in memory simultaneously.  20 MB of source ≈ 200–400 MB peak working memory per chunk after AST expansion.

**Worker pool:** Parsing is CPU-intensive.  `workers/worker-pool.ts` spawns up to 4 Node.js worker threads (or Web Workers in the browser) that each run Tree-sitter on their assigned file slice.

---

## 5. Knowledge Graph Schema

### Node labels

| Label | Key properties | Description |
|-------|---------------|-------------|
| `File` | `path`, `language`, `size` | Source file |
| `Folder` | `path` | Directory |
| `Function` | `name`, `filePath`, `startLine`, `endLine`, `isExported`, `language` | Top-level function |
| `Method` | `name`, `filePath`, `startLine`, `endLine`, `className` | Class member |
| `Class` | `name`, `filePath`, `startLine`, `endLine`, `isExported` | Class / struct |
| `Interface` | `name`, `filePath` | Interface / trait / protocol |
| `Enum` | `name`, `filePath` | Enumeration |
| `Variable` | `name`, `filePath` | Top-level variable / constant |
| `Community` | `id`, `heuristicLabel`, `cohesion`, `symbolCount` | Leiden-detected cluster |
| `Process` | `id`, `name`, `entryPointId`, `stepCount`, `crossesCommunities` | BFS execution flow |

### Relationship types

| Type | Source → Target | Properties |
|------|----------------|------------|
| `CONTAINS` | `Folder → File`, `File → Symbol` | — |
| `CALLS` | `Symbol → Symbol` | `confidence`, `reason` |
| `IMPORTS` | `File → File` \| `Symbol` | `confidence` |
| `EXTENDS` | `Class → Class` | `confidence` |
| `IMPLEMENTS` | `Class → Interface` | `confidence` |
| `OVERRIDES` | `Method → Method` | `confidence` |
| `HAS_METHOD` | `Class → Method` | — |
| `MEMBER_OF` | `Symbol → Community` | — |
| `STEP_IN_PROCESS` | `Symbol → Process` | `stepIndex`, `depth` |

All edges in LadybugDB are stored as `CodeRelation` with a `type` property discriminating the relationship kind.  This allows generic graph traversal (`MATCH ()-[r:CodeRelation]-()`) as well as filtered traversal (`WHERE r.type = 'CALLS'`).

---

## 6. Key Algorithms

### BFS Process Detection

**File:** `gitnexus/src/core/ingestion/process-processor.ts`

```
1. Score all symbols for "entry-point-ness":
   - Public/exported symbols get higher scores
   - Framework entry-point patterns (e.g. @Controller, main(), handler())
   - Test files are excluded by default

2. Pick top-N entry points (N = maxProcesses = 75)

3. For each entry point, BFS over CALLS edges:
   - maxTraceDepth = 10 hops
   - maxBranching  = 4 branches per node
   - minSteps      = 3 (2-step "A calls B" is not a useful flow)

4. Deduplicate: overlapping flows that share > 70% of symbols
   are merged under the highest-priority entry point

5. Attach STEP_IN_PROCESS edges (stepIndex, depth)
```

The result is a set of named execution flows (e.g. "LoginFlow", "OrderCheckout") that give AI agents a structural mental model of the codebase without reading every file.

---

### Leiden Community Detection

**File:** `gitnexus/src/core/ingestion/community-processor.ts`

The **Leiden algorithm** (a refinement of Louvain) maximises graph modularity: it partitions nodes so that connections *within* a community are denser than connections *between* communities.

```
1. Build a Graphology in-memory graph from CALLS edges only

2. Run Leiden (vendored from graphology-communities-leiden)
   → assigns each symbol a community ID

3. Compute cohesion per community:
   cohesion = internal_edges / (internal_edges + external_edges)

4. Assign heuristic labels from the most-referenced file paths
   (e.g. symbols in src/auth/*.ts → label "auth")

5. Create Community nodes and MEMBER_OF edges
```

Communities let the `query` tool return process-grouped results ranked by which cluster they belong to, so agents get spatial context ("these functions are in the Auth cluster") alongside keyword/semantic results.

> **Note:** The Leiden algorithm is vendored as `vendor/leiden/index.cjs` because `graphology-communities-leiden` was never published to npm.  `community-processor.ts` loads it with `createRequire` to bridge ESM ↔ CommonJS.

---

### Hybrid Search (BM25 + Semantic + RRF)

**File:** `gitnexus/src/core/search/hybrid-search.ts`

```
query q
  │
  ├─► BM25 keyword search (LadybugDB FTS)  → ranked list A
  └─► Semantic embedding search (cosine)   → ranked list B
                                                    │
                                          RRF merge │
                                                    │
             RRF score(d) = Σ 1 / (K + rank_i + 1) │  K = 60
                                                    │
                              → unified ranked list C (top-k)
```

**Why RRF?**  Reciprocal Rank Fusion combines rankings without needing to normalise scores from heterogeneous systems.  The constant `K = 60` is the standard value from the literature (same as Elasticsearch, Pinecone, etc.).  Documents that appear highly in *both* lists get a strong boost; documents in only one list are still surfaced but ranked lower.

**Embedding model:** `snowflake-arctic-embed-xs` via `@huggingface/transformers` — small enough to run in the browser (WebGPU or WASM fallback) and in Node.js without a GPU.

---

## 7. Graph RAG Agent (Web UI)

**Files:** `gitnexus-web/src/core/llm/agent.ts`, `tools.ts`, `context-builder.ts`

The web chat UI uses a **LangChain ReAct agent** (via `@langchain/langgraph/prebuilt` `createReactAgent`).

### Supported LLM providers

| Provider | Class |
|----------|-------|
| OpenAI / compatible | `ChatOpenAI` |
| Azure OpenAI | `AzureChatOpenAI` |
| Google Gemini | `ChatGoogleGenerativeAI` |
| Anthropic Claude | `ChatAnthropic` |
| Ollama (local) | `ChatOllama` |
| OpenRouter | `ChatOpenAI` (custom base URL) |

API keys are stored in browser `localStorage` only — they are never sent to any server other than the chosen LLM provider.

### 7 agent tools

| Tool | Description |
|------|-------------|
| `search` | Hybrid search (BM25 + semantic + RRF), results grouped by process and cluster |
| `cypher` | Execute raw Cypher against LadybugDB WASM; `{{QUERY_VECTOR}}` is auto-replaced with the query embedding |
| `grep` | Regex search across file contents |
| `read` | Read a file path from the in-memory file contents map |
| `overview` | Returns community + process map (codebase spatial overview) |
| `explore` | 360-degree view of a symbol, community, or process |
| `impact` | Blast-radius analysis: upstream/downstream callers/callees with depth grouping |

### System prompt design

The `BASE_SYSTEM_PROMPT` enforces three invariants:

1. **Grounding** — every factual claim must include a file citation: `[[src/auth.ts:45-60]]`
2. **Validation** — the agent must use `cypher` to cross-check any search results before final output
3. **Investigation protocol** — Search → Read → Trace → Cite → Validate

These constraints are based on research into reducing hallucination in code-analysis agents (Aider/Cline methodology): short directives outperform long explanations, and explicit "no citation = no claim" rules are more effective than soft encouragements.

### Dynamic system prompt injection

`context-builder.ts` appends a live codebase snapshot (community list, top processes, entry-point count) at agent startup so the model has spatial context before the user asks the first question.

---

## 8. MCP Server (CLI)

**Files:** `gitnexus/src/mcp/server.ts`, `mcp/tools.ts`

The MCP server starts on stdio and implements the [Model Context Protocol](https://modelcontextprotocol.io/) spec.  It exposes the same 7 tools as the web agent, plus a `list_repos` discovery tool, all as JSON-RPC 2.0 messages.

### Resources (read-only URIs)

| URI pattern | Content |
|-------------|---------|
| `gitnexus://repo/{name}/context` | Repo overview (stats, languages, entry-point count) |
| `gitnexus://repo/{name}/clusters` | All community clusters |
| `gitnexus://repo/{name}/processes` | All detected execution flows |
| `gitnexus://repo/{name}/process/{flowName}` | Step-by-step flow trace |
| `gitnexus://repo/{name}/schema` | Graph schema |

Resources are addressable from AI agent context without a tool call, which saves round-trips for orientation queries.

### LocalBackend

`mcp/server.ts` uses a `LocalBackend` class that opens every indexed repo found in `~/.gitnexus/registry.json` at startup.  Each repo's database is accessed via a session lock (see §9) that serialises concurrent tool calls, so requests for different repos queue cleanly without race conditions.

---

## 9. LadybugDB Integration

**Files:** `gitnexus/src/core/lbug/lbug-adapter.ts`

LadybugDB (formerly KuzuDB) is an embedded property graph database with:
- Cypher query language
- Full-text search (FTS) for BM25
- Vector / embedding column support

**Connection management pattern:**
```
initLbug(repoPath)           → opens DB + creates tables (one DB at a time)
executeQuery(cypher)         → acquires session lock, runs query, releases lock
searchFTSFromLbug(query, k)  → calls LadybugDB FTS API under session lock
withLbugDb(repoPath, fn)     → switches active DB atomically, runs fn
```

The adapter keeps a single active connection protected by a sequential session lock (`sessionLock` promise chain).  Switching to a different repo's DB closes the previous connection first.  This serialises all DB access so concurrent tool calls queue up cleanly rather than racing on the underlying native bindings.

A native `console` patch suppresses LadybugDB's internal C++ stdout logs that would otherwise corrupt the MCP stdio transport.

In the **browser**, `@ladybugdb/wasm` is used instead of the native bindings.  The WASM binary is bundled in `gitnexus-web/vendor/`.  The adapter API is identical, so all ingestion and query code is shared without modification.

---

## 10. TypeScript vs Python — What Lives Where and Why

| Component | Language | Reason |
|-----------|----------|--------|
| Core ingestion pipeline | TypeScript (Node.js) | Same code runs in browser (compiled to WASM-safe JS) and in Node.js — sharing one implementation eliminates drift |
| MCP server | TypeScript (Node.js) | MCP SDK is TypeScript-first; stdio transport requires Node.js |
| HTTP server | TypeScript (Node.js) | Shared with MCP server binary |
| Web UI | TypeScript (React + Vite) | Browser target; full type safety for graph schema |
| LangChain agent | TypeScript (LangChain.js) | LangChain.js runs in both Node.js and browser |
| Tree-sitter parsers | TypeScript bindings over C | Tree-sitter core is C; the Node.js and WASM bindings are maintained by the Tree-sitter project |
| SWE-bench eval suite | Python | SWE-bench harness is Python; the eval agents call the Node.js MCP server via subprocess |

**Key design constraint:** the ingestion pipeline (`gitnexus/src/core/ingestion/`) must be browser-compatible.  Any Node.js-specific API (e.g. `fs`, `path`, worker threads) is injected via dependency inversion so the browser can substitute `JSZip` and `Web Workers` without changing pipeline logic.

---

## 11. Performance Considerations

| Concern | Mitigation |
|---------|-----------|
| Large repos (> 5k files) | 20 MB chunked file reading; LRU AST cache (`AST_CACHE_CAP = 50`); streaming progress events |
| Parallel parsing | Worker pool (4 threads) distributes CPU-bound Tree-sitter work |
| DB connection overhead | Single serialised connection per active repo (session lock); switching repos closes the previous DB atomically |
| Embedding generation | Optional (`--skip-embeddings` is the default for fast indexing); `snowflake-arctic-embed-xs` is ~22 MB — small enough for browser WASM |
| Graph query latency | Cypher queries against LadybugDB are typically < 10 ms for queries that stay within a community; cross-community traversals touching thousands of edges may reach 50–200 ms |
| Browser memory | In-browser mode is capped at ~5 k files by available RAM; bridge mode removes this cap |
| BFS process tracing | Bounded: `maxTraceDepth = 10`, `maxBranching = 4`, `maxProcesses = 75`; keeps worst-case complexity manageable |

---

## 12. Security & Privacy Model

**Privacy-first design:**

- **CLI path:** Everything runs locally.  No source code, graph data, or telemetry is sent to any external server.  The index lives in `.gitnexus/` (automatically gitignored) and `~/.gitnexus/` (path metadata only).
- **Browser path:** Everything runs in the user's browser tab.  The only outbound network calls are to the LLM provider chosen by the user (OpenAI, Anthropic, Google, Ollama, etc.).  No code is sent to gitnexus.vercel.app — the site serves only the static JS bundle.
- **API keys:** Stored in browser `localStorage`; never transmitted to any GitNexus infrastructure.

**HTTP server (bridge mode):**

- Binds to `127.0.0.1` (loopback only) by default.
- CORS whitelist: `localhost:*` and `gitnexus.vercel.app`.
- No authentication — assumes the loopback interface is trusted.
- Do **not** expose the server on a public interface (`--host 0.0.0.0`) without adding an authentication layer.

**MCP server:**

- Communicates only over stdio with the calling process — no network socket is opened.
- The `rename` tool modifies files on disk; always use `dry_run: true` first and review the diff.

---

## 13. Developer Setup

### Prerequisites

- Node.js ≥ 18 (20 LTS recommended)
- npm ≥ 9

### Build the core library

```bash
cd gitnexus
npm install
npm run build          # Compiles TypeScript → dist/
```

### Run the CLI from source

```bash
cd gitnexus
npm run dev            # tsx watch — rebuilds on file changes
```

### Run tests

```bash
cd gitnexus
npm test               # Unit tests (vitest)
npm run test:integration  # Integration tests (requires built dist/)
npm run test:all       # Unit + integration
npm run test:coverage  # Coverage report
```

### Build and run the web UI

```bash
cd gitnexus-web
npm install
npm run dev            # Vite dev server → http://localhost:5173
npm run build          # Production build → dist/
npm run preview        # Serve production build locally
```

### Run the eval suite (Python)

```bash
cd eval
pip install -e .
python run_eval.py --help
```

The eval agents start the Node.js MCP server as a subprocess via `bridge/mcp_bridge.py`, so the core library must be built first.

### Environment variables

| Variable | Used by | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | CLI `wiki`, web agent | OpenAI API access |
| `ANTHROPIC_API_KEY` | Web agent | Anthropic Claude API |
| `GOOGLE_API_KEY` | Web agent | Google Gemini API |
| `AZURE_OPENAI_API_KEY` | Web agent | Azure OpenAI API |
| `GITNEXUS_DATA_DIR` | CLI | Override default `~/.gitnexus/` registry location |
| `NODE_ENV=development` | CLI | Enables verbose debug logging |

### Useful development commands

```bash
# Index the GitNexus repo itself (meta!)
npx gitnexus analyze

# Start MCP server pointing at all indexed repos
npx gitnexus mcp

# Start HTTP bridge server on custom port
npx gitnexus serve --port 4747

# Generate wiki from the current repo's graph
OPENAI_API_KEY=sk-... npx gitnexus wiki

# Query the graph directly from the CLI
npx gitnexus query "ingestion pipeline"
npx gitnexus context runPipelineFromRepo
npx gitnexus impact mergeWithRRF --direction upstream
```
