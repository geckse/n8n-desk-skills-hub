# SDK API Reference

Condensed reference for `@n8n/workflow-sdk`. Covers the public API surface used when building workflows.

## Contents

- [Core Factory Functions](#core-factory-functions)
- [Subnode Factory Functions](#subnode-factory-functions)
- [Expression Utilities](#expression-utilities)
- [Code Helpers](#code-helpers)
- [Validation](#validation)
- [Code Generation](#code-generation)
- [WorkflowBuilder Interface](#workflowbuilder-interface)
- [NodeInstance Interface](#nodeinstance-interface)
- [Branching Interfaces](#branching-interfaces)
- [Node Configuration](#node-configuration)
- [Data Types](#data-types)
- [Node Type Constants](#node-type-constants)
- [Convenience Type Aliases](#convenience-type-aliases)

---

## Core Factory Functions

| Name | Signature | Description |
|---|---|---|
| `workflow` | `(id: string, name: string, options?: WorkflowSettings \| WorkflowBuilderOptions) => WorkflowBuilder` | Create a new workflow builder |
| `workflow.fromJSON` | `(json: WorkflowJSON) => WorkflowBuilder` | Create from existing JSON |
| `node` | `(input: NodeInput) => NodeInstance` | Create a regular node |
| `trigger` | `(input: TriggerInput) => TriggerInstance` | Create a trigger node |
| `sticky` | `(content: string, nodesOrConfig?, config?) => NodeInstance` | Create a sticky note. Optionally auto-positions around given nodes |
| `placeholder` | `(hint: string) => PlaceholderValue` | Create a placeholder for user input |
| `newCredential` | `(name: string) => NewCredentialValue` | Create a new credential marker |
| `ifElse` | `(input: { version, config? }) => NodeInstance` | Create IF node. Use `.onTrue()`/`.onFalse()` |
| `switchCase` | `(input: { version, config? }) => NodeInstance` | Create Switch node. Use `.onCase()` |
| `merge` | `(input: { version, config? }) => NodeInstance` | Create Merge node. Use `.input(n)` |
| `splitInBatches` | `(input: { version, config? }) => SplitInBatchesBuilder` | Create SIB node. Use `.onEachBatch()`/`.onDone()` |
| `nextBatch` | `(sib: NodeInstance \| SplitInBatchesBuilder) => NodeInstance` | Loop-back helper for SIB |

---

## Subnode Factory Functions

All follow the same pattern: `(input: NodeInput) => SubnodeInstance`

| Name | Subnode Type | Description |
|---|---|---|
| `languageModel` | `ai_languageModel` | Language model (OpenAI, Anthropic, etc.) |
| `memory` | `ai_memory` | Memory (Buffer Window, etc.) |
| `tool` | `ai_tool` | Tool (Calculator, Code, service tools) |
| `outputParser` | `ai_outputParser` | Output parser (Structured, Auto-fixing) |
| `embedding` / `embeddings` | `ai_embedding` | Embedding model |
| `vectorStore` | `ai_vectorStore` | Vector store (Pinecone, Qdrant, etc.) |
| `retriever` | `ai_retriever` | Retriever |
| `documentLoader` | `ai_document` | Document loader |
| `textSplitter` | `ai_textSplitter` | Text splitter |
| `reranker` | `ai_reranker` | Reranker (not currently exported from main) |
| `fromAi` | — | `(key, description?, type?, defaultValue?) => string` — creates `$fromAI` expression |

---

## Expression Utilities

| Name | Signature | Description |
|---|---|---|
| `expr` | `(expression: string) => string` | Mark string as n8n expression (adds `=` prefix) |
| `serializeExpression` | `(fn: Expression) => string` | Serialize expression function to n8n string |
| `parseExpression` | `(expr: string) => string` | Extract inner expression (strips `={{ }}`) |
| `isExpression` | `(value: unknown) => boolean` | Check if string is an n8n expression |
| `createFromAIExpression` | `(key, description?, type?, defaultValue?) => string` | Create `$fromAI` expression with marker |

---

## Code Helpers

| Name | Signature | Description |
|---|---|---|
| `runOnceForAllItems` | `(fn: (ctx: AllItemsContext) => Array<{ json: T }>) => CodeResult` | Code that runs once with all items |
| `runOnceForEachItem` | `(fn: (ctx: EachItemContext) => { json: T } \| null) => CodeResult` | Code that runs once per item |

---

## Validation

| Name | Description |
|---|---|
| `validateWorkflow(workflow, options?) => ValidationResult` | Validate workflow for errors and warnings |
| `ValidationResult` | `{ valid: boolean; errors: ValidationError[]; warnings: ValidationWarning[] }` |
| `ValidationError` | `{ code: ValidationErrorCode; message; nodeName?; parameterName? }` |
| `ValidationWarning` | `{ code: ValidationErrorCode; message; nodeName?; parameterPath? }` |

**ValidationErrorCode values:** `NO_NODES`, `MISSING_TRIGGER`, `DISCONNECTED_NODE`, `MISSING_PARAMETER`, `INVALID_CONNECTION`, `CIRCULAR_REFERENCE`, `INVALID_EXPRESSION`, `AGENT_STATIC_PROMPT`, `AGENT_NO_SYSTEM_MESSAGE`, `HARDCODED_CREDENTIALS`, `SET_CREDENTIAL_FIELD`, `MERGE_SINGLE_INPUT`, `TOOL_NO_PARAMETERS`, `FROM_AI_IN_NON_TOOL`, `MISSING_EXPRESSION_PREFIX`, `INVALID_PARAMETER`, `INVALID_INPUT_INDEX`, `SUBNODE_NOT_CONNECTED`, `SUBNODE_PARAMETER_MISMATCH`, `UNSUPPORTED_SUBNODE_INPUT`, `MAX_NODES_EXCEEDED`, `INVALID_EXPRESSION_PATH`, `PARTIAL_EXPRESSION_PATH`, `INVALID_DATE_METHOD`

---

## Code Generation

| Name | Description |
|---|---|
| `generateWorkflowCode(input: WorkflowJSON \| GenerateWorkflowCodeOptions) => string` | Generate SDK code from workflow JSON |
| `parseWorkflowCode(code: string) => WorkflowJSON` | Parse SDK code back to JSON (AST interpreter) |
| `parseWorkflowCodeToBuilder(code: string) => WorkflowBuilder` | Parse SDK code to builder (allows validation) |

---

## WorkflowBuilder Interface

| Method | Description |
|---|---|
| `add(node)` | Add a node, chain, or composite to the workflow |
| `to(target)` | Connect current node to target |
| `settings(settings)` | Set workflow-level settings |
| `connect(source, sourceOutput, target, targetInput)` | Explicit connection with indices |
| `getNode(name) => NodeInstance \| undefined` | Get node by name |
| `validate(options?) => ValidationResult` | Validate workflow |
| `toJSON() => WorkflowJSON` | Convert to n8n JSON |
| `generatePinData(options?)` | Generate pin data from node outputs |

---

## NodeInstance Interface

| Property / Method | Description |
|---|---|
| `type` | Node type identifier (e.g., `'n8n-nodes-base.httpRequest'`) |
| `version` | Node version string |
| `config` | Node configuration |
| `to(target, outputIndex?)` | Connect to target node, chain, or input target |
| `input(index) => InputTarget` | Create input target (for Merge nodes) |
| `output(index) => OutputSelector` | Select specific output index |
| `onTrue(target) => IfElseBuilder` | IF nodes: set true branch |
| `onFalse(target) => IfElseBuilder` | IF nodes: set false branch |
| `onCase(index, target) => SwitchCaseBuilder` | Switch nodes: set case target |
| `onError(handler)` | Set error handler node |
| `update(config) => NodeInstance` | Create new instance with updated config |

`TriggerInstance` extends `NodeInstance` with `isTrigger: true`.

`NodeChain` extends `NodeInstance` with `head`, `tail`, `allNodes`, and chainable `.to()`.

---

## Branching Interfaces

**IfElseBuilder:**

| Method | Description |
|---|---|
| `onTrue(target)` | Set true branch (output 0) |
| `onFalse(target)` | Set false branch (output 1) |

**SwitchCaseBuilder:**

| Method | Description |
|---|---|
| `onCase(index, target)` | Set case target (0-based output index) |

**SplitInBatchesBuilder:**

| Method | Description |
|---|---|
| `onEachBatch(target)` | Set batch processing target (output 1) |
| `onDone(target)` | Set completion target (output 0) |

---

## Node Configuration

**NodeConfig:**
`parameters?`, `credentials?`, `name?`, `position?`, `disabled?`, `notes?`, `executeOnce?`, `retryOnFail?`, `alwaysOutputData?`, `onError?`, `pinData?`, `output?`, `subnodes?`

**SubnodeConfig:**
`model?`, `memory?`, `tools?`, `outputParser?`, `embedding?`, `vectorStore?`, `retriever?`, `documentLoader?`, `textSplitter?`, `reranker?`

**StickyNoteConfig:**
`color?`, `position?`, `width?`, `height?`, `name?`

**Credentials & Placeholders:**

| Type | Properties |
|---|---|
| `CredentialReference` | `name: string; id: string` |
| `NewCredentialValue` | `__newCredential: true; name: string` |
| `PlaceholderValue` | `__placeholder: true; hint: string` |

---

## Data Types

| Type | Properties |
|---|---|
| `WorkflowJSON` | `id?, name, nodes: NodeJSON[], connections, settings?, pinData?, meta?` |
| `NodeJSON` | `id, name?, type, typeVersion, position, parameters?, credentials?, disabled?, executeOnce?, onError?` |
| `WorkflowSettings` | `timezone?, errorWorkflow?, executionTimeout?, executionOrder?, callerPolicy?` |
| `IDataObject` | `Record<string, string \| number \| boolean \| null \| undefined \| object \| IDataObject \| Array>` |

---

## Node Type Constants

| Constant | Value |
|---|---|
| `NODE_TYPES.IF` | `'n8n-nodes-base.if'` |
| `NODE_TYPES.SWITCH` | `'n8n-nodes-base.switch'` |
| `NODE_TYPES.MERGE` | `'n8n-nodes-base.merge'` |
| `NODE_TYPES.STICKY_NOTE` | `'n8n-nodes-base.stickyNote'` |
| `NODE_TYPES.SPLIT_IN_BATCHES` | `'n8n-nodes-base.splitInBatches'` |
| `NODE_TYPES.HTTP_REQUEST` | `'n8n-nodes-base.httpRequest'` |
| `NODE_TYPES.WEBHOOK` | `'n8n-nodes-base.webhook'` |
| `NODE_TYPES.DATA_TABLE` | `'n8n-nodes-base.dataTable'` |

---

## Convenience Type Aliases

| Type | Definition |
|---|---|
| `AnyNode` | `NodeInstance<string, string, unknown>` |
| `AnyChain` | `NodeChain<AnyNode, AnyNode>` |
| `AnyTrigger` | `TriggerInstance<string, string, unknown>` |
| `OnError` | `'stopWorkflow' \| 'continueRegularOutput' \| 'continueErrorOutput'` |
| `Expression<T>` | `($: ExpressionContext) => T` |
| `FromAIArgumentType` | `'string' \| 'number' \| 'boolean' \| 'json'` |
| `CompositeType` | `'ifElse' \| 'switchCase' \| 'merge' \| 'splitInBatches'` |
