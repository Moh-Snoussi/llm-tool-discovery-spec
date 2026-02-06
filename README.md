# Tool Discovery & Expansion Spec (v0.1)

This repository contains a minimal, scalable specification for tool discovery and expansion for large language models.
It explains how to organize a large catalogue of tools in a way that minimizes context size, improves reasoning, and scales to thousands of capabilities.

The full specification lives in [`spec/TOOL_DISCOVERY_SPEC_v0_1.md`](spec/TOOL_DISCOVERY_SPEC_v0_1.md).  Below is a short overview of what the specification covers.

## Why this spec?

Traditional LLM → tool integrations expose every tool and its full JSON schema up front.  This approach wastes tokens, slows down inference, and becomes unmanageable as the number of tools grows.

This spec introduces a hierarchical tool index, progressive disclosure of metadata, and a semantic escape hatch so that models can discover the right tool with minimal overhead.

## Key ideas

- **Hierarchical navigation:** Tools are organised by capability (e.g. Coding → Refactoring → `rename_symbol`), allowing models to reason in a coarse→fine manner.
- **Progressive disclosure:** Models first see just the top‑level categories, then tool summaries, and only fetch full schemas when necessary.
- **Semantic escape hatch:** When the hierarchical navigation is insufficient or ambiguous, the model can search for tools or categories by natural‑language intent.
- **Pointer‑based results:** Search and listing return lightweight “tool pointers” rather than full schemas to keep the context lean.

## Contents

| Path | Description |
|---|---|
| `spec/TOOL_DISCOVERY_SPEC_v0_1.md` | The full specification, including goals, concepts, API definitions, error model and behavioural rules. |
| `examples/` | Illustrative examples demonstrating how a model would navigate the hierarchy, search for tools, expand schemas and handle errors. |

This project is intended as a living document; feel free to fork it, adapt it to your own tooling architecture, and contribute improvements via pull request.
