# Tool Discovery & Expansion Spec (v0.1)

This document defines a scalable protocol for exposing large tool catalogues to language models. It introduces hierarchical navigation, progressive disclosure, and semantic search to make tool discovery efficient and auditable.

## Goals

1. **Scalable discovery:** avoid sending the full tool list or JSON schemas to the LLM up front.
2. **Progressive disclosure:** return minimal metadata until the model decides to use a tool.
3. **Navigable ontology:** organise tools in a hierarchy (tree or DAG) by capability.
4. **Semantic escape hatch:** support natural‑language search to recover from misnavigation or ambiguity.
5. **Auditability:** tool selection should be explicit and traceable (search → choose → expand → call).

## Non‑goals

The specification does **not** cover execution semantics of tools, authentication/authorization models, or serialization beyond what is needed to transport schema definitions. These are left to individual implementations.

## Concepts

### Node

A node is a category in the tool ontology. Nodes can contain child nodes and/or tool leaves.

- *Example path:* `["Coding", "Refactoring"]`
- *Example path:* `["Storage", "Buckets"]`

### Tool pointer

A pointer is a minimal reference to a tool. Pointers are returned from listing and searching operations and contain only the information necessary for the model to decide whether to expand.

### Tool schema

The full callable schema for a tool. Schemas (e.g. JSON Schema for arguments) are returned only via the `expand_tool` operation.

## API surface

The protocol consists of a handful of operations. These may be implemented as host functions, HTTP endpoints, or message channels. Names below are illustrative.

### `list`

Lists child nodes and tool pointers at a given path, optionally filtered by tags or a query.

**Request**

```json
{
  "path": ["Storage"],        // optional; empty means root
  "tags": ["auth","oauth"],   // optional
  "query": "upload file",     // optional semantic filter within this path
  "limit": 10,                // optional; default 10, max 50
  "cursor": "opaque-string"   // optional for pagination
}
```

**Response**

```json
{
  "path": ["Storage"],
  "nodes": [
    {
      "name": "Buckets",
      "path": ["Storage","Buckets"],
      "summary": "Create and manage object storage buckets",
      "tags": ["storage","objects"]
    }
  ],
  "tools": [
    {
      "tool_id": "storage.put_object",
      "path": ["Storage","Buckets"],
      "summary": "Uploads an object to a bucket",
      "tags": ["storage","upload"],
      "confidence": 0.72
    }
  ],
  "next_cursor": null
}
```

### `search_tool_by_category`

Natural‑language search scoped to a specific category. Returns tool pointers ranked by relevance.

**Request**

```json
{
  "query": "rename a symbol across the codebase",
  "category_path": ["Coding","Refactoring"],
  "limit": 5,
  "cursor": null
}
```

**Response**

```json
{
  "category_path": ["Coding","Refactoring"],
  "results": [
    {
      "tool_id": "refactor.rename_symbol",
      "path": ["Coding","Refactoring"],
      "summary": "Renames a symbol project‑wide and updates references",
      "tags": ["refactor","rename"],
      "confidence": 0.89
    }
  ],
  "next_cursor": null
}
```

### `search_nodes`

Searches for nodes when the model does not know where to start. Returns candidate category paths.

**Request**

```json
{ "query": "oauth login", "limit": 5 }
```

**Response**

```json
{
  "results": [
    {
      "path": ["User","Auth"],
      "summary": "Authentication and authorization workflows",
      "confidence": 0.83
    },
    {
      "path": ["System","Identity"],
      "summary": "Service identity and credential handling",
      "confidence": 0.62
    }
  ]
}
```

### `expand_tool`

Returns the full schema for a single tool. Only this operation provides the argument definitions that a model needs to call the tool.

**Request**

```json
{ "tool_id": "refactor.rename_symbol" }
```

**Response**

```json
{
  "tool_id": "refactor.rename_symbol",
  "path": ["Coding","Refactoring"],
  "summary": "Renames a symbol project‑wide and updates references",
  "args_schema": {
    "type": "object",
    "properties": {
      "project_root": { "type": "string" },
      "symbol": { "type": "string" },
      "new_name": { "type": "string" },
      "language": { "type": "string", "enum": ["ts","js","py","go","java"] },
      "dry_run": { "type": "boolean", "default": true }
    },
    "required": ["project_root","symbol","new_name"]
  },
  "result_schema": {
    "type": "object",
    "properties": {
      "changed_files": { "type": "integer" },
      "diff_preview": { "type": "string" }
    },
    "required": ["changed_files"]
  }
}
```

### `call_tool`

Optional; some stacks separate discovery from execution. If used, this operation invokes the tool with arguments conforming to the schema returned by `expand_tool`.

## Error model

Errors should be helpful but not leak unnecessary information. The spec defines standard error responses such as:

- **`NO_MATCH_IN_CATEGORY`**: no tools match the query within the given category. Includes hints with suggested categories.
- **`UNKNOWN_PATH`**: the requested category path does not exist. Suggests the closest root category.
- **`TOOL_NOT_FOUND`**: the requested tool ID is unknown or unavailable.
- **`NOT_AUTHORIZED`**: the model does not have access to the requested tool.

Each error includes an actionable `message`, optional `hints`, and a `next_action` indicating how the model should recover.

## Behavioural rules for models

These guidelines should be encoded in the system prompt for models interacting with the index:

1. **Do not assume tools exist.** Always discover via `list` or `search_*`.
2. **Prefer hierarchical navigation.** Use `list(path=…)` to explore before searching.
3. **Use the escape hatch when uncertain.** If no child node seems appropriate, call `search_nodes` or `search_tool_by_category`.
4. **Never call a tool without expanding it.** Always call `expand_tool` before execution to obtain the argument schema.
5. **Follow error hints.** On error, retry with suggested categories or queries.

## Example interaction

Imagine a model tasked with “rename a symbol across the codebase.” A full discovery‑execution cycle might look like:

1. `list()` → returns root nodes such as `Coding`, `Rendering`, `Storage`, `User`, `Data`.
2. `list(path=["Coding","Refactoring"])` → returns tool summaries under Refactoring.
3. `search_tool_by_category(query="rename symbol", category_path=["Coding","Refactoring"])` → returns a pointer to `refactor.rename_symbol`.
4. `expand_tool(tool_id="refactor.rename_symbol")` → returns the full JSON schema for arguments and result.
5. The model constructs arguments and passes them to the execution environment (e.g. via `call_tool`).

This pattern keeps the context small and ensures that models only see detailed schemas when they are ready to execute a tool.

## Future considerations

This specification is intentionally minimal. Real implementations can layer on:

- Embedding‑based ranking for node and tool search.
- Advanced metadata (cost, risk flags, access control).
- Versioning of tool schemas and invalidation of caches.
- DAG structures for tools that belong to multiple categories.

Feedback and improvements are welcome.