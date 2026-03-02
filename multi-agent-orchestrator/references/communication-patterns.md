# Communication Patterns

How agents share information in multi-agent systems.

## Direct Message Passing

Agents send messages directly to each other. Simplest pattern.

**Schema-first design**: Define message types as schemas before implementation. Every inter-agent message should be a typed object, not free-form text.

```python
# Example with Pydantic
class ResearchResult(BaseModel):
    query: str
    findings: list[Finding]
    confidence: float  # 0-1
    sources: list[str]
    
class Finding(BaseModel):
    claim: str
    evidence: str
    relevance: float  # 0-1
```

**Rules**:
- Never pass raw LLM output between agents. Always parse into structured format first.
- Validate output schema before sending. Validate input schema on receive.
- Include metadata: agent_id, timestamp, request_id for tracing.
- Keep messages minimal — only what the receiving agent needs. Don't forward full conversation history unless necessary.

## Shared State / Blackboard

A central state store that all agents can read from and write to. Agents observe the state, decide if they can contribute, and update it.

**When to use**: When multiple agents need access to the same evolving context. Examples: collaborative document editing, multi-step planning where context accumulates.

**Implementation**:
```python
class SharedState:
    def __init__(self):
        self._state: dict = {}
        self._history: list[StateUpdate] = []
    
    def read(self, key: str) -> Any:
        return self._state.get(key)
    
    def write(self, key: str, value: Any, agent_id: str):
        self._history.append(StateUpdate(
            agent_id=agent_id,
            key=key,
            old_value=self._state.get(key),
            new_value=value,
            timestamp=time.time()
        ))
        self._state[key] = value
    
    def get_history(self, key: str = None) -> list[StateUpdate]:
        if key:
            return [u for u in self._history if u.key == key]
        return self._history
```

**Rules**:
- Always log writes with agent_id and timestamp for debugging.
- Use read-modify-write with optimistic locking if agents run concurrently.
- Scope state keys by domain — `research.findings`, `draft.current`, `review.feedback` — to avoid collisions.
- Set size limits on state values. An agent writing a novel into shared state will blow context windows of other agents reading it.

## Event-Driven / Pub-Sub

Agents publish events; interested agents subscribe and react. Decoupled communication.

**When to use**: When agents don't need to know about each other, only about events. Good for extensible systems where you want to add agents without modifying existing ones.

**Implementation**:
```python
class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        self._subscribers.setdefault(event_type, []).append(handler)
    
    async def publish(self, event_type: str, data: dict):
        for handler in self._subscribers.get(event_type, []):
            await handler(data)
```

**Rules**:
- Define event types as an enum or constant set. No free-form event names.
- Events should be immutable facts about what happened, not commands.
- Include correlation_id in every event for end-to-end tracing.
- Handle subscriber failures independently — one failing subscriber shouldn't block others.

## Hierarchical Reporting

Agents report to a parent agent, which aggregates and reports upward. Information flows up; commands flow down.

**When to use**: Hierarchical architectures where higher-level agents need summaries, not raw details.

**Rules**:
- Define report schemas at each level. Lower levels produce detailed reports; higher levels consume summaries.
- Implement summarization at each boundary — raw output from 5 workers should be condensed before reaching the orchestrator.
- Set reporting deadlines. If a worker hasn't reported within timeout, the supervisor proceeds without it.

## Choosing a Pattern

| Pattern | Complexity | Best For | Avoid When |
|---------|-----------|----------|------------|
| Direct Message | Low | Small systems, pipelines | Many-to-many communication |
| Shared State | Medium | Collaborative tasks | High concurrency |
| Event-Driven | Medium | Extensible systems | Simple linear flows |
| Hierarchical | High | Deep task decomposition | Flat task structure |

**Default recommendation**: Start with direct message passing. Move to shared state if agents need common context. Use event-driven only if you need decoupling. Hierarchical reporting is only for hierarchical architectures.
