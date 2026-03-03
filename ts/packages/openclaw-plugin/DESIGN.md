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

## Integration Approaches Considered

### Option A: OpenClaw Plugin (Recommended)

Build an OpenClaw plugin that imports StructuredRAG's TypeScript packages directly.

```
User (chat/webhook)
  → OpenClaw Agent
    → moment_log tool   → ConversationMemory.addMessage() → StructuredRAG index
    → moment_search tool → ConversationMemory.getAnswerFromLanguage() → answer
```

The plugin registers two tools the agent can call:
- **`moment_log`** — Takes moment text + metadata, creates a `ConversationMessage`, calls `addMessage()` which triggers knowledge extraction + indexing
- **`moment_search`** — Takes a natural language question, calls `searchWithLanguage()` / `getAnswerFromLanguage()`, returns structured results

**Pros:** Single process, no network hops, simplest deployment, full access to StructuredRAG's typed APIs (entity/action/topic queries). Fits OpenClaw's plugin paradigm natively.

**Cons:** Tightly coupled — StructuredRAG's dependencies (TypeChat, GPT-4o for extraction) run inside the OpenClaw process. Version coupling between OpenClaw and TypeAgent packages.

### Option B: StructuredRAG Microservice + Thin Plugin

Run StructuredRAG as a standalone HTTP service (small Express/Fastify wrapper). OpenClaw calls it via an HTTP-calling plugin or tool.

```
User (chat/webhook)
  → OpenClaw Agent
    → moment_log tool → HTTP POST /moments → StructuredRAG service → index
    → moment_search tool → HTTP POST /search → StructuredRAG service → results
```

**Pros:** Loose coupling, independent deployment/scaling, StructuredRAG service reusable by other clients. Clean process boundary.

**Cons:** More moving parts (two processes to run), HTTP serialization overhead, need to define an API contract.

### Option C: Dual-Write (Both Memory Systems)

Write Moments to both OpenClaw's native memory (for quick conversational recall) and StructuredRAG (for deep structured queries).

```
User → OpenClaw Agent
  → moment_log tool
      → append to memory/YYYY-MM-DD.md  (OpenClaw native, auto-indexed)
      → ConversationMemory.addMessage()  (StructuredRAG index)
  → memory_search (quick recall, built-in)
  → structured_search (deep query, StructuredRAG)
```

**Pros:** Best of both — OpenClaw's temporal memory for "what happened today?" and StructuredRAG's knowledge graph for "what entities are related to X?". Graceful degradation.

**Cons:** Dual writes, potential consistency issues, more complex agent instructions.

---

## Recommendation: Option A (OpenClaw Plugin)

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
2. **LLM configuration** — StructuredRAG uses GPT-4o for knowledge extraction. Need to decide whether to share OpenClaw's LLM credentials or configure separately.
3. **Storage location** — Where should the StructuredRAG index live? Options: inside OpenClaw's agent workspace (`~/.openclaw/agents/<id>/structuredrag/`) or a separate configurable path.
4. **Concurrency** — If multiple Moments arrive simultaneously, `ConversationMemory` has `queueAddMessage()` for async background processing. Should the plugin use this or synchronous `addMessage()`?
