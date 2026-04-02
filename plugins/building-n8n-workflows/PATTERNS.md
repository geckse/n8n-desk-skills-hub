# Workflow Composition Patterns

## Contents

- [Linear Chain](#linear-chain)
- [Independent Sources](#independent-sources)
- [Conditional Branching (If/Else)](#conditional-branching-ifelse)
- [Multi-Way Routing (Switch)](#multi-way-routing-switch)
- [Parallel Execution with Merge](#parallel-execution-with-merge)
- [Batch Processing](#batch-processing)
- [Multiple Triggers](#multiple-triggers)
- [Fan-In (Shared Processing)](#fan-in-shared-processing)
- [AI Agent Patterns](#ai-agent-patterns)

---

## Linear Chain

Define all nodes first, then compose with `.add()` and `.to()`:

```javascript
const startTrigger = trigger({
  type: 'n8n-nodes-base.manualTrigger',
  version: 1,
  config: { name: 'Start', position: [240, 300] },
  output: [{}]
});

const fetchData = node({
  type: 'n8n-nodes-base.httpRequest',
  version: 4.3,
  config: { name: 'Fetch Data', parameters: { method: 'GET', url: '...' }, position: [540, 300] },
  output: [{ id: 1, title: 'Item 1' }]
});

const processData = node({
  type: 'n8n-nodes-base.set',
  version: 3.4,
  config: { name: 'Process Data', parameters: {}, position: [840, 300] },
  output: [{ id: 1, title: 'Item 1', processed: true }]
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(fetchData)
  .to(processData);
```

---

## Independent Sources

When nodes return more than 1 item, chaining causes **item multiplication**: if Source A returns N items, a chained Source B runs N times instead of once.

**Fix 1 — `executeOnce: true`** (simplest):

```javascript
const sourceB = node({ ..., config: { ..., executeOnce: true } });
startTrigger.to(sourceA.to(sourceB.to(processResults)));
```

**Fix 2 — Parallel branches + Merge (combine by position):**

Pairs items by index, merging fields from both inputs into one item.

```javascript
const combineResults = merge({
  version: 3.2,
  config: { name: 'Combine Results', parameters: { mode: 'combine', combineBy: 'combineByPosition' } }
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(sourceA.to(combineResults.input(0)))
  .add(startTrigger)
  .to(sourceB.to(combineResults.input(1)))
  .add(combineResults)
  .to(processResults);
```

**Fix 3 — Parallel branches + Merge (append):**

Concatenates all items from all inputs into one list.

```javascript
const allResults = merge({
  version: 3.2,
  config: { name: 'All Results', parameters: { mode: 'append' } }
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(sourceA.to(allResults.input(0)))
  .add(startTrigger)
  .to(sourceB.to(allResults.input(1)))
  .add(allResults)
  .to(processResults);
```

---

## Conditional Branching (If/Else)

**CRITICAL:** Each branch defines a COMPLETE processing path. Chain multiple steps INSIDE the branch using `.to()`.

```javascript
const checkValid = ifElse({
  version: 2.2,
  config: { name: 'Check Valid', parameters: { ... } }
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(checkValid
    .onTrue(formatData.to(enrichData.to(saveToDb)))
    .onFalse(logError));
```

---

## Multi-Way Routing (Switch)

```javascript
const routeByPriority = switchCase({
  version: 3.2,
  config: { name: 'Route by Priority', parameters: { ... } }
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(routeByPriority
    .onCase(0, processUrgent.to(notifyTeam.to(escalate)))
    .onCase(1, processNormal)
    .onCase(2, archive));
```

---

## Parallel Execution with Merge

```javascript
const combineResults = merge({
  version: 3.2,
  config: { name: 'Combine Results', parameters: { mode: 'combine' }, position: [840, 300] }
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(branch1.to(combineResults.input(0)))
  .add(startTrigger)
  .to(branch2.to(combineResults.input(1)))
  .add(combineResults)
  .to(processResults);
```

---

## Batch Processing

```javascript
const sibNode = splitInBatches({
  version: 3,
  config: { name: 'Batch Process', parameters: { batchSize: 10 }, position: [840, 300] }
});

export default workflow('id', 'name')
  .add(startTrigger)
  .to(fetchRecords)
  .to(sibNode
    .onDone(finalizeResults)
    .onEachBatch(processRecord.to(nextBatch(sibNode)))
  );
```

---

## Multiple Triggers

Each trigger starts an independent branch:

```javascript
export default workflow('id', 'name')
  .add(webhookTrigger)
  .to(processWebhook)
  .add(scheduleTrigger)
  .to(processSchedule);
```

---

## Fan-In (Shared Processing)

Multiple triggers can feed into the same processing chain. Each trigger's execution runs in **COMPLETE ISOLATION** — never duplicate chains for "isolation."

```javascript
export default workflow('id', 'name')
  .add(webhookTrigger)
  .to(processData)
  .to(sendNotification)
  .add(scheduleTrigger)
  .to(processData);
```

---

## AI Agent Patterns

### Basic AI Agent

```javascript
const openAiModel = languageModel({
  type: '@n8n/n8n-nodes-langchain.lmChatOpenAi',
  version: 1.3,
  config: {
    name: 'OpenAI Model',
    parameters: {},
    credentials: { openAiApi: newCredential('OpenAI') },
    position: [540, 500]
  }
});

const aiAgent = node({
  type: '@n8n/n8n-nodes-langchain.agent',
  version: 3.1,
  config: {
    name: 'AI Assistant',
    parameters: { promptType: 'define', text: 'You are a helpful assistant' },
    subnodes: { model: openAiModel },
    position: [540, 300]
  },
  output: [{ output: 'AI response text' }]
});

export default workflow('ai-assistant', 'AI Assistant')
  .add(startTrigger)
  .to(aiAgent);
```

### Agent with Tools

```javascript
const calculatorTool = tool({
  type: '@n8n/n8n-nodes-langchain.toolCalculator',
  version: 1,
  config: { name: 'Calculator', parameters: {}, position: [700, 500] }
});

const aiAgent = node({
  type: '@n8n/n8n-nodes-langchain.agent',
  version: 3.1,
  config: {
    name: 'Math Agent',
    parameters: { promptType: 'define', text: 'You can use tools to help users' },
    subnodes: { model: openAiModel, tools: [calculatorTool] },
    position: [540, 300]
  },
  output: [{ output: '42' }]
});
```

### Agent with `fromAi()` (Dynamic Parameters)

Use `fromAi()` to let the AI agent fill parameter values dynamically:

```javascript
const gmailTool = tool({
  type: 'n8n-nodes-base.gmailTool',
  version: 1,
  config: {
    name: 'Gmail Tool',
    parameters: {
      sendTo: fromAi('recipient', 'Email address'),
      subject: fromAi('subject', 'Email subject'),
      message: fromAi('body', 'Email content')
    },
    credentials: { gmailOAuth2: newCredential('Gmail') },
    position: [700, 500]
  }
});

const aiAgent = node({
  type: '@n8n/n8n-nodes-langchain.agent',
  version: 3.1,
  config: {
    name: 'Email Agent',
    parameters: { promptType: 'define', text: 'You can send emails' },
    subnodes: { model: openAiModel, tools: [gmailTool] },
    position: [540, 300]
  },
  output: [{ output: 'Email sent successfully' }]
});
```

### Agent with Structured Output

```javascript
const structuredParser = outputParser({
  type: '@n8n/n8n-nodes-langchain.outputParserStructured',
  version: 1.3,
  config: {
    name: 'Structured Output Parser',
    parameters: {
      schemaType: 'fromJson',
      jsonSchemaExample: '{ "sentiment": "positive", "confidence": 0.95, "summary": "brief summary" }'
    },
    position: [700, 500]
  }
});

const aiAgent = node({
  type: '@n8n/n8n-nodes-langchain.agent',
  version: 3.1,
  config: {
    name: 'Sentiment Analyzer',
    parameters: { promptType: 'define', text: 'Analyze the sentiment of the input text', hasOutputParser: true },
    subnodes: { model: openAiModel, outputParser: structuredParser },
    position: [540, 300]
  },
  output: [{ sentiment: 'positive', confidence: 0.95, summary: 'The text expresses satisfaction' }]
});
```
