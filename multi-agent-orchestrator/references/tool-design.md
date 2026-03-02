# Tool Design for Agents

Principles for designing effective tools that agents can use reliably.

## Core Principles

### 1. Single Responsibility

Each tool does one thing. A tool that "searches and summarizes" should be two tools: `search` and `summarize`. Agents are better at composing simple tools than using complex ones.

**Bad**: `research_and_analyze(query)` — does too much, hard to retry partially.
**Good**: `web_search(query)` + `extract_content(url)` + `analyze_text(text, criteria)`.

### 2. Clear, Descriptive Names and Descriptions

The tool name and description are the agent's primary interface. If the agent can't understand when and how to use a tool from its description alone, the tool will be misused.

```python
# Bad — vague
Tool(name="process", description="Processes data")

# Good — specific
Tool(
    name="extract_structured_data",
    description="Extract structured key-value pairs from unstructured text. "
    "Input: raw text and a list of fields to extract. "
    "Output: JSON object with extracted fields and confidence scores. "
    "Use when you have unstructured text and need specific data points."
)
```

### 3. Strict Input/Output Schemas

Define explicit schemas for every tool. Never accept or return `Any` or free-form strings.

```python
class SearchInput(BaseModel):
    query: str = Field(description="Search query, 3-10 words for best results")
    max_results: int = Field(default=5, ge=1, le=20, description="Number of results")
    
class SearchResult(BaseModel):
    title: str
    url: str
    snippet: str
    relevance_score: float
    
class SearchOutput(BaseModel):
    results: list[SearchResult]
    total_found: int
    query_used: str  # Echo back for verification
```

### 4. Informative Error Messages

When a tool fails, the error message should help the agent recover. Generic errors lead to generic (bad) retries.

```python
# Bad
raise ToolError("Failed")

# Good  
raise ToolError(
    "Search failed: rate limit exceeded. "
    "Wait 30 seconds before retrying. "
    "Alternatively, reduce max_results or simplify query."
)
```

### 5. Idempotent Where Possible

Calling a tool twice with the same input should produce the same result (or at least not cause side effects). This makes retries safe.

**Naturally idempotent**: search, read, calculate, validate.
**Needs care**: write, send, create, delete — implement idempotency keys or check-before-act patterns.

## Tool Set Design

### Right-Sizing the Tool Set

- **Too few tools**: Agent tries to use tools for purposes they weren't designed for. Or it gives up and hallucinates an answer.
- **Too many tools**: Agent gets confused about which tool to use. Tool selection accuracy drops with more than ~15 tools per agent.

**Sweet spot**: 3-10 tools per agent, each clearly differentiated.

### Tool Overlap

Avoid tools with overlapping purposes. If two tools could handle the same request, the agent will inconsistently choose between them.

**Bad**: `search_google(query)` and `search_web(query)` — agent won't know which to pick.
**Good**: `search_web(query)` and `search_academic(query)` — clearly different domains.

### Composite vs. Atomic Tools

Some operations are best exposed as a single tool even if they involve multiple steps internally.

**Expose as composite** when: the sub-steps are always done together, the intermediate results aren't useful to the agent, and the agent would make the same sequence of calls every time.

**Keep atomic** when: the agent might want to use sub-steps independently, intermediate results inform the agent's next decision, or the sub-steps have different failure modes.

## Common Tool Categories

### Information Retrieval
- Web search, document search, database query
- Keep results concise — return snippets, not full documents
- Include relevance scores to help agents prioritize

### Data Processing  
- Parse, extract, transform, validate
- Always return structured output
- Include confidence/quality scores where applicable

### External Actions
- Send message, create record, update status
- Always require confirmation for irreversible actions
- Return proof of execution (ID, timestamp, status)

### Computation
- Calculate, compare, aggregate
- Use deterministic implementations, not LLM-based
- Include units and precision in output

## Testing Tools

Test tools independently before giving them to agents:

1. **Schema validation**: Does the tool accept and return the correct types?
2. **Edge cases**: Empty input, very long input, special characters, invalid formats.
3. **Error handling**: Does it return helpful errors, not stack traces?
4. **Performance**: Is it fast enough? Agents amplify tool latency.
5. **Agent usability**: Give the tool to an agent with just the name and description — can it figure out how to use it correctly?
