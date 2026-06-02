# TreeSitterPlugin

The `TreeSitterPlugin` is the core structural-analysis plugin for the `understand-anything-plugin` codebase. It uses Tree-sitter grammars to parse source files and extract language-aware structure such as functions, classes, imports, exports, and call graphs. It is designed to be config-driven, so the set of supported languages can be expanded or reduced through `LanguageConfig` entries without changing the plugin implementation.

This module is part of the core plugin system and is typically used as the fast, deterministic analysis layer before higher-level analysis modules build graphs, summaries, or LLM-assisted interpretations.

---

## Purpose and responsibilities

`TreeSitterPlugin` provides:

- **Synchronous structural analysis** after asynchronous initialization
- **Language selection by file extension**
- **Tree-sitter grammar loading from language configuration**
- **Extractor-based parsing** for structure and call graphs
- **Graceful fallback** when a grammar or extractor is unavailable
- **Import resolution** for relative and package-style imports

It implements the shared `AnalyzerPlugin` contract, which makes it compatible with the broader analyzer pipeline.

---

## Module placement in the system

This plugin sits inside the core plugin system and depends on shared types, language configuration, and language-specific extractors.

```mermaid
flowchart LR
  A[File path and content] --> B[TreeSitterPlugin]
  B --> C[Tree-sitter parser]
  C --> D[Language-specific extractor]
  D --> E[StructuralAnalysis]
  D --> F[CallGraph entries]
  E --> G[Graph builder and downstream analyzers]
  F --> G
  B --> H[ImportResolution]
```

### Related modules

- [Plugin registry](plugin_registry.md) â€” manages plugin lifecycle and selection
- [Plugin discovery](plugin_discovery.md) â€” discovers available plugins
- [Shared graph and analysis types](shared_graph_and_analysis_types.md) â€” defines `AnalyzerPlugin`, `StructuralAnalysis`, `ImportResolution`, and `CallGraphEntry`
- [Language support](core_language_support.md) â€” language registry and extractor ecosystem
- [Core analysis](core_analysis.md) â€” downstream consumers of structural output

---

## Architecture overview

The plugin is composed of three main layers:

1. **Configuration layer**
   - Accepts `LanguageConfig[]`
   - Builds extension-to-language mappings
   - Determines which grammars should be loaded

2. **Runtime parsing layer**
   - Initializes `web-tree-sitter`
   - Loads WASM grammars
   - Creates parsers on demand for each file

3. **Extraction layer**
   - Delegates AST traversal to a `LanguageExtractor`
   - Produces structural analysis and call graph data

```mermaid
flowchart TB
  subgraph Config[Configuration]
    LC[LanguageConfig]
    EX[LanguageExtractor]
  end

  subgraph Plugin[TreeSitterPlugin]
    MAP[Extension to language map]
    INIT[init]
    PARSER[getParser]
    ANALYZE[analyzeFile]
    IMPORTS[resolveImports]
    CALLS[extractCallGraph]
  end

  subgraph Runtime[Tree-sitter runtime]
    WT[web-tree-sitter Parser]
    WL[web-tree-sitter Language]
    WASM[WASM grammars]
  end

  subgraph Extractors[Language extractors]
    LE[LanguageExtractor]
  end

  LC --> MAP
  EX --> LE
  INIT --> WT
  INIT --> WL
  INIT --> WASM
  MAP --> PARSER
  PARSER --> WT
  ANALYZE --> LE
  CALLS --> LE
  IMPORTS --> ANALYZE
```

---

## Core API

### `TreeSitterPlugin`

Implements `AnalyzerPlugin`.

#### Public fields

- `name = "tree-sitter"`
- `languages: string[]` â€” supported language IDs

#### Constructor

```ts
constructor(configs?: LanguageConfig[], extractors?: LanguageExtractor[])
```

##### Behavior

- Filters configs to those that include `treeSitter`
- Builds extension-to-language mappings from config extensions
- Falls back to TypeScript and JavaScript when no configs are provided
- Registers either provided extractors or built-in extractors

##### Backward compatibility fallback

If no configs are passed, the plugin defaults to:

- `typescript`
- `javascript`

and maps common extensions such as `.ts`, `.tsx`, `.js`, `.jsx`, `.mjs`, and `.cjs`.

---

## Initialization lifecycle

`TreeSitterPlugin` must be initialized before use.

### `init(): Promise<void>`

Loads the Tree-sitter runtime and all configured grammars.

#### Process flow

```mermaid
sequenceDiagram
  participant Caller
  participant Plugin as TreeSitterPlugin
  participant Runtime as web-tree-sitter
  participant FS as WASM package resolution

  Caller->>Plugin: init()
  Plugin->>Runtime: import web-tree-sitter
  Plugin->>Runtime: Parser.init()
  Plugin->>FS: resolve grammar paths
  Plugin->>Runtime: Language.load(wasm)
  Runtime-->>Plugin: loaded languages
  Plugin-->>Caller: ready
```

#### Important details

- Uses `createRequire(import.meta.url)` to resolve `.wasm` files from packages
- Loads grammars in parallel with `Promise.all`
- Special-cases TypeScript to also attempt loading a TSX grammar
- Skips missing grammars gracefully instead of failing the whole plugin
- Sets an internal initialized flag to prevent repeated work

#### Legacy fallback mode

When no configs are supplied, the plugin loads grammars directly for:

- TypeScript
- TSX
- JavaScript

This preserves compatibility with older setups.

---

## File-to-language resolution

The plugin determines the language key from the file extension.

### `languageKeyFromPath(filePath: string): string | null`

Rules:

- `.tsx` maps to synthetic language key `tsx`
- Other extensions are looked up in the extension map built from `LanguageConfig.extensions`
- Unknown extensions return `null`

### `getParser(filePath: string): Parser | null`

Creates a parser for the fileâ€™s language.

Behavior:

- Throws if called before `init()`
- Returns `null` when the file extension is unsupported
- Returns `null` when the grammar was not loaded successfully
- Creates a fresh parser instance per call
- Assigns the loaded language before parsing

```mermaid
flowchart TD
  A[filePath] --> B{extension known?}
  B -- no --> C[null]
  B -- yes --> D{grammar loaded?}
  D -- no --> C
  D -- yes --> E[create parser]
  E --> F[set language]
  F --> G[parser ready]
```

---

## Structural analysis

### `analyzeFile(filePath: string, content: string): StructuralAnalysis`

Parses a file and extracts structural information.

#### Output shape

Returns a `StructuralAnalysis` object containing:

- `functions`
- `classes`
- `imports`
- `exports`

The broader schema also allows optional non-code sections, definitions, services, endpoints, steps, and resources, but this plugin currently focuses on code structure.

#### Behavior

1. Resolve parser for the file
2. Parse source content into a syntax tree
3. Select the matching extractor
4. Extract structure from the root AST node
5. Clean up parser and tree resources

#### Failure handling

- Unsupported language â†’ empty structural result
- Missing grammar â†’ empty structural result
- Parse failure â†’ empty structural result
- Missing extractor â†’ empty structural result

```mermaid
flowchart LR
  A[file content] --> B[getParser]
  B -->|null| C[empty StructuralAnalysis]
  B --> D[parse AST]
  D -->|fail| C
  D --> E[getExtractor]
  E -->|none| C
  E --> F[extractStructure rootNode]
  F --> G[StructuralAnalysis]
```

---

## Import resolution

### `resolveImports(filePath: string, content: string): ImportResolution[]`

Builds resolved import records from the structural analysis.

#### Resolution rules

- Relative imports (`./` or `../`) are resolved against the fileâ€™s directory using `path.resolve`
- Non-relative imports are returned as-is in `resolvedPath`
- Specifiers are preserved from the structural analysis

#### Notes

This method does not perform package resolution, alias resolution, or filesystem existence checks. It provides a lightweight path normalization step for downstream graph construction.

```mermaid
flowchart TD
  A[imports from analyzeFile] --> B{relative?}
  B -- yes --> C[resolved relative path]
  B -- no --> D[resolvedPath equals source]
  C --> E[ImportResolution]
  D --> E
```

---

## Call graph extraction

### `extractCallGraph(filePath: string, content: string): CallGraphEntry[]`

Extracts caller-to-callee relationships from the AST.

#### Behavior

- Uses the same parser selection and initialization rules as `analyzeFile`
- Parses the file content
- Delegates call graph extraction to the language extractor
- Returns an empty array when parsing or extraction is unavailable

#### Output

Each `CallGraphEntry` contains:

- `caller`
- `callee`
- `lineNumber`

```mermaid
flowchart LR
  A[file content] --> B[getParser]
  B -->|null| C[empty array]
  B --> D[parse AST]
  D -->|fail| C
  D --> E[getExtractor]
  E -->|none| C
  E --> F[extractCallGraph rootNode]
  F --> G[CallGraphEntry array]
```

---

## Extractor system

The plugin does not hard-code language-specific AST traversal. Instead, it delegates to `LanguageExtractor` implementations.

### `LanguageExtractor`

A language extractor must provide:

- `languageIds: string[]`
- `extractStructure(rootNode)`
- `extractCallGraph(rootNode)`

### Registration

```ts
registerExtractor(extractor: LanguageExtractor): void
```

Registers the extractor for all language IDs it supports.

### Language lookup behavior

- Extractors are stored by language ID
- `tsx` is treated as a synthetic alias and mapped to `typescript` extractor logic
- This allows TSX parsing without requiring a separate extractor implementation

```mermaid
flowchart TB
  A[LanguageExtractor] --> B[languageIds]
  B --> C[registerExtractor]
  C --> D[extractor map]
  D --> E[getExtractor]
  E --> F[extractStructure or extractCallGraph]
```

---

## Dependencies

### Direct dependencies

- `node:module` â†’ `createRequire` for resolving WASM assets
- `node:path` â†’ `dirname`, `resolve`, `extname`
- `web-tree-sitter` â†’ parser and language runtime
- `../types.js` â†’ shared analyzer contracts and result types
- `../languages/types.js` â†’ `LanguageConfig`
- `./extractors/types.js` â†’ `LanguageExtractor`
- `./extractors/index.js` â†’ built-in extractor registry

### Dependency diagram

```mermaid
flowchart LR
  TPS[TreeSitterPlugin] --> T[AnalyzerPlugin / StructuralAnalysis / ImportResolution / CallGraphEntry]
  TPS --> LC[LanguageConfig]
  TPS --> LE[LanguageExtractor]
  TPS --> BE[builtinExtractors]
  TPS --> WTS[web-tree-sitter]
  TPS --> PATH[node:path]
  TPS --> MOD[node:module]
```

---

## Data flow summary

```mermaid
flowchart TD
  A[Source file path + content] --> B[Extension lookup]
  B --> C[Language key]
  C --> D[Loaded grammar]
  D --> E[Tree-sitter parse]
  E --> F[AST root node]
  F --> G[LanguageExtractor]
  G --> H[StructuralAnalysis]
  G --> I[CallGraph entries]
  H --> J[Graph builder / analyzers]
  I --> J
  H --> K[Import resolution]
```

---

## Error handling and graceful degradation

The plugin is intentionally tolerant of missing capabilities.

### It returns empty results when:

- the file extension is unsupported
- the grammar failed to load
- parsing fails
- no extractor exists for the language

### It throws when:

- a synchronous analysis method is called before `init()`

This design keeps the plugin safe to use in mixed-language repositories where only some languages have Tree-sitter support.

---

## Integration with the wider system

The plugin is typically used as the first-pass structural analyzer.

- It feeds structural data into graph-building and normalization modules
- It complements LLM-based analyzers by handling languages with available grammars deterministically
- It supports downstream search, dependency mapping, and dashboard visualization by providing consistent code structure metadata

For downstream consumers, see:

- [Core analysis](core_analysis.md)
- [Graph builder](analyzer_graph_builder.md)
- [Normalize graph](analyzer_normalize_graph.md)
- [LLM analyzer](analyzer_llm_analyzer.md)

```mermaid
flowchart LR
  A[TreeSitterPlugin] --> B[GraphBuilder]
  A --> C[Normalize graph]
  A --> D[LLM analyzer fallback]
  B --> E[KnowledgeGraph]
  C --> E
  D --> E
```

---

## Practical usage pattern

1. Construct the plugin with language configs and extractors
2. Await `init()` once at startup
3. Call `analyzeFile()` for structural metadata
4. Call `resolveImports()` when building dependency edges
5. Call `extractCallGraph()` when building call relationships

```mermaid
sequenceDiagram
  participant App
  participant Plugin as TreeSitterPlugin

  App->>Plugin: new TreeSitterPlugin(configs)
  App->>Plugin: await init()
  App->>Plugin: analyzeFile(path, content)
  Plugin-->>App: StructuralAnalysis
  App->>Plugin: resolveImports(path, content)
  Plugin-->>App: ImportResolution[]
  App->>Plugin: extractCallGraph(path, content)
  Plugin-->>App: CallGraphEntry[]
```

---

## Key implementation notes

- `web-tree-sitter` is loaded dynamically to avoid eager runtime coupling
- WASM grammars are resolved via package paths rather than hard-coded filesystem locations
- Parsers are created per analysis call and explicitly deleted after use
- TSX support is handled as a special case because it shares extractor logic with TypeScript
- The plugin is designed to be extensible through extractor registration rather than subclassing

---

## Summary

`TreeSitterPlugin` is the deterministic, grammar-backed structural analysis engine in the core plugin system. It converts source files into structural metadata and call graphs, resolves imports at a basic path level, and degrades safely when a language is unsupported. Its config-driven design makes it the primary bridge between language support, AST extraction, and downstream graph analysis.
