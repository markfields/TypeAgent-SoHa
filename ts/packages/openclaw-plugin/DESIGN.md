# OpenClaw + StructuredRAG Integration Design

## Overview

Integrate OpenClaw (self-hosted AI agent gateway) with TypeAgent's StructuredRAG to capture **Moments** — events worth remembering — via chat or webhooks, index them with structured knowledge extraction (entities, actions, topics), and query them with natural language.

**OpenClaw = UX layer** (capture + query Moments via chat/webhooks)
**StructuredRAG = Knowledge layer** (structured indexing + search)

---

## System Research

### OpenClaw

A **self-hosted AI agent gateway** connecting chat platforms (WhatsApp, Telegram, Slack, etc.) to AI agents. Key characteristics:

- **Memory is Markdown files on disk** (`memory/YYYY-MM-DD.md`) with hybrid vector+BM25 search backed by SQLite
- **Plugin system**: In-process TypeScript modules that register tools, memory backends, RPC methods, and more
- **Webhooks**: `POST /hooks/agent` to trigger agent turns, `POST /hooks/wake` for lightweight events
- **Tools Invoke API**: `POST /tools/invoke` for direct tool calls
- **OpenResponses API**: OpenAI-compatible `POST /v1/responses` (disabled by default)
- **Skills**: `SKILL.md` files that teach agents how to use tools (injected into system prompts)
- **Memory backends are swappable** via plugins (e.g., `memory-core`, `memory-lancedb`)

Docs: https://docs.openclaw.ai/

### TypeAgent StructuredRAG

Everything is modeled as `IConversation` → sequences of `IMessage`. Key characteristics:

- **Knowledge extraction**: Message text → LLM (GPT-4o) → `KnowledgeResponse` containing entities, actions, topics
- **Inverted index** over extracted knowledge (not traditional vector RAG)
- **Incremental indexing**: `addMessage()` indexes without rebuilding
- **Two search modes**: Programmatic (`SearchTermGroup`) or natural language (`searchWithLanguage()`)
- **Answer generation**: Search results → LLM → natural language answer
- **File-based persistence**: JSON + binary embeddings

Key packages:
| Package | Path | Purpose |
|---------|------|---------|
| `knowPro` | `ts/packages/knowPro/` | Core indexing, search, and query engine |
| `conversation-memory` | `ts/packages/memory/conversation/` | High-level `ConversationMemory` API |
| `knowledgeProcessor` | `ts/packages/knowledgeProcessor/` | Knowledge extraction schemas |

Key API (from `ConversationMemory`):
```typescript
// Index a moment
await memory.addMessage(message);

// Natural language search
const results = await memory.searchWithLanguage("what happened with X?");

// Natural language Q&A
const answer = await memory.getAnswerFromLanguage("summarize meetings about Y");
```

---

## Chosen Approach: OpenClaw Plugin

### Why

1. **Simplest path** — one process, one deployment, direct TypeScript imports
2. **Fits OpenClaw's paradigm** — plugins are the idiomatic extension mechanism
3. **Evolvable** — can extract into a microservice (Option B) or add dual-write (Option C) later
4. **StructuredRAG's extraction is the value-add** — entities/actions/topics are what OpenClaw's native memory doesn't do

### Plugin Architecture

```
ts/packages/openclaw-plugin/
├── package.json
├── src/
│   ├── index.ts          # Plugin entry point (registers tools)
│   ├── momentLog.ts      # moment_log tool implementation
│   ├── momentSearch.ts   # moment_search tool implementation
│   └── memoryManager.ts  # ConversationMemory lifecycle (init, persist, load)
├── SKILL.md              # Teaches the OpenClaw agent when/how to use the tools
└── DESIGN.md             # This file
```

### Tool Definitions

#### `moment_log`

```typescript
{
  name: "moment_log",
  description: "Log a Moment — something that happened worth remembering in detail or aggregating with other Moments later.",
  parameters: {
    text: string,         // What happened (free-form description)
    tags?: string[],      // Optional categorization tags
    source?: string,      // Where this moment came from (e.g., "meeting", "thought", "observation")
    timestamp?: string,   // ISO 8601; defaults to now
  }
}
```

Implementation:
```typescript
const message = new ConversationMessage(
  [params.text],
  new ConversationMessageMeta(params.source ?? "user"),
  params.tags ?? [],
  undefined,            // let StructuredRAG auto-extract knowledge
  params.timestamp ?? new Date().toISOString()
);
await memory.addMessage(message, /* extractKnowledge */ true, /* retain */ true);
```

#### `moment_search`

```typescript
{
  name: "moment_search",
  description: "Search previously logged Moments using natural language. Returns relevant moments and a synthesized answer.",
  parameters: {
    query: string,        // Natural language question
    maxResults?: number,  // Max moments to return (default 10)
  }
}
```

Implementation:
```typescript
const answer = await memory.getAnswerFromLanguage(params.query, {
  maxKnowledgeMatches: params.maxResults ?? 10,
});
return {
  answer: answer.answer,
  // Optionally include raw matched entities/actions/topics for the agent
};
```

### SKILL.md (Agent Instructions)

The skill file teaches the OpenClaw agent when to use these tools:

```markdown
# Moments

You have access to a structured memory system for logging and recalling Moments.

## When to use `moment_log`
- When the user says "remember this", "log this", "note that..."
- When the user shares something they want to recall later
- When capturing meeting notes, observations, ideas, or events

## When to use `moment_search`
- When the user asks "what did I say about...", "when did...", "summarize..."
- When the user wants to recall or aggregate past Moments
- When answering questions that depend on previously logged information

## Tips
- Extract a clear, descriptive text for each Moment
- Add relevant tags for categorization
- Set the source field to help distinguish where Moments came from
```

### Data Flow (End to End)

#### Logging a Moment

```
1. User (via WhatsApp/Telegram/webhook): "Remember that I met with Alice about Project X today"
2. OpenClaw routes message to agent
3. Agent recognizes this as a moment to log
4. Agent calls moment_log({ text: "Met with Alice about Project X", tags: ["meeting", "project-x"], source: "user" })
5. Plugin creates ConversationMessage
6. ConversationMemory.addMessage() triggers:
   a. LLM extracts knowledge:
      - Entity: Alice (person)
      - Entity: Project X (project)
      - Action: [user] met [Alice] about [Project X]
      - Topic: "Meeting about Project X progress"
   b. Knowledge indexed into inverted index
   c. Data persisted to disk
7. Agent confirms: "Got it — logged your meeting with Alice about Project X."
```

#### Querying Moments

```
1. User: "What have I discussed with Alice recently?"
2. Agent calls moment_search({ query: "discussions with Alice" })
3. Plugin calls memory.getAnswerFromLanguage("discussions with Alice")
4. StructuredRAG:
   a. Translates NL query → SearchQuery (entity: Alice, action: discussed)
   b. Searches inverted index for matching SemanticRefs
   c. Retrieves matched messages + knowledge
   d. LLM generates answer from matched context
5. Agent returns synthesized answer to user
```

### Dependencies

The plugin would depend on:
- `@typeagent/knowpro` — Core indexing and search
- `@typeagent/conversation-memory` — `ConversationMemory` high-level API
- OpenClaw's plugin SDK (for tool registration)

### Open Questions

1. **OpenClaw plugin SDK specifics** — Need to verify the exact plugin registration API (tool schema format, lifecycle hooks). The docs describe the concept but implementation details may require inspecting existing plugins.
2. **Storage location** — Where should the StructuredRAG index live? Options: inside OpenClaw's agent workspace (`~/.openclaw/agents/<id>/structuredrag/`) or a separate configurable path.
3. **Concurrency** — If multiple Moments arrive simultaneously, `ConversationMemory` has `queueAddMessage()` for async background processing. Should the plugin use this or synchronous `addMessage()`?

---

## Using Claude as the LLM

StructuredRAG was built and tested against GPT-4o, but the architecture is largely model-agnostic. Here's what's involved in using a Claude API key instead.

### Where StructuredRAG Uses an LLM

There are **four** distinct LLM touch-points:

| Purpose | What it does | Interface | Latency sensitivity |
|---------|-------------|-----------|-------------------|
| **Knowledge extraction** | Converts message text → `KnowledgeResponse` (entities, actions, topics) | TypeChat `JsonTranslator` | Low (background indexing) |
| **Query translation** | Converts NL question → `SearchQuery` (structured filters) | TypeChat `JsonTranslator` | High (user-facing) |
| **Answer generation** | Converts search results → NL answer | TypeChat `JsonTranslator` | High (user-facing) |
| **Answer combining** | Merges chunked partial answers into one | Free-form `model.complete()` | Medium |

All four accept an injected `TypeChatLanguageModel` — a trivial interface with one method:
```typescript
interface TypeChatLanguageModel {
    complete(prompt: string | PromptSection[]): Promise<Result<string>>;
}
```

The prompts are standard "translate this text to JSON matching this TypeScript schema" instructions. No OpenAI-specific features (function calling, structured outputs, tool_use) are used.

### Where Embeddings Are Used

In addition to chat LLM calls, StructuredRAG uses **text embeddings** for two secondary indexes:

| Index | Purpose | Default |
|-------|---------|---------|
| **Related terms index** | Fuzzy term matching (e.g., "novel" ≈ "book") | OpenAI `text-embedding-ada-002` (1536 dims) |
| **Message text index** | Semantic similarity re-ranking of messages | Same |

Anthropic does not offer an embeddings API. Options:
- **Keep OpenAI embeddings** — use Claude for chat, OpenAI just for embeddings (cheap, simple, proven)
- **Switch to Voyage AI** — Anthropic's recommended embedding partner; implement `TextEmbeddingModel` adapter
- **Skip secondary indexes** — set `relatedTermIndexSettings.embeddingModel = undefined` and rely on the primary inverted index only. This loses fuzzy matching but the core entity/action/topic search still works.

**Recommendation:** Keep OpenAI embeddings for now. It's a few cents per million tokens and avoids any regression risk. Swap later if desired.

### What Needs to Change in Code

The `aiclient` package (`ts/packages/aiclient/`) speaks the OpenAI/Azure REST protocol exclusively. There are two paths to Claude:

#### Path 1: Implement `createClaudeChatModel()` (Recommended)

Write a small adapter (~80 lines) that implements `ChatModel` / `TypeChatLanguageModel` using the Anthropic Messages API:

```typescript
// Pseudocode
function createClaudeChatModel(apiKey: string, model: string): ChatModel {
    return {
        complete(prompt) {
            // Convert PromptSection[] → Anthropic messages format
            // POST to https://api.anthropic.com/v1/messages
            // Return result.content[0].text
        }
    };
}
```

Key translation details:
- `PromptSection[]` has `role: "system" | "user" | "assistant"` — maps directly to Anthropic's API
- Strip `response_format: { type: "json_object" }` — not needed because TypeChat's prompt already instructs JSON output
- Strip `seed` parameter (OpenAI-specific, used for determinism in tests)
- `temperature` maps directly

Then inject this model into the three creation points:
- `ConversationMemory` constructor (for knowledge extraction)
- `SearchQueryTranslator` constructor (for query translation)
- `AnswerGeneratorSettings` (for answer generation + combining)

All three already accept an optional model parameter — no changes to StructuredRAG internals needed.

#### Path 2: OpenAI-Compatible Proxy (LiteLLM, etc.)

Run a proxy that translates OpenAI API calls to Anthropic calls. Point `OPENAI_ENDPOINT` at it. Zero code changes but adds a deployment dependency.

**Recommendation:** Path 1. It's a small, self-contained adapter with no external dependencies.

### Risks and Mitigations

#### Risk 1: Knowledge Extraction Quality

The knowledge extraction schema (`KnowledgeResponse`) is a deeply nested TypeScript type with `ConcreteEntity`, `Action`, `Facet`, `VerbTense`, etc. The LLM must produce valid JSON matching this schema precisely.

**Mitigation:**
- Claude is strong at structured JSON output. TypeChat's repair loop (sends JSON validation errors back to the model for correction) provides a safety net.
- The knowPro README notes: *"KnowPro has been primarily tested with GPT-4o. Results with other models are not guaranteed."* Start by running the existing test suite (`pnpm --filter knowpro test`) with Claude and comparing extraction quality.
- Consider running a small A/B: extract knowledge from 20-30 sample Moments with both GPT-4o and Claude, diff the results.

#### Risk 2: Query Translation Accuracy

The `SearchQuery` schema requires the LLM to decompose a natural language question into `ActionTerm` (verbs, actors, targets), `EntityTerm` (name, type, facets), and `DateTimeRange` filters. Subtle differences in how Claude interprets the schema could affect search recall.

**Mitigation:**
- TypeChat's schema-as-prompt approach is model-agnostic by design — the TypeScript types *are* the instructions.
- The query translation tests in knowPro provide a regression baseline.

#### Risk 3: JSON Format Compliance

The `createJsonChatModel()` function sets `response_format: { type: "json_object" }`, an OpenAI-specific parameter that guarantees valid JSON output. Claude doesn't have this exact feature.

**Mitigation:**
- TypeChat already includes JSON instructions in the prompt text itself ("respond with JSON matching this schema").
- TypeChat's repair loop catches and fixes malformed JSON (up to `retryMaxAttempts` retries).
- Claude's JSON compliance rate from prompt instructions alone is very high in practice. The `response_format` parameter is a belt-and-suspenders safeguard, not a hard requirement.

#### Risk 4: Cost Differences

Claude and GPT-4o have different pricing. Knowledge extraction runs on every `addMessage()` call.

**Mitigation:**
- Use Claude Haiku for knowledge extraction (background, latency-insensitive) and Claude Sonnet for query translation / answer generation (user-facing, quality-sensitive).
- The `ConversationSettings.semanticRefIndexSettings.batchSize` controls how many messages are batched per extraction call — tune this to manage cost.

---

## Alternatives Considered

- **StructuredRAG Microservice + Thin Plugin** — Run StructuredRAG as a separate HTTP service; OpenClaw calls it via plugin. Better separation of concerns but adds deployment complexity. Can evolve to this later if needed.
- **Dual-Write (Both Memory Systems)** — Write Moments to both OpenClaw's native memory and StructuredRAG. Gets benefits of both but adds consistency complexity. Worth revisiting once the core integration is proven.

---

## Future Ideas

- **Webhook endpoint for external Moments** — Expose a dedicated `POST /hooks/moment` webhook so external systems (scripts, IoT, other apps) can log Moments without going through the chat agent. The hook would call `moment_log` directly, bypassing the LLM agent turn for lower latency and cost.
- **Moment aggregation reports** — Scheduled agent turns (via OpenClaw heartbeat hooks) that aggregate recent Moments into daily/weekly summaries, stored as their own Moments for hierarchical recall.
- **Structured Moment types** — Define typed Moment schemas (meeting, observation, decision, idea) with type-specific fields, enabling richer knowledge extraction and faceted search.
- **Dual-write to OpenClaw native memory** — Once the core plugin is stable, optionally also write Moments to OpenClaw's `memory/YYYY-MM-DD.md` files so the agent can recall them even without the StructuredRAG tool.
- **Multi-agent Moment sharing** — Use OpenClaw's multi-agent routing to let different agents (work, personal, project-specific) share a single StructuredRAG index or maintain isolated indexes.
