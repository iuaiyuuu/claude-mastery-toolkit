# State Management

How to manage shared state, memory, and context across agents in multi-agent systems.

## The Context Problem

Every agent call starts with a fresh context window. Multi-agent state management is the problem of getting the right information into the right agent's context at the right time — without exceeding token limits or introducing stale data.

## Strategies

### 1. Full Context Forwarding

Pass the entire conversation/state history to each agent.

**Pros**: Agents have complete information. No information loss.
**Cons**: Expensive in tokens. Hits context limits fast. Agents get confused by irrelevant context.

**When to use**: Small systems with short conversations. Prototype/MVP stage.

**Implementation**: Simply prepend the full state as a system message or first user message to each agent call.

### 2. Selective Context Extraction

Extract only the relevant key-value pairs for each agent.

**Pros**: Efficient token usage. Agents get focused context. Scales better.
**Cons**: Requires knowing what each agent needs upfront. Risk of missing relevant context.

**When to use**: Production systems where you understand each agent's information needs.

**Implementation**:
```python
class ContextManager:
    def __init__(self):
        self._context: dict = {}
    
    def update(self, key: str, value: Any, source_agent: str):
        self._context[key] = {
            "value": value,
            "source": source_agent,
            "updated_at": time.time()
        }
    
    def get_context_for_agent(self, agent_id: str, required_keys: list[str]) -> dict:
        """Return only the context keys this agent needs."""
        return {
            k: self._context[k]["value"] 
            for k in required_keys 
            if k in self._context
        }
```

Define a context manifest per agent — a list of keys it reads and writes:
```python
AGENT_CONTEXT_MANIFEST = {
    "researcher": {
        "reads": ["query", "search_constraints"],
        "writes": ["research_findings", "sources"]
    },
    "writer": {
        "reads": ["query", "research_findings", "style_guide"],
        "writes": ["draft"]
    },
    "reviewer": {
        "reads": ["query", "draft", "review_criteria"],
        "writes": ["review_feedback", "approval_status"]
    }
}
```

### 3. Summarized Context

Use an LLM to summarize accumulated context before passing it to the next agent.

**Pros**: Handles long-running workflows. Keeps context window manageable.
**Cons**: Information loss through summarization. Additional LLM call (cost + latency). Summarization quality affects downstream agents.

**When to use**: Long-running workflows where context accumulates beyond window limits. Multi-turn conversations that need history compression.

**Implementation**:
```python
async def summarize_context(full_context: str, focus: str, model: str) -> str:
    """Summarize context with a focus area for the next agent."""
    response = await llm_call(
        model=model,
        system="Summarize the following context, focusing on information relevant to: " + focus,
        messages=[{"role": "user", "content": full_context}],
        max_tokens=500
    )
    return response.text
```

**Rules**:
- Always specify a focus area for summarization — generic summaries lose important details.
- Keep the original context available for fallback — if a summarized context leads to a bad result, retry with full context.
- Use a cheaper/faster model for summarization if the task is straightforward.

### 4. External Memory Store

Store state in a database or vector store that agents query as needed.

**Pros**: Scales to arbitrary amounts of state. Agents can query relevant information dynamically. Persistent across sessions.
**Cons**: Most complex to implement. Query quality affects agent performance. Additional infrastructure dependency.

**When to use**: Long-running systems, multi-session workflows, systems that need to remember across conversations.

**Implementation options**:
- **Key-value store** (Redis, DynamoDB): For structured state with known keys.
- **Vector store** (Pinecone, ChromaDB, pgvector): For semantic retrieval of relevant context.
- **Relational DB**: For complex state with relationships.

```python
class AgentMemory:
    def __init__(self, vector_store, kv_store):
        self.vector_store = vector_store
        self.kv_store = kv_store
    
    async def store(self, key: str, value: Any, metadata: dict = None):
        """Store in KV for exact lookup, vector store for semantic search."""
        await self.kv_store.set(key, value)
        if metadata:
            await self.vector_store.upsert(
                id=key,
                text=str(value),
                metadata=metadata
            )
    
    async def recall(self, query: str, top_k: int = 5) -> list:
        """Semantic search for relevant memories."""
        return await self.vector_store.query(query, top_k=top_k)
    
    async def get(self, key: str) -> Any:
        """Exact key lookup."""
        return await self.kv_store.get(key)
```

## Common Pitfalls

1. **Stale reads**: Agent A updates state, but Agent B reads an old version. Use timestamps and version numbers.
2. **Context overflow**: Accumulating state without pruning. Set TTLs or implement sliding windows.
3. **Circular dependencies**: Agent A needs Agent B's output, which needs Agent A's output. Detect and break cycles.
4. **Missing state initialization**: Agent expects a key that hasn't been set yet. Always define default values.
5. **State corruption**: One agent writes bad data that poisons all downstream agents. Validate writes.

## Choosing a Strategy

| Strategy | Token Cost | Complexity | Information Loss | Best For |
|----------|-----------|------------|-----------------|----------|
| Full Forwarding | High | Low | None | Prototypes, short workflows |
| Selective Extraction | Low | Medium | Controlled | Production pipelines |
| Summarized Context | Medium | Medium | Some | Long-running workflows |
| External Memory | Low (per call) | High | Query-dependent | Multi-session systems |

**Default recommendation**: Start with selective context extraction. It's the best balance of simplicity and efficiency. Add summarization or external memory only when context exceeds window limits.
