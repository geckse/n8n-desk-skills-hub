---
name: building-n8n-workflows
description: >-
  Builds, validates, and manages n8n workflows programmatically using the
  @n8n/workflow-sdk and the n8n MCP server. Covers workflow creation, node
  discovery, AI agent workflows, expression syntax, and the validate-fix loop.
  Use when the user asks to create, update, or fix an n8n workflow, or mentions
  n8n, workflow automation, or workflow SDK.
---

# Building n8n Workflows

## SDK Import

```javascript
import {
  workflow, node, trigger, sticky, placeholder, newCredential,
  ifElse, switchCase, merge, splitInBatches, nextBatch,
  languageModel, memory, tool, outputParser, embedding, embeddings,
  vectorStore, retriever, documentLoader, textSplitter, reranker,
  fromAi, expr
} from '@n8n/workflow-sdk';
```

## Workflow Building Process

Follow these steps in order. Do not skip steps.

### Step 1: Read SDK reference

Call the MCP resource `n8n://workflow-sdk/reference` or read [PATTERNS.md](PATTERNS.md) for composition patterns.

### Step 2: Discover nodes

Call `claude.ai n8n beta:search_nodes` with queries for services needed (e.g., `["gmail", "slack", "schedule trigger"]`) and utility nodes (e.g., `["set", "if", "merge", "code"]`). Note the **discriminators** (resource/operation/mode) in the results.

Before searching, check [SUGGESTED-NODES.md](SUGGESTED-NODES.md) for the right category to understand which nodes and patterns fit your use case.

### Step 3: (Optional) Get suggestions

Call `claude.ai n8n beta:get_suggested_nodes` with workflow technique categories for curated recommendations.

### Step 4: Get type definitions

Call `claude.ai n8n beta:get_node_types` with ALL node IDs you plan to use, including discriminators from search results. This returns exact TypeScript parameter definitions.

**DO NOT SKIP THIS.** Guessing parameter names creates invalid workflows.

### Step 5: Write workflow code

Use the SDK patterns from [PATTERNS.md](PATTERNS.md) and exact parameter names from the type definitions. Follow the Coding Guidelines and Expression System rules below.

### Step 6: Validate (feedback loop)

Call `claude.ai n8n beta:validate_workflow` with your full code.

**If validation fails:** Fix errors and re-validate. Repeat until valid. Do not proceed with invalid code.

### Step 7: Create

Call `claude.ai n8n beta:create_workflow_from_code` with validated code. Include a short `description` (1-2 sentences) summarizing what the workflow does.

### Step 8: Update (existing workflows)

Call `claude.ai n8n beta:update_workflow` with workflow ID and validated code. Always re-run steps 2-6 for the new code.

### Step 9: Lifecycle management

- `claude.ai n8n beta:archive_workflow` — archive by ID
- `claude.ai n8n beta:publish_workflow` — activate
- `claude.ai n8n beta:unpublish_workflow` — deactivate

---

## Coding Guidelines

These rules are mandatory. Each prevents a specific class of error.

1. **Always use `newCredential()` for authentication.** NEVER use placeholder strings, fake API keys, or hardcoded auth values.
   ```javascript
   credentials: { slackApi: newCredential('Slack Bot') }
   ```

2. **Use exact parameter names and structures** from the type definitions returned by `get_node_types`.

3. **Use unique variable names.** Never reuse builder function names as variable names.
   ```javascript
   // WRONG: shadows the `node` import
   const node = node({ ... });
   // CORRECT
   const fetchData = node({ ... });
   ```

4. **Use descriptive node names.** Good: "Fetch Weather Data", "Check Temperature". Bad: "HTTP Request", "Set", "If".

5. **Credential types must match** what the node expects:
   ```javascript
   credentials: { slackApi: newCredential('Slack Bot') }
   ```

6. **Expressions: use `expr()` with single or double quotes, NEVER backtick template literals.**
   ```javascript
   // WRONG
   expr(`Daily Digest - ${$now.toFormat('MMMM d')}`)
   // WRONG: $now outside {{ }}
   expr('Daily Digest - ' + $now.toFormat('MMMM d'))
   // CORRECT
   expr('Daily Digest - {{ $now.toFormat("MMMM d") }}\n{{ $json.output }}')
   ```
   For multiline: use string concatenation `expr('Line 1\n' + 'Line 2 {{ $json.value }}')`

7. **Placeholders: use `placeholder('hint')` directly as the parameter value.** Never wrap in `expr()`, objects, or arrays.
   ```javascript
   // WRONG
   parameters: { url: expr(placeholder('Your API URL')) }
   // CORRECT
   parameters: { url: placeholder('Your API URL (e.g. https://api.example.com/v1)') }
   ```

8. **Every node MUST have an `output` property** with sample data. Following nodes depend on it for expressions.

9. **String quoting:** When a string contains an apostrophe, use double quotes.
   ```javascript
   output: [{ text: "I've arrived" }]
   ```

10. **Do NOT add or edit comments.** Comments are ignored and not shared with user. Use `sticky(...)` to provide guidance on the canvas.

---

## Expression System

### Available variables inside `expr('{{ ... }}')`

| Variable | Description |
|---|---|
| `$json` | Current item's JSON data from immediate predecessor |
| `$('NodeName').item.json` | Access any node's output by name |
| `$input.first()` | First item from immediate predecessor |
| `$input.all()` | All items from immediate predecessor |
| `$input.item` | Current item being processed |
| `$binary` | Binary data of current item |
| `$now` | Current date/time (Luxon DateTime) |
| `$today` | Start of today (Luxon DateTime) |
| `$itemIndex` | Index of current item |
| `$runIndex` | Current run index |
| `$execution.id` | Unique execution ID |
| `$execution.mode` | `'test'` or `'production'` |
| `$workflow.id` / `$workflow.name` | Workflow metadata |

### Webhook data structure (critical gotcha)

Webhook data is nested under `$json.body`, NOT at the root. This is the #1 expression mistake.

```javascript
// WRONG — fields are NOT at root
expr('{{ $json.email }}')

// CORRECT — webhook payload is under .body
expr('{{ $json.body.email }}')

// Full webhook structure: { body: {...}, headers: {...}, params: {...}, query: {...} }
```

When referencing webhook data from downstream nodes:
```javascript
expr('{{ $("Webhook").item.json.body.email }}')
```

### String composition rules

Variables MUST always be inside `{{ }}`, never outside as JS variables:

```javascript
expr('Hello {{ $json.name }}, welcome!')
expr('Report for {{ $now.toFormat("MMMM d, yyyy") }} - {{ $json.title }}')
expr('{{ $json.firstName }} {{ $json.lastName }}')
expr('Status: {{ $json.count > 0 ? "active" : "empty" }}')
```

### Dynamic data from other nodes

`$()` MUST always be inside `{{ }}`, never used as plain JavaScript:

```javascript
// WRONG: $() outside {{ }}
expr('{{ ' + JSON.stringify($('Source').all().map(i => i.json.name)) + ' }}')

// CORRECT: $() inside {{ }} — evaluated at runtime
expr('{{ $("Source").all().map(i => ({ option: i.json.name })) }}')
expr('{{ { "fields": [{ "values": $("Fetch Projects").all().map(i => ({ option: i.json.name })) }] } }}')
```

---

## Design Guidance

- **Trace item counts:** For each connection A -> B, if A returns N items, should B run N times or once? If B doesn't need A's items, set `executeOnce: true` on B or use parallel branches + Merge.
- **Handling convergence after branches:** When a node receives data from multiple paths (after Switch, IF, Merge), use optional chaining `expr('{{ $json.data?.approved ?? $json.status }}')`, reference a node that ALWAYS runs `expr("{{ $('Webhook').item.json.field }}")`, or normalize with Set nodes before convergence.
- **Prefer dedicated integration nodes** over HTTP Request when `search_nodes` shows one is available.
- **Pay attention to `@builderHint` annotations** in type definitions — they provide critical guidance on how to correctly configure node parameters.

---

## SDK Functions Quick Reference

| Function | Description |
|---|---|
| `placeholder('hint')` | Marks a value for user input. Use directly as parameter value. |
| `sticky('content', nodes?, config?)` | Creates a sticky note on the canvas. |
| `.output(n)` | Selects a specific output index for multi-output nodes. |
| `.onError(handler)` | Connects error output. Requires `onError: 'continueErrorOutput'` in config. |
| `newCredential('Name')` | Creates a credential reference. Type must match node expectation. |
| `fromAi('key', 'description')` | Marks a parameter to be filled by AI agent dynamically. |
| `expr('...')` | Wraps n8n expression syntax. Single/double quotes only, no backticks. |

**Subnode factories** (same pattern as `languageModel()` and `tool()`):
`memory()`, `outputParser()`, `embeddings()`, `vectorStore()`, `retriever()`, `documentLoader()`, `textSplitter()`, `reranker()`

---

## MCP Tools Reference

| Tool | Description |
|---|---|
| `claude.ai n8n beta:search_nodes` | Search nodes by service name, trigger type, or utility |
| `claude.ai n8n beta:get_node_types` | Get TypeScript type definitions for nodes |
| `claude.ai n8n beta:get_suggested_nodes` | Get curated node recommendations by category |
| `claude.ai n8n beta:validate_workflow` | Validate workflow code before creating |
| `claude.ai n8n beta:create_workflow_from_code` | Save validated workflow to n8n |
| `claude.ai n8n beta:update_workflow` | Update existing workflow with new code |
| `claude.ai n8n beta:get_workflow_details` | Get workflow details by ID |
| `claude.ai n8n beta:search_workflows` | Search workflows in the instance |
| `claude.ai n8n beta:execute_workflow` | Execute a workflow |
| `claude.ai n8n beta:get_execution` | Get execution details |
| `claude.ai n8n beta:archive_workflow` | Archive a workflow |
| `claude.ai n8n beta:publish_workflow` | Activate a workflow |
| `claude.ai n8n beta:unpublish_workflow` | Deactivate a workflow |

---

## Reference Files

Load these as needed:

- **Workflow composition patterns** (linear, branching, parallel, batch, AI agents): [PATTERNS.md](PATTERNS.md)
- **Choosing the right nodes** before calling `search_nodes`: [SUGGESTED-NODES.md](SUGGESTED-NODES.md)
- **SDK factory function signatures, interfaces, and type details**: [SDK-API.md](SDK-API.md)
