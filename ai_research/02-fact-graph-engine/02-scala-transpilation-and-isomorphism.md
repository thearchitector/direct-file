# Scala Transpilation and Isomorphic Execution

## The Core Problem

Tax calculations must happen in two places simultaneously:
1. **On the client (browser)**: For instant feedback as users enter data — no waiting for server round-trips
2. **On the server (JVM)**: For authoritative computation before submission, and for backend services that generate XML/PDF

If two separate implementations exist (e.g., JavaScript on the client, Java on the server), they WILL diverge. A rounding difference, an off-by-one error, or a missed edge case would mean the user sees one tax amount on screen but files a different amount. This is unacceptable for a tax filing system.

## The Solution: Scala.js

DirectFile solves this by writing the Fact Graph engine in **Scala** and compiling it to two targets:

1. **JVM bytecode** (via standard Scala compiler) — runs on the backend as a Java library
2. **JavaScript** (via Scala.js transpiler) — runs in the browser

The same source code produces both outputs, guaranteeing identical behavior.

### Project Structure

```
direct-file/fact-graph-scala/
├── shared/          # Scala source code that compiles to BOTH JVM and JS
│   └── src/main/scala/   # The core Fact Graph engine
├── jvm/             # JVM-specific code (if any)
├── js/              # JS-specific code and transpiled output
│   └── target/scala-3.2.0/fact-graph-opt/
│       ├── main.js      # Transpiled JavaScript
│       └── main.js.map  # Source maps for debugging
├── build.sbt        # SBT build configuration
└── project/         # SBT project configuration
```

### Build Process

1. **Compile + Test**: `sbt test` — runs all Scala unit tests on the JVM
2. **Fast Transpile** (dev): `sbt fastOptJS` — quick JS output for development
3. **Full Transpile** (prod): `sbt fullOptJS` — optimized JS output for production
4. **Copy to Frontend**: `npm run copy-transpiled-js` in `df-client-app` copies the transpiled JS to `js-factgraph-scala/src/`

### How the Frontend Uses the Transpiled Code

The transpiled Scala code is consumed as a JavaScript module by the React frontend:

```
direct-file/df-client/js-factgraph-scala/  # Contains the transpiled JS
direct-file/df-client/df-client-app/       # The React app that imports it
```

The frontend imports Fact Graph operations from `@irs/js-factgraph-scala`, which exposes:
- `FactGraph` class — the main graph instance
- Factory types for creating/parsing fact values (e.g., `TinFactory`, `DollarFactory`)
- Path resolution utilities
- Serialization/deserialization (to/from JSON)

### Global Debug Variables

In development mode, the following globals are exposed in the browser console:
- `debugFactGraph` — The current fact graph instance (can call `.toJson()`, `.get()`, `.set()`, `.save()`)
- `debugFacts` — Shorthand for fact inspection
- `debugFactGraphMeta` — Metadata about the graph structure
- `debugScalaFactGraphLib` — Direct access to Scala factory methods (e.g., `debugFactGraphScala.IpPinFactory('000000')`)

These are invaluable for development and debugging — you can inspect any fact's value, test Scala factory methods, and manipulate the graph directly from the browser console.

## The Fact Graph Lifecycle on the Frontend

### Initialization
1. On app startup, the client fetches the **Fact Dictionary** (serialized as JSON by the backend)
2. A mostly-empty `FactGraph` instance is created from this dictionary
3. This instance is stored as a **global singleton** — it is THE source of truth for all tax data on the client
4. The graph is provided to React components via a context (`FactGraphContext`)

### Reading Facts
```typescript
const { factGraph } = useFactGraph();
const result = factGraph.get('/filers/#uuid/firstName');
if (result.complete) {
  const value = result.get; // The actual value
}
```

### Writing Facts
```typescript
factGraph.set('/filers/#uuid/firstName', 'John');
factGraph.save(); // Triggers re-derivation of all dependent facts
```

### Saving to Server
When the user clicks "Save and continue" on a screen, the writable facts are serialized and sent to the backend API, which encrypts and persists them.

## The Fact Graph on the Backend

On the JVM side, the same Scala code runs natively. The backend uses it for:
1. **Authoritative computation**: Before generating XML/PDF, the backend loads the saved fact graph and re-derives all values to ensure they match the client
2. **Validation**: The backend verifies that all required facts are complete before allowing submission
3. **XML generation**: The backend reads fact values from the JVM-side graph to populate MeF XML
4. **PDF generation**: Same process but for PDF form fields

## Key Technologies

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Language | Scala 3 | Core graph engine implementation |
| JVM Target | Standard Scala compiler | Backend library |
| JS Target | Scala.js | Frontend library |
| Build Tool | SBT (Scala Build Tool) | Compilation and transpilation |
| JS Module Format | ES6 Modules (.mjs) | Compatible with modern bundlers |
| Testing | ScalaTest / Specs | Unit tests run on JVM |

## Implications for a Replacement System

This isomorphic execution model is one of the most architecturally significant decisions in DirectFile. A replacement system must address the same fundamental problem: **how to ensure client and server tax calculations are identical**.

### Options for a Python-backend replacement:

1. **Server-only computation**: All tax math happens on the server. The client sends data via API calls and receives computed results. This is simpler but introduces latency — every keystroke might need a round-trip.

2. **Duplicated implementations**: Write tax logic in both Python (server) and JavaScript (client). This is dangerous — the implementations WILL diverge unless extremely rigorous testing prevents it.

3. **Shared computation via WebAssembly**: Write the tax engine in a language that compiles to both native code and WebAssembly (Rust, Go, C). The WASM module runs in the browser. This is the closest analog to Scala.js.

4. **JavaScript everywhere**: Write the tax engine in JavaScript/TypeScript that runs on both Node.js (server) and the browser. This is the simplest approach but means the Python backend delegates tax computation to a Node.js subprocess or service.

5. **Python → JS transpilation**: Tools like Transcrypt or Brython exist but are not production-grade for complex computation.

The recommended approach for a Python+React stack would likely be option (4): a shared TypeScript tax engine that runs isomorphically, with the Python backend calling it via a Node.js subprocess or embedding a JS runtime.
