---
name: writing-n8n-code
description: >-
  Writes JavaScript and Python code for n8n Code nodes. Covers data access
  patterns ($input, $json, $node), return format requirements, built-in
  helpers ($helpers.httpRequest, DateTime, $jmespath), webhook data structure,
  and common error prevention. Use when writing code inside n8n Code nodes
  or when the user mentions $input, $json, Code node, or custom
  JavaScript/Python in workflows.
---

# Writing Code in n8n Code Nodes

## Quick Start

```javascript
// Run Once for All Items mode (default, recommended)
const items = $input.all();
const results = [];

for (const item of items) {
  results.push({
    json: {
      name: item.json.name,
      processed: true
    }
  });
}

return results;
```

**Essential rules:**
1. MUST return `[{json: {...}}, ...]` — array of objects with `json` key
2. Use `$input.all()` for batch, `$input.first()` for single item
3. Webhook data is under `$json.body`, NOT at root
4. No `{{ }}` expressions — use plain JavaScript
5. `$helpers.httpRequest()` for HTTP calls (not fetch/axios)

---

## Mode Selection

### Run Once for All Items (recommended, 95% of cases)

Processes all items in a single execution. Use for: filtering, aggregation, transformation, combining data, generating summaries.

```javascript
const items = $input.all();
return items.filter(item => item.json.status === 'active')
  .map(item => ({ json: { name: item.json.name } }));
```

### Run Once for Each Item (specialized)

Runs your code once per item. Use when: each item needs independent processing, you need `$input.item`, or items require different error handling.

```javascript
// $input.item is only available in this mode
const data = $input.item.json;
return [{ json: { processed: data.name.toUpperCase() } }];
```

---

## Data Access Patterns

### `$input.all()` — All items (most common)

```javascript
const items = $input.all();
// items = [{json: {name: "Alice"}}, {json: {name: "Bob"}}]

// Filter
const active = items.filter(i => i.json.active);

// Transform
const names = items.map(i => ({ json: { name: i.json.name.toUpperCase() } }));

// Aggregate
const total = items.reduce((sum, i) => sum + i.json.amount, 0);
return [{ json: { total } }];
```

### `$input.first()` — Single item

```javascript
const item = $input.first();
const data = item.json;
return [{ json: { id: data.id, name: data.name } }];
```

### `$input.item` — Current item (Each Item mode only)

```javascript
const data = $input.item.json;
return [{ json: { result: data.value * 2 } }];
```

### `$node["NodeName"]` — Reference other nodes

```javascript
// Access output of a specific node by name
const webhookData = $node["Webhook"].json;
const apiResult = $node["HTTP Request"].json;

// Use bracket notation for names with spaces
const data = $node["My Custom Node"].json;
```

---

## Webhook Data Structure

Webhook data is nested — the #1 mistake in Code nodes.

```javascript
// WRONG — fields are NOT at root
const email = $input.first().json.email;

// CORRECT — payload is under .body
const email = $input.first().json.body.email;

// Full webhook structure:
// {
//   body: { ...posted data... },
//   headers: { content-type: "...", ... },
//   params: { ...path params... },
//   query: { ...query string... }
// }

// Access query parameters
const page = $input.first().json.query.page;

// Access headers
const auth = $input.first().json.headers.authorization;
```

---

## Return Format

MUST return an array of objects with `json` key. Everything else causes errors.

```javascript
// CORRECT — array of {json: ...} objects
return [{ json: { name: "Alice" } }];
return items.map(i => ({ json: { ...i.json, processed: true } }));

// WRONG — missing json wrapper
return [{ name: "Alice" }];

// WRONG — not an array
return { json: { name: "Alice" } };

// WRONG — empty return
// (no return statement)

// Returning nothing (skip item in Each Item mode)
return [];
```

---

## Built-in Helpers

### `$helpers.httpRequest()` — HTTP calls

Use this instead of fetch/axios (which are not available).

```javascript
// GET request
const response = await $helpers.httpRequest({
  method: 'GET',
  url: 'https://api.example.com/data',
  headers: { 'Authorization': 'Bearer TOKEN' }
});
return [{ json: response }];

// POST request
const result = await $helpers.httpRequest({
  method: 'POST',
  url: 'https://api.example.com/submit',
  body: { name: 'Alice', email: 'alice@example.com' },
  headers: { 'Content-Type': 'application/json' }
});
return [{ json: result }];

// With error handling
try {
  const data = await $helpers.httpRequest({ method: 'GET', url: '...' });
  return [{ json: data }];
} catch (error) {
  return [{ json: { error: error.message } }];
}
```

### `DateTime` — Date/time operations (Luxon)

```javascript
// Current date/time
const now = DateTime.now();
const today = DateTime.now().startOf('day');

// Formatting
const formatted = DateTime.now().toFormat('yyyy-MM-dd');
const iso = DateTime.now().toISO();
const readable = DateTime.now().toFormat('MMMM d, yyyy');

// Parsing
const date = DateTime.fromISO('2024-01-15');
const fromFormat = DateTime.fromFormat('15/01/2024', 'dd/MM/yyyy');

// Arithmetic
const tomorrow = DateTime.now().plus({ days: 1 });
const lastWeek = DateTime.now().minus({ weeks: 1 });

// Comparison
const isAfter = date1 > date2;
const diff = date1.diff(date2, 'days').days;

// Timezone
const utc = DateTime.now().toUTC();
const berlin = DateTime.now().setZone('Europe/Berlin');
```

### `$jmespath()` — JSON querying

```javascript
const data = $input.first().json;

// Extract nested values
const names = $jmespath(data, 'users[*].name');
const active = $jmespath(data, 'users[?active==`true`].name');
const first = $jmespath(data, 'users[0].name');

// Filter and project
const result = $jmespath(data, 'items[?price > `10`].{name: name, cost: price}');
```

### `$getWorkflowStaticData()` — Persistent storage

Data persists across workflow executions. Use for: tracking processed IDs, rate limiting, counters.

```javascript
const staticData = $getWorkflowStaticData('global');

// Store a value
staticData.lastRunTime = DateTime.now().toISO();
staticData.processedIds = staticData.processedIds || [];

// Read and update
const lastRun = staticData.lastRunTime;
staticData.processedIds.push(currentId);

return [{ json: { lastRun, totalProcessed: staticData.processedIds.length } }];
```

---

## Error Prevention

### Error #1: Empty code / missing return (38% of failures)

```javascript
// WRONG — no return
const items = $input.all();
items.filter(i => i.json.active);

// CORRECT — always return
const items = $input.all();
return items.filter(i => i.json.active).map(i => ({ json: i.json }));
```

### Error #2: Expression syntax in Code nodes (8%)

```javascript
// WRONG — {{ }} syntax is for expression fields, not Code nodes
const name = {{ $json.name }};

// CORRECT — use plain JavaScript
const name = $input.first().json.name;
```

### Error #3: Wrong return wrapper (5%)

```javascript
// WRONG — missing json key
return [{ name: "test" }];

// CORRECT
return [{ json: { name: "test" } }];
```

### Error #4: Missing null checks

```javascript
// WRONG — crashes if field is missing
const email = $input.first().json.body.user.email;

// CORRECT — safe access
const email = $input.first().json.body?.user?.email ?? 'unknown';
```

### Error #5: Wrong data access in Each Item mode

```javascript
// WRONG in Each Item mode — $input.all() returns only current item
const items = $input.all(); // misleading, gives 1 item

// CORRECT in Each Item mode
const data = $input.item.json;
```

---

## Common Patterns

For complete patterns with production examples, see [PATTERNS.md](PATTERNS.md).

**Quick reference:**

```javascript
// Filter items
return $input.all()
  .filter(i => i.json.status === 'active')
  .map(i => ({ json: i.json }));

// Transform and enrich
return $input.all().map(i => ({
  json: { ...i.json, fullName: `${i.json.first} ${i.json.last}`, processedAt: DateTime.now().toISO() }
}));

// Aggregate into single item
const items = $input.all();
return [{ json: {
  count: items.length,
  total: items.reduce((s, i) => s + i.json.amount, 0),
  names: items.map(i => i.json.name)
}}];

// Deduplicate by field
const seen = new Set();
return $input.all().filter(i => {
  if (seen.has(i.json.email)) return false;
  seen.add(i.json.email);
  return true;
}).map(i => ({ json: i.json }));

// Group by field
const groups = {};
for (const item of $input.all()) {
  const key = item.json.category;
  groups[key] = groups[key] || [];
  groups[key].push(item.json);
}
return Object.entries(groups).map(([k, v]) => ({ json: { category: k, items: v, count: v.length } }));
```

---

## Python in Code Nodes

Use JavaScript for 95% of cases. Python is beta with significant limitations.

**Key differences from JavaScript:**
- `_input` instead of `$input`, `_json` instead of `$json`
- Must return `[{"json": {...}}]` (dict, not object)
- **No external libraries** — only standard library (json, datetime, re, base64, hashlib, math, random, urllib.parse)
- No `$helpers.httpRequest()` equivalent

```python
# Basic Python example
items = _input.all()
results = []
for item in items:
    results.append({"json": {"name": item.json["name"].upper()}})
return results
```

**When to use Python:** Only when you specifically need Python's standard library features (regex with `re`, statistical functions, specific data parsing) and the same cannot be done in JavaScript.

---

## Reference Files

- **Complete production patterns** (10 patterns with examples): [PATTERNS.md](PATTERNS.md)
