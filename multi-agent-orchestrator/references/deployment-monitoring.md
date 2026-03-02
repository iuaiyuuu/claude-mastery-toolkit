# Deployment & Monitoring

Production deployment patterns and observability for multi-agent systems.

## Deployment Architecture

### Synchronous (Request-Response)

Agent pipeline runs within a single request. User waits for completion.

**When to use**: Total latency < 30 seconds. Simple pipelines. Interactive applications.

**Implementation**: Standard API endpoint that orchestrates agents internally.
```python
@app.post("/process")
async def process(request: TaskRequest) -> TaskResponse:
    result = await orchestrator.run(request.task)
    return TaskResponse(result=result)
```

**Timeout strategy**: Set overall request timeout (e.g., 30s). Budget time across agents — if 3 agents each get 10s max, the pipeline fits within 30s.

### Asynchronous (Job Queue)

Submit task, get job ID, poll for results.

**When to use**: Total latency > 30 seconds. Complex multi-step workflows. Background processing.

**Implementation**:
```python
@app.post("/submit")
async def submit(request: TaskRequest) -> JobResponse:
    job_id = await job_queue.enqueue(request.task)
    return JobResponse(job_id=job_id, status="queued")

@app.get("/status/{job_id}")
async def status(job_id: str) -> JobStatus:
    return await job_store.get_status(job_id)
```

**Progress reporting**: Store per-agent completion status. Client can poll and show progress:
```json
{
    "job_id": "abc123",
    "status": "running",
    "progress": {
        "researcher": "completed",
        "writer": "running", 
        "reviewer": "pending"
    }
}
```

### Streaming

Stream partial results as agents complete.

**When to use**: Long-running tasks where users benefit from seeing intermediate results. Chat-like interfaces.

**Implementation**: Use Server-Sent Events (SSE) or WebSockets.
```python
@app.get("/stream/{job_id}")
async def stream(job_id: str):
    async def event_generator():
        async for event in orchestrator.stream(job_id):
            yield f"data: {event.json()}\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

## Observability

### Structured Logging

Every agent invocation must produce a structured log entry. This is non-negotiable for debugging multi-agent systems.

**Required fields per agent call**:
```json
{
    "request_id": "req_abc123",
    "agent_id": "researcher",
    "model": "claude-sonnet-4-20250514",
    "input_tokens": 1523,
    "output_tokens": 487,
    "latency_ms": 2340,
    "status": "success",
    "timestamp": "2025-01-15T10:30:00Z",
    "parent_agent": "orchestrator",
    "tools_used": ["web_search", "extract_content"],
    "retry_count": 0
}
```

### Distributed Tracing

Use a trace ID that follows the request through all agents. Each agent call is a span within the trace.

```
Trace: req_abc123
├── Span: orchestrator (0-8500ms)
│   ├── Span: researcher (100-3200ms)
│   │   ├── Span: web_search tool (150-1800ms)
│   │   └── Span: extract_content tool (1850-3100ms)
│   ├── Span: writer (3300-6100ms)
│   └── Span: reviewer (6200-8400ms)
```

**Tools**: OpenTelemetry, LangSmith, Arize Phoenix, or custom implementation.

### Key Metrics

**Per-agent metrics** (track as time series):
- Latency: p50, p95, p99
- Token consumption: input + output per call
- Error rate: percentage of calls that fail or timeout
- Retry rate: percentage of calls that needed retries
- Tool usage: which tools each agent uses and how often

**System-level metrics**:
- End-to-end latency: total time from request to response
- End-to-end success rate: percentage of requests completed successfully
- Cost per request: total token cost across all agents
- Throughput: requests processed per second/minute
- Queue depth: (async only) number of pending jobs

### Alerting

**Critical alerts** (page on-call):
- End-to-end error rate > 10% over 5 minutes
- Any agent error rate > 25% over 5 minutes
- End-to-end p95 latency > 2x baseline for 10 minutes

**Warning alerts** (notify during business hours):
- Cost per request > 2x baseline for 1 hour
- Any agent's retry rate > 20% over 30 minutes
- Token consumption trending upward (potential prompt injection or infinite loop)

## Error Handling in Production

### Circuit Breaker Pattern

If an agent fails repeatedly, stop calling it temporarily.

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = "closed"  # closed=normal, open=blocking, half-open=testing
        self.last_failure_time = None
    
    async def call(self, func, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError("Agent temporarily disabled")
        
        try:
            result = await func(*args, **kwargs)
            if self.state == "half-open":
                self.state = "closed"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            raise
```

### Graceful Degradation

When parts of the system fail, return partial results rather than failing entirely.

**Strategies**:
- Skip optional agents (e.g., skip "fact-checker" and return unchecked draft)
- Use cached results from previous successful runs
- Fall back to a simpler single-agent approach
- Return partial results with a quality warning

### Cost Controls

Multi-agent systems can burn tokens fast. Implement guardrails:

- **Per-request token budget**: Kill the pipeline if total tokens exceed a threshold.
- **Per-agent token budget**: Limit individual agent consumption.
- **Daily/hourly cost caps**: Disable the system if costs exceed budget.
- **Anomaly detection**: Alert if cost per request suddenly spikes (potential prompt injection or infinite loop).

## Scaling Considerations

- **Horizontal scaling**: Each request is independent — scale by adding more workers.
- **Rate limits**: LLM APIs have rate limits. Implement request queuing with backpressure.
- **Caching**: Cache agent outputs for identical inputs. Especially useful for research/retrieval agents.
- **Model routing**: Use cheaper models for simple agents (routing, formatting) and expensive models for complex agents (reasoning, synthesis).
