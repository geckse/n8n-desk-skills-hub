# Code Node Patterns

Production-tested patterns for n8n Code nodes (JavaScript).

## Contents

- [Multi-Source Aggregation](#multi-source-aggregation)
- [Regex Filtering](#regex-filtering)
- [Data Transformation and Enrichment](#data-transformation-and-enrichment)
- [Top N Filtering and Ranking](#top-n-filtering-and-ranking)
- [Aggregation and Reporting](#aggregation-and-reporting)
- [Slack Block Kit Formatting](#slack-block-kit-formatting)
- [HTTP API Call with Retry](#http-api-call-with-retry)
- [JSON Comparison and Diff](#json-comparison-and-diff)
- [Date-Based Filtering](#date-based-filtering)
- [Deduplication with Persistent State](#deduplication-with-persistent-state)

---

## Multi-Source Aggregation

Combine data from multiple upstream nodes into a unified output.

```javascript
const customers = $node["Get Customers"].json;
const orders = $node["Get Orders"].json;

const ordersByCustomer = {};
for (const order of orders.data) {
  const id = order.customerId;
  ordersByCustomer[id] = ordersByCustomer[id] || [];
  ordersByCustomer[id].push(order);
}

return customers.data.map(c => ({
  json: {
    name: c.name,
    email: c.email,
    orderCount: (ordersByCustomer[c.id] || []).length,
    totalSpent: (ordersByCustomer[c.id] || []).reduce((s, o) => s + o.amount, 0)
  }
}));
```

---

## Regex Filtering

Filter and extract data using regular expressions.

```javascript
const items = $input.all();
const pattern = /error|fail|exception/i;

return items
  .filter(i => pattern.test(i.json.message))
  .map(i => {
    const matches = i.json.message.match(/\b(ERROR|FAIL|EXCEPTION)\b:?\s*(.*)/i);
    return {
      json: {
        original: i.json.message,
        errorType: matches?.[1]?.toUpperCase() ?? 'UNKNOWN',
        detail: matches?.[2]?.trim() ?? i.json.message,
        timestamp: i.json.timestamp
      }
    };
  });
```

---

## Data Transformation and Enrichment

Transform fields and add computed values.

```javascript
const items = $input.all();

return items.map(i => {
  const d = i.json;
  return {
    json: {
      fullName: `${d.firstName} ${d.lastName}`,
      email: d.email.toLowerCase().trim(),
      age: DateTime.now().diff(DateTime.fromISO(d.birthDate), 'years').years | 0,
      tier: d.totalPurchases > 10000 ? 'gold' : d.totalPurchases > 1000 ? 'silver' : 'bronze',
      processedAt: DateTime.now().toISO()
    }
  };
});
```

---

## Top N Filtering and Ranking

Sort items and return the top N with rank.

```javascript
const items = $input.all();
const topN = 10;

return items
  .sort((a, b) => b.json.score - a.json.score)
  .slice(0, topN)
  .map((item, index) => ({
    json: {
      rank: index + 1,
      ...item.json
    }
  }));
```

---

## Aggregation and Reporting

Generate summary statistics from a dataset.

```javascript
const items = $input.all();
const values = items.map(i => i.json.amount);

const sum = values.reduce((a, b) => a + b, 0);
const avg = sum / values.length;
const sorted = [...values].sort((a, b) => a - b);
const median = sorted.length % 2 === 0
  ? (sorted[sorted.length / 2 - 1] + sorted[sorted.length / 2]) / 2
  : sorted[Math.floor(sorted.length / 2)];

const byCategory = {};
for (const item of items) {
  const cat = item.json.category;
  byCategory[cat] = (byCategory[cat] || 0) + item.json.amount;
}

return [{
  json: {
    count: items.length,
    total: sum,
    average: Math.round(avg * 100) / 100,
    median,
    min: sorted[0],
    max: sorted[sorted.length - 1],
    byCategory
  }
}];
```

---

## Slack Block Kit Formatting

Build Slack Block Kit messages programmatically.

```javascript
const items = $input.all();
const blocks = [
  { type: 'header', text: { type: 'plain_text', text: `Daily Report - ${DateTime.now().toFormat('MMMM d')}` } },
  { type: 'divider' }
];

for (const item of items.slice(0, 10)) {
  blocks.push({
    type: 'section',
    text: {
      type: 'mrkdwn',
      text: `*${item.json.title}*\nStatus: ${item.json.status} | Priority: ${item.json.priority}`
    }
  });
}

blocks.push(
  { type: 'divider' },
  { type: 'context', elements: [{ type: 'mrkdwn', text: `_${items.length} total items_` }] }
);

return [{ json: { blocks } }];
```

---

## HTTP API Call with Retry

Make HTTP requests with retry logic inside Code nodes.

```javascript
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await $helpers.httpRequest({ url, ...options });
    } catch (error) {
      if (attempt === maxRetries) throw error;
      // Wait with exponential backoff
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt - 1)));
    }
  }
}

const items = $input.all();
const results = [];

for (const item of items) {
  const data = await fetchWithRetry(
    `https://api.example.com/users/${item.json.id}`,
    { method: 'GET', headers: { 'Authorization': 'Bearer TOKEN' } }
  );
  results.push({ json: { ...item.json, apiData: data } });
}

return results;
```

---

## JSON Comparison and Diff

Compare two JSON objects and report differences.

```javascript
const current = $node["Get Current"].json;
const previous = $node["Get Previous"].json;

function diff(a, b, path = '') {
  const changes = [];
  const keys = new Set([...Object.keys(a || {}), ...Object.keys(b || {})]);

  for (const key of keys) {
    const fullPath = path ? `${path}.${key}` : key;
    if (!(key in (a || {}))) {
      changes.push({ path: fullPath, type: 'added', value: b[key] });
    } else if (!(key in (b || {}))) {
      changes.push({ path: fullPath, type: 'removed', value: a[key] });
    } else if (typeof a[key] === 'object' && typeof b[key] === 'object' && a[key] && b[key]) {
      changes.push(...diff(a[key], b[key], fullPath));
    } else if (a[key] !== b[key]) {
      changes.push({ path: fullPath, type: 'changed', from: a[key], to: b[key] });
    }
  }
  return changes;
}

const changes = diff(previous, current);
return [{ json: { hasChanges: changes.length > 0, changeCount: changes.length, changes } }];
```

---

## Date-Based Filtering

Filter items by date ranges using Luxon DateTime.

```javascript
const items = $input.all();
const now = DateTime.now();
const thirtyDaysAgo = now.minus({ days: 30 });

return items
  .filter(i => {
    const itemDate = DateTime.fromISO(i.json.createdAt);
    return itemDate >= thirtyDaysAgo && itemDate <= now;
  })
  .map(i => ({
    json: {
      ...i.json,
      daysAgo: Math.floor(now.diff(DateTime.fromISO(i.json.createdAt), 'days').days),
      isRecent: DateTime.fromISO(i.json.createdAt) >= now.minus({ days: 7 })
    }
  }));
```

---

## Deduplication with Persistent State

Track processed items across executions to avoid duplicates.

```javascript
const staticData = $getWorkflowStaticData('global');
staticData.processedIds = staticData.processedIds || [];

const items = $input.all();
const newItems = items.filter(i => !staticData.processedIds.includes(i.json.id));

// Remember new IDs (keep last 1000 to prevent unbounded growth)
for (const item of newItems) {
  staticData.processedIds.push(item.json.id);
}
if (staticData.processedIds.length > 1000) {
  staticData.processedIds = staticData.processedIds.slice(-1000);
}

return newItems.map(i => ({ json: i.json }));
```
