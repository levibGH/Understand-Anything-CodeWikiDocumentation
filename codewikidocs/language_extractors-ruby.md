# language_extractors-ruby

## Introduction

The `language_extractors-ruby` module provides Ruby-specific source analysis for the core language extraction pipeline. It converts Ruby tree-sitter syntax trees into the shared structural model used by the rest of the system, including functions, classes/modules, imports, exports, and call graph edges.

This module is intentionally focused on Ruby semantics that differ from other languages, such as `require`-based imports, `attr_*` macros, singleton methods (`def self.foo`), and RubyÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢s permissive top-level export model.

For the shared extractor contract and common extractor patterns, see [language_extractors-types](language_extractors-types.md). For broader language registration and selection, see [language_registries](language_registries.md).

---

## Purpose and responsibilities

`RubyExtractor` is responsible for:

- Parsing Ruby tree-sitter nodes into the shared `StructuralAnalysis` shape
- Extracting top-level and nested definitions:
  - instance methods
  - singleton methods
  - classes
  - modules
- Detecting Ruby imports via `require` and `require_relative`
- Detecting class properties declared through `attr_accessor`, `attr_reader`, and `attr_writer`
- Building intra-file call graph edges from method bodies
- Treating top-level Ruby definitions as exports, since Ruby does not have a formal export syntax

The extractor is designed to be used by the core analysis pipeline alongside other language extractors in [language_extractors](language_extractors.md).

---

## Core component

### `RubyExtractor`

`RubyExtractor` implements the shared `LanguageExtractor` interface and exposes Ruby support through:

- `languageIds = ["ruby"]`
- `extractStructure(rootNode)`
- `extractCallGraph(rootNode)`

It relies on tree-sitter node shapes and field names to identify Ruby constructs.

---

## Architecture overview

```mermaid
flowchart TD
  A[Tree-sitter Ruby AST] --> B[RubyExtractor]
  B --> C[StructuralAnalysis]
  B --> D[CallGraphEntries]
  B --> E[Imports]
  B --> F[Exports]

  C --> G[Core analysis pipeline]
  D --> G
  E --> G
  F --> G

  G --> H[GraphBuilder / normalization / downstream analysis]
```

### How it fits into the system

`RubyExtractor` is one implementation in the core language support layer. The extractor is selected through the language registry and then used by the analysis pipeline to produce normalized project knowledge.

```mermaid
flowchart LR
  P[Project source files] --> R[LanguageRegistry]
  R --> X[RubyExtractor]
  X --> S[StructuralAnalysis]
  X --> C[CallGraphEntries]
  S --> A[Core analyzers]
  C --> A
  A --> G[Knowledge graph / summaries]
```

Related modules:

- [language_registries](language_registries.md)
- [core_analysis](core_analysis.md)
- [core_schema_and_types](core_schema_and_types.md)

---

## Data model produced by the extractor

### Structural analysis output

`extractStructure()` returns a `StructuralAnalysis` object with these Ruby-populated sections:

- `functions`: all discovered methods and singleton methods
- `classes`: Ruby classes and modules
- `imports`: top-level `require` / `require_relative` statements
- `exports`: top-level definitions treated as exported symbols

```mermaid
classDiagram
  class StructuralAnalysis {
    functions
    classes
    imports
    exports
  }

  class FunctionInfo {
    name
    lineRange
    params
  }

  class ClassInfo {
    name
    lineRange
    methods
    properties
  }

  class ImportInfo {
    source
    specifiers
    lineNumber
  }

  class ExportInfo {
    name
    lineNumber
  }

  StructuralAnalysis --> FunctionInfo
  StructuralAnalysis --> ClassInfo
  StructuralAnalysis --> ImportInfo
  StructuralAnalysis --> ExportInfo
```

### Call graph output

`extractCallGraph()` returns `CallGraphEntry[]`, where each entry records:

- `caller`: current method context
- `callee`: invoked method or receiver-qualified call
- `lineNumber`: source line of the call

---

## Extraction behavior

### 1) Top-level definitions

The extractor scans only the root nodeÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢s direct children for structural extraction.

It recognizes these top-level node types:

- `method`
- `singleton_method`
- `class`
- `module`
- `call`

#### Methods

For `method` nodes:

- the method name is read from the `name` field
- parameters are extracted from the `parameters` field
- the method is added to `functions`
- the method is also added to `exports`

#### Singleton methods

For `singleton_method` nodes:

- the method name is prefixed with `self.`
- parameters are extracted the same way as regular methods
- the method is added to `functions`
- the method is also added to `exports`

#### Classes and modules

For both `class` and `module` nodes:

- the name is read from the `name` field
- the body is scanned for methods and `attr_*` macros
- the node is added to `classes`
- the class/module name is added to `exports`

Ruby modules are intentionally represented in the same `classes` collection because they behave like named containers for methods and constants in the shared model.

```mermaid
flowchart TD
  A[root children] --> B{node type}
  B -->|method| C[extractMethod]
  B -->|singleton_method| D[extractSingletonMethod]
  B -->|class| E[extractClass]
  B -->|module| F[extractModule]
  B -->|call| G[extractTopLevelCall]

  C --> H[functions + exports]
  D --> H
  E --> I[classes + nested methods/properties + exports]
  F --> I
  G --> J[imports]
```

---

### 2) Parameter extraction

`extractParams()` handles Ruby method parameter node variants:

- `identifier` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ plain parameter name
- `optional_parameter` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ parameter name without default value
- `splat_parameter` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ prefixed with `*`
- `hash_splat_parameter` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ prefixed with `**`
- `block_parameter` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ prefixed with `&`

This preserves RubyÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢s parameter semantics in a compact string form.

```mermaid
flowchart TD
  A[method_parameters node] --> B{child type}
  B -->|identifier| C[param]
  B -->|optional_parameter| D[name]
  B -->|splat_parameter| E[*name]
  B -->|hash_splat_parameter| F[**name]
  B -->|block_parameter| G[&name]
```

---

### 3) Class and module body scanning

`extractClassBody()` scans the body of a class or module and collects:

- nested `method` definitions
- nested `singleton_method` definitions
- `attr_accessor`, `attr_reader`, `attr_writer` calls

Methods discovered inside the body are added both to:

- the enclosing class/moduleÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢s `methods` list
- the global `functions` list

Properties are derived from symbol arguments passed to `attr_*` macros.

```mermaid
flowchart TD
  A[class/module body] --> B{member type}
  B -->|method| C[add method name]
  B -->|singleton_method| D[add self.method name]
  B -->|call attr_*| E[extract symbol args as properties]
  C --> F[functions]
  D --> F
  E --> G[properties]
```

#### `attr_*` property extraction

The extractor recognizes:

- `attr_accessor`
- `attr_reader`
- `attr_writer`

It reads `simple_symbol` arguments such as `:name` and stores them as `name`.

---

### 4) Import extraction

Ruby imports are handled specially because they are expressed as method calls rather than language-level import statements.

The extractor treats top-level calls to:

- `require`
- `require_relative`

as imports.

The first string argument is used as the import source.

```mermaid
sequenceDiagram
  participant AST as Ruby AST
  participant RX as RubyExtractor
  participant OUT as imports[]

  AST->>RX: top-level call(require "foo")
  RX->>RX: detect method name
  RX->>RX: read first string argument
  RX->>OUT: push { source: "foo", specifiers: ["foo"] }
```

#### String handling

`getStringContent()` prefers the `string_content` child node and falls back to stripping surrounding quotes if needed.

---

### 5) Call graph extraction

`extractCallGraph()` walks the full AST and tracks the current method context using a stack.

It records call edges for:

- regular `call` nodes inside methods
- bare identifier calls inside `body_statement` nodes, which Ruby often uses for zero-argument method calls

It skips:

- `require` / `require_relative`
- `attr_accessor` / `attr_reader` / `attr_writer`

```mermaid
flowchart TD
  A[walk AST] --> B{enter method?}
  B -->|yes| C[push caller name]
  B -->|no| D[continue]

  D --> E{node type}
  E -->|call| F{import or attr_*?}
  F -->|no| G[emit CallGraphEntry]
  F -->|yes| H[skip]
  E -->|identifier in body_statement| I[emit CallGraphEntry]
  E -->|other| J[recurse]

  G --> K[entries]
  I --> K
  C --> J
  J --> L{leave method?}
  L -->|yes| M[pop caller name]
```

#### Caller naming rules

- regular methods use their method name
- singleton methods use `self.<name>`

#### Callee naming rules

- receiver-qualified calls become `receiver.method`
- unqualified calls remain as the method name

---

## Dependency map

`RubyExtractor` depends on a small set of shared core abstractions and helper utilities.

```mermaid
flowchart LR
  RX[RubyExtractor] --> T[types.ts
StructuralAnalysis
CallGraphEntry
LanguageExtractor]
  RX --> B[base-extractor.ts
findChild]
  RX --> A[Tree-sitter Ruby AST nodes]
  RX --> L[Language registry / plugin system]
```

### Direct dependencies

- `../../types.js`
  - `StructuralAnalysis`
  - `CallGraphEntry`
- `./types.js`
  - `LanguageExtractor`
  - `TreeSitterNode`
- `./base-extractor.js`
  - `findChild()`

### Indirect system dependencies

- [core_plugin_system](core_plugin_system.md) for extractor registration and discovery
- [core_language_support](core_language_support.md) for language selection and extractor orchestration
- [core_schema_and_types](core_schema_and_types.md) for shared graph and analysis types

---

## Ruby-specific design decisions

### Modules are treated like classes

Ruby modules are stored in the same `classes` collection as classes. This keeps the shared structural model simple while still preserving named containers with methods and properties.

### Top-level definitions are exported

Because Ruby lacks explicit export syntax, the extractor marks top-level classes, modules, methods, and singleton methods as exports.

### Singleton methods are namespaced with `self.`

This makes class-level methods distinguishable from instance methods in downstream analysis.

### Bare identifier calls are treated as call graph edges

Ruby often parses zero-argument method invocations as identifiers inside method bodies. The extractor explicitly converts these into call graph entries when inside a function context.

---

## Process flow summary

```mermaid
flowchart TD
  A[Parse Ruby source with tree-sitter] --> B[Build AST]
  B --> C[RubyExtractor.extractStructure]
  B --> D[RubyExtractor.extractCallGraph]
  C --> E[functions/classes/imports/exports]
  D --> F[call graph entries]
  E --> G[Core analysis pipeline]
  F --> G
  G --> H[Graph building / normalization / downstream consumers]
```

---

## Integration notes

### When to use this module

Use `RubyExtractor` when analyzing Ruby source files in the core analysis pipeline. It is the language-specific adapter that translates Ruby syntax into the shared analysis model.

### What it does not do

This module does not:

- resolve imports to filesystem paths
- normalize graph edges
- perform semantic LLM analysis
- validate the final graph schema

Those responsibilities belong to other core modules such as [core_analysis](core_analysis.md), [core_change_tracking](core_change_tracking.md), and [core_schema_and_types](core_schema_and_types.md).

---

## Related documentation

- [language_extractors](language_extractors.md)
- [language_extractors-types](language_extractors-types.md)
- [language_registries](language_registries.md)
- [core_language_support](core_language_support.md)
- [core_analysis](core_analysis.md)
- [core_schema_and_types](core_schema_and_types.md)
