# Suggested Nodes by Category

Use this before calling `claude.ai n8n beta:search_nodes` to understand which nodes and patterns fit the use case.

## Contents

- [Chatbot](#chatbot)
- [Notification](#notification)
- [Scheduling](#scheduling)
- [Data Transformation](#data-transformation)
- [Data Persistence](#data-persistence)
- [Data Extraction](#data-extraction)
- [Document Processing](#document-processing)
- [Form Input](#form-input)
- [Content Generation](#content-generation)
- [Triage](#triage)
- [Scraping and Research](#scraping-and-research)

---

## Chatbot

**Pattern:** Chat Trigger -> AI Agent -> Memory -> Response

| Node | Display Name | Notes |
|---|---|---|
| `@n8n/n8n-nodes-langchain.chatTrigger` | Chat Trigger | When `loadPreviousSession` is set to memory, downstream Agent must also have a memory subnode |
| `@n8n/n8n-nodes-langchain.agent` | AI Agent | Every conversational agent MUST have a memory subnode. Shared conversations need same session key |
| `@n8n/n8n-nodes-langchain.lmChatOpenAi` | OpenAI Chat Model | |
| `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Google Gemini Chat Model | |
| `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Simple Memory | No credentials required. Must be subnode of every Agent in a conversation |
| `@n8n/n8n-nodes-langchain.retrieverVectorStore` | Vector Store Retriever | Connects any Vector Store to Agent for RAG |
| `n8n-nodes-base.slack` | Slack | |
| `n8n-nodes-base.telegram` | Telegram | |
| `n8n-nodes-base.whatsApp` | WhatsApp Business Cloud | |
| `n8n-nodes-base.discord` | Discord | |

---

## Notification

**Pattern:** Trigger -> Condition -> Send (Email/Slack/SMS)

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.webhook` | Webhook | Event-based notifications |
| `n8n-nodes-base.scheduleTrigger` | Schedule Trigger | Periodic monitoring |
| `n8n-nodes-base.gmail` | Gmail | Default — easy setup |
| `n8n-nodes-base.slack` | Slack | |
| `n8n-nodes-base.telegram` | Telegram | |
| `n8n-nodes-base.twilio` | Twilio | SMS, WhatsApp, phone calls |
| `n8n-nodes-base.httpRequest` | HTTP Request | For services without dedicated nodes |
| `n8n-nodes-base.if` | If | Check alert conditions |
| `n8n-nodes-base.switch` | Switch | Route by severity/type |

---

## Scheduling

**Pattern:** Schedule Trigger -> Fetch -> Process -> Act

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.scheduleTrigger` | Schedule Trigger | |
| `n8n-nodes-base.httpRequest` | HTTP Request | |
| `n8n-nodes-base.set` | Edit Fields (Set) | |
| `n8n-nodes-base.wait` | Wait | Respect rate limits |

---

## Data Transformation

**Pattern:** Input -> Filter/Map -> Transform -> Output

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.set` | Edit Fields (Set) | |
| `n8n-nodes-base.if` | If | Validate inputs early |
| `n8n-nodes-base.filter` | Filter | Reduce data volume early |
| `n8n-nodes-base.summarize` | Summarize | Pivot table-style aggregations |
| `n8n-nodes-base.aggregate` | Aggregate | Combine multiple items into one |
| `n8n-nodes-base.splitOut` | Split Out | Array to multiple items |
| `n8n-nodes-base.sort` | Sort | |
| `n8n-nodes-base.limit` | Limit | |
| `n8n-nodes-base.removeDuplicates` | Remove Duplicates | |
| `n8n-nodes-base.splitInBatches` | Loop Over Items | Batch 100+ items to prevent timeouts |

---

## Data Persistence

**Pattern:** Trigger -> Process -> Store

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.dataTable` | Data Table | **PREFERRED** — no external config needed |
| `n8n-nodes-base.googleSheets` | Google Sheets | For collaboration; consider DataTable if >10k rows |
| `n8n-nodes-base.airtable` | Airtable | When relationships between tables needed |
| `n8n-nodes-base.postgres` | Postgres | |
| `n8n-nodes-base.mySql` | MySQL | |
| `n8n-nodes-base.mongoDb` | MongoDB | |

---

## Data Extraction

**Pattern:** Source -> Extract -> Parse -> Structure

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.extractFromFile` | Extract from File | Route by file type with IF/Switch |
| `n8n-nodes-base.htmlExtract` | HTML Extract | JS-rendered content may be empty |
| `n8n-nodes-base.splitOut` | Split Out | Use before Loop for arrays |
| `n8n-nodes-base.splitInBatches` | Loop Over Items | 200 rows at a time |
| `n8n-nodes-base.code` | Code | Custom JavaScript or Python |
| `@n8n/n8n-nodes-langchain.informationExtractor` | Information Extractor | For unstructured text |
| `@n8n/n8n-nodes-langchain.chainSummarization` | Summarization Chain | Context window limits may truncate |

---

## Document Processing

**Pattern:** Trigger -> Extract Text -> AI Parse -> Store

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.gmailTrigger` | Gmail Trigger | |
| `n8n-nodes-base.googleDriveTrigger` | Google Drive Trigger | |
| `n8n-nodes-base.extractFromFile` | Extract from File | Different operations per file type |
| `n8n-nodes-base.awsTextract` | AWS Textract | Tables and forms in scanned docs |
| `n8n-nodes-base.mindee` | Mindee | Invoice/receipt parsing |
| `@n8n/n8n-nodes-langchain.agent` | AI Agent | |
| `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | Default Data Loader | Loads PDF, CSV, JSON, DOCX, EPUB, text into LangChain Documents |
| `@n8n/n8n-nodes-langchain.vectorStoreInMemory` | Simple Vector Store | No external dependencies |
| `n8n-nodes-base.splitInBatches` | Loop Over Items | 5-10 files at a time |

---

## Form Input

**Pattern:** Form Trigger -> Validate -> Store -> Respond

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.formTrigger` | n8n Form Trigger | ALWAYS store raw data to persistent storage |
| `n8n-nodes-base.form` | n8n Form | Each node is one page/step |
| `n8n-nodes-base.dataTable` | Data Table | **PREFERRED** for form data storage |
| `n8n-nodes-base.googleSheets` | Google Sheets | |
| `n8n-nodes-base.airtable` | Airtable | |

---

## Content Generation

**Pattern:** Trigger -> Generate (Text/Image/Video) -> Deliver

| Node | Display Name | Notes |
|---|---|---|
| `@n8n/n8n-nodes-langchain.agent` | AI Agent | Text generation |
| `@n8n/n8n-nodes-langchain.openAi` | OpenAI | DALL-E, TTS, Sora video |
| `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Google Gemini | Imagen, video generation |
| `n8n-nodes-base.httpRequest` | HTTP Request | APIs without dedicated nodes |
| `n8n-nodes-base.editImage` | Edit Image | Resize, crop, format conversion |
| `n8n-nodes-base.markdown` | Markdown | Convert to HTML |
| `n8n-nodes-base.wait` | Wait | Video generation is async — poll for status |

---

## Triage

**Pattern:** Trigger -> Classify -> Route -> Act

| Node | Display Name | Notes |
|---|---|---|
| `@n8n/n8n-nodes-langchain.agent` | AI Agent | Use structured output parser + temperature 0-0.2 for consistency |
| `@n8n/n8n-nodes-langchain.outputParserStructured` | Structured Output Parser | Critical for consistent, schema-matching output |

---

## Scraping and Research

**Pattern:** Trigger -> Fetch -> Extract -> Store

| Node | Display Name | Notes |
|---|---|---|
| `n8n-nodes-base.dataTable` | Data Table | Default storage for scraped data |
| `n8n-nodes-base.phantombuster` | Phantombuster | LinkedIn, Facebook, Instagram, Twitter |
| `@n8n/n8n-nodes-langchain.toolSerpApi` | SerpApi (Google Search) | Agent web search capability |
| `n8n-nodes-base.perplexity` | Perplexity | Up-to-date news with citations |
| `n8n-nodes-base.perplexityTool` | Perplexity Tool | As agent tool |
| `n8n-nodes-base.htmlExtract` | HTML Extract | JS-rendered sites may return empty |
| `n8n-nodes-base.splitInBatches` | Loop Over Items | 200 rows at a time |
| `n8n-nodes-base.wait` | Wait | Avoid rate limits (429 errors) |
| `n8n-nodes-base.httpRequest` | HTTP Request | |
| `n8n-nodes-base.httpRequestTool` | HTTP Request Tool | As agent tool |
