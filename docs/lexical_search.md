# lexical_search

The `lexical_search` module provides fast, fuzzy, text-based lookup over the project knowledge graph. It is the primary search layer for finding graph nodes by name, tags, summaries, and language notes, with optional filtering by node type and result limiting.

This module is intentionally lightweight: it wraps [`fuse.js`](https://fusejs.io/) around the core graph node model and exposes a small API for constructing, querying, and refreshing the search index.

## Purpose

`lexical_search` is used when the system needs **keyword-oriented retrieval** rather than semantic embedding search. Typical use cases include:

- locating nodes by partial names or abbreviations
- searching across tags and summaries
- narrowing results to specific node types
- refreshing search results after the graph changes

For semantic retrieval, see [semantic_search.md](semantic_search.md). For the graph data model used by search, see [core_schema_and_types.md](core_schema_and_types.md).

---

## Public API

### `SearchResult`

Represents a single search hit.

```ts
interface SearchResult {
  nodeId: string;
  score: number; // 0 = perfect match, 1 = worst match
}
```

#### Fields

- `nodeId`: the matched graph node identifier
- `score`: Fuse score normalized to the moduleÃƒÆ’Ã†â€™Ãƒâ€ Ã¢â‚¬â„¢ÃƒÆ’Ã¢â‚¬Å¡Ãƒâ€šÃ‚Â¢ÃƒÆ’Ã†â€™Ãƒâ€šÃ‚Â¢ÃƒÆ’Ã‚Â¢ÃƒÂ¢Ã¢â‚¬Å¡Ã‚Â¬Ãƒâ€¦Ã‚Â¡ÃƒÆ’Ã¢â‚¬Å¡Ãƒâ€šÃ‚Â¬ÃƒÆ’Ã†â€™Ãƒâ€šÃ‚Â¢ÃƒÆ’Ã‚Â¢ÃƒÂ¢Ã¢â‚¬Å¡Ã‚Â¬Ãƒâ€¦Ã‚Â¾ÃƒÆ’Ã¢â‚¬Å¡Ãƒâ€šÃ‚Â¢s result format

Lower scores indicate better matches.

### `SearchOptions`

Controls query execution.

```ts
interface SearchOptions {
  types?: GraphNode["type"][];
  limit?: number;
}
```

#### Fields

- `types`: optional allow-list of node types to include in the final results
- `limit`: maximum number of results to return; defaults to `50`

### `SearchEngine`

The main search service.

```ts
class SearchEngine {
  constructor(nodes: GraphNode[])
  search(query: string, options?: SearchOptions): SearchResult[]
  updateNodes(nodes: GraphNode[]): void
}
```

---

## Architecture

`SearchEngine` maintains an in-memory Fuse index over `GraphNode` records. The engine is rebuilt whenever the node set changes.

```mermaid
flowchart LR
  A[GraphNode input list] --> B[SearchEngine]
  B --> C[Fuse.js index]
  C --> D[search query]
  D --> E[SearchResult list]
  E --> F[Consumers / UI / graph tools]
```

### Component relationships

```mermaid
classDiagram
  class GraphNode {
    id: string
    type: NodeType
    name: string
    summary: string
    tags: string[]
    languageNotes?: string
  }

  class SearchOptions {
    types?: GraphNode.type[]
    limit?: number
  }

  class SearchResult {
    nodeId: string
    score: number
  }

  class SearchEngine {
    +SearchEngine(nodes)
    +search(query, options)
    +updateNodes(nodes)
  }

  SearchEngine --> GraphNode
  SearchEngine --> SearchOptions
  SearchEngine --> SearchResult
```

---

## Search behavior

### Indexed fields

The engine searches across these `GraphNode` fields:

- `name` with the highest weight
- `tags`
- `summary`
- `languageNotes`

```mermaid
flowchart TB
  Q[Query string] --> T[Trim whitespace]
  T --> E[Split into tokens]
  E --> O[Join with OR syntax]
  O --> F[Fuse.js search]
  F --> R[Raw ranked matches]
  R --> P[Optional type filtering]
  P --> L[Apply limit]
  L --> M[Map to SearchResult]
```

### Fuse configuration

The module uses the following Fuse settings:

- `keys`
  - `name` weight `0.4`
  - `tags` weight `0.3`
  - `summary` weight `0.2`
  - `languageNotes` weight `0.1`
- `threshold: 0.4`
- `includeScore: true`
- `ignoreLocation: true`
- `useExtendedSearch: true`

These settings favor name matches while still allowing broader discovery through tags and descriptive text.

### Extended query handling

Before searching, the query is normalized as follows:

1. leading/trailing whitespace is removed
2. the query is split on whitespace
3. tokens are joined using `|` to create an OR-style extended search expression

Example:

- input: `auth contrl`
- transformed query: `auth | contrl`

This makes the search more forgiving for partial or misspelled terms.

---

## Filtering and ranking

After Fuse returns raw matches, `SearchEngine` applies optional filtering:

- if `options.types` is provided, only nodes whose `type` is in the allow-list are kept
- results are truncated to `options.limit` or `50` by default

The final output preserves Fuse ranking order.

```mermaid
sequenceDiagram
  participant C as Caller
  participant S as SearchEngine
  participant F as Fuse.js

  C->>S: search(query, options)
  S->>S: trim + normalize query
  S->>F: search(extendedQuery)
  F-->>S: ranked matches
  S->>S: filter by types (optional)
  S->>S: slice to limit
  S-->>C: SearchResult[]
```

---

## Updating the index

`updateNodes(nodes)` replaces the current node set and rebuilds the Fuse index.

```mermaid
flowchart LR
  A[New GraphNode list] --> B[updateNodes]
  B --> C[Replace internal node cache]
  C --> D[Rebuild Fuse index]
  D --> E[SearchEngine ready for new queries]
```

This is the mechanism used when the underlying graph changes and the search index must stay in sync.

---

## Data flow

```mermaid
flowchart TD
  G[Knowledge graph nodes] --> S[SearchEngine constructor]
  S --> I[In-memory Fuse index]
  Q[User query] --> P[Query normalization]
  P --> X[Extended Fuse search]
  I --> X
  X --> Y[Optional type filter]
  Y --> Z[Limit results]
  Z --> R[SearchResult list]
```

---

## Integration with the wider system

`lexical_search` sits in the core search layer and is typically used alongside graph-building and analysis modules:

- graph construction and metadata enrichment come from [core_analysis.md](core_analysis.md)
- graph node definitions come from [core_schema_and_types.md](core_schema_and_types.md)
- semantic retrieval is handled by [semantic_search.md](semantic_search.md)

In practice, the search engine consumes the graph produced by analysis pipelines and exposes a fast lookup path for UI and tooling.

```mermaid
flowchart LR
  A[core_analysis] --> B[KnowledgeGraph and GraphNode data]
  B --> C[lexical_search]
  C --> D[SearchResult list]
  E[semantic_search] -. complementary .-> C
```

---

## Implementation notes

- The engine is **in-memory** and optimized for interactive use.
- Search is **fuzzy**, not exact.
- The module does not mutate graph nodes; it only indexes and queries them.
- `score` is passed through from Fuse and should be interpreted as a relative ranking signal.

---

## When to use this module

Use `lexical_search` when you need:

- fast keyword lookup
- partial matching on names or tags
- simple filtering by node type
- deterministic, lightweight search behavior

Prefer semantic search when the query is conceptual or natural-language oriented.

---

## Related documentation

- [core_schema_and_types.md](core_schema_and_types.md)
- [core_analysis.md](core_analysis.md)
- [semantic_search.md](semantic_search.md)
