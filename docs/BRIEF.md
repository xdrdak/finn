# finn

> Module graph explorer for TypeScript/JavaScript projects. TUI-first.

---

## Features

### Module Graph Explorer

- Build full dependency graph from any entrypoint
- Interactive navigation: drill into a module, see what it imports/exports, walk the tree
- Render subgraphs as mermaid diagrams (ASCII in terminal)
- Filter by depth, folder, pattern

### Blast Radius (git diff)

- Feed a git diff (or staged files), see every module affected downstream
- Highlights changed files + their transitive dependents
- Answers: "what breaks if I change this file?"

### Circular Dependency Detection

- Detect all cycles in the graph
- Show each cycle as a navigable path
- Highlight circular edges in the rendered graph

### Symbol-Level Analysis

- Track which exported functions/classes/types are used where
- Per-symbol: list of consumers (file + line)
- Identify unused exports (dead code candidates)
- Powered by TS type checker, not regex

---

## Tech Stack

| Concern | Library |
|---|---|
| Dependency extraction + resolution | `dependency-cruiser` (file-level graph, cruise API) |
| Symbol analysis + reference tracking | TypeScript Compiler API (`ts.createProgram`, type checker) |
| Graph data structure + algorithms | `graphology` (cycle detection, traversal, filtering) |
| Mermaid rendering (terminal) | `beautiful-mermaid` (renderMermaidASCII) |
| TUI framework | `@opentui/react` (or `ink` as fallback) |
| Git integration | `simple-git` (diff extraction) |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    finn CLI / TUI                     │
│         (navigation, rendering, user input)           │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│                   Graph Service                      │
│  - builds/caches the unified graph                   │
│  - queries: dependents, dependencies, cycles, reach  │
│  - filters: by depth, path, diff                     │
│  - generates mermaid strings for subgraphs           │
└──────────┬───────────────────────┬──────────────────┘
           │                       │
           ▼                       ▼
┌────────────────────┐  ┌────────────────────────────┐
│  File-Level Adapter │  │  Symbol-Level Adapter       │
│                     │  │                             │
│  interface:         │  │  interface:                 │
│   extractImports()  │  │   extractExports()          │
│   resolveModule()   │  │   findReferences()          │
│   getFileGraph()    │  │   getSymbolUsages()         │
│                     │  │                             │
│  impl: dep-cruiser  │  │  impl: TS Compiler API      │
│  (cruise API, JSON) │  │  (Program + TypeChecker)     │
└────────────────────┘  └────────────────────────────┘
           │                       │
           ▼                       ▼
┌─────────────────────────────────────────────────────┐
│                 graphology Graph                      │
│  nodes = files (modules)                             │
│  edges = imports (with symbol metadata)              │
│  node attrs: path, isExternal, isEntrypoint          │
│  edge attrs: importedSymbols[], isCircular, isTypeOnly│
└─────────────────────────────────────────────────────┘
```

### Key design decisions

**Adapter interface is language-gated.** TS/JS adapter uses dep-cruiser + TS compiler. Future adapters (Go, Rust, Python) implement the same interface with different backends. Symbol-level analysis (`findReferences`) is optional per adapter — file-level graph is the minimum.

**Graph is built once, queried many times.** The TUI doesn't re-cruise on every interaction. Graph is loaded into graphology on startup (or on file watch trigger), then all navigation/filtering/cycle detection runs against the in-memory graph.

**Mermaid is generated from the graph, not from dep-cruiser's mermaid reporter.** We build mermaid strings ourselves from graphology subgraphs. This gives us full control over what's rendered: focused views, highlighted cycles, depth-limited slices, blast radius overlays.

**Two-speed analysis.** File-level graph (dep-cruiser) is fast and always available. Symbol-level analysis (TS compiler) is heavier and loaded on-demand when user drills into a specific module's exports.
