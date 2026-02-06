# Simple Example: Renaming a Symbol

This example illustrates how a language model might use the tool discovery protocol to find and execute a symbol‑renaming tool.

### User query

> Rename the `calculate` function across my project.

### Model steps

1. **List root categories**
   
   The model calls `list()` with no arguments and receives high‑level categories:

   ```json
   { "path": [], "nodes": ["Coding","Rendering","Storage","Data","User"], "tools": [] }
   ```

2. **Navigate to the “Refactoring” node**

   The model decides that renaming code is a refactoring task and drills down:

   ```json
   { "path": ["Coding","Refactoring"] }
   ```

   The response includes summaries of refactoring tools, such as `inline_function`, `rename_symbol`, etc.

3. **Search within the category**

   To be sure, the model performs a semantic search within this node:

   ```json
   {
     "query": "rename function across project",
     "category_path": ["Coding","Refactoring"],
     "limit": 5
   }
   ```

   It receives a high‑confidence pointer to `refactor.rename_symbol`.

4. **Expand the tool**

   The model calls `expand_tool("refactor.rename_symbol")` to retrieve the full argument schema.

5. **Execute the tool**

   After constructing the arguments, the model invokes the tool (via `call_tool`, if applicable) to perform the refactoring.

This pattern demonstrates progressive disclosure (categories → tool pointer → full schema) and the use of semantic search as an escape hatch.