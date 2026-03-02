# Architecture Patterns

Detailed guide for each multi-agent orchestration pattern with trade-offs and implementation guidance.

## Sequential Pipeline

```
[Agent A] → [Agent B] → [Agent C] → [Output]
```

**When to use**: Linear workflows where each step depends on the previous step's output. Examples: research → summarize → draft → review; extract → transform → validate → load.

**Strengths**: Simple to implement, easy to debug (just check each stage's output), deterministic execution order, straightforward error handling (fail at the stage that broke).

**Weaknesses**: No parallelism — total latency is sum of all stages. A bottleneck at any stage blocks everything downstream. Not suitable for tasks where sub-task order isn't fixed.

**Implementation notes**:
- Each agent's output schema must match the next agent's input schema exactly. Validate at boundaries.
- Implement checkpointing: save intermediate results so you can resume from the last successful stage on retry.
- Consider adding a "gate" between stages — a lightweight check that verifies the output is sane before passing it on.

**Error handling**: On failure at stage N, you have options: retry stage N, restart from stage N-1 (if non-determinism helps), or fail the entire pipeline. Decision depends on whether the stage is deterministic and whether upstream stages have side effects.

## Supervisor/Worker (Router)

```
        ┌→ [Worker A] →┐
[Supervisor] → [Worker B] → [Supervisor] → [Output]
        └→ [Worker C] →┘
```

**When to use**: Tasks where the type of work varies per input. The supervisor analyzes the input, decides which workers to invoke, and synthesizes results. Examples: customer support routing, multi-tool task execution, adaptive research.

**Strengths**: Flexible — can handle diverse inputs. Workers are specialized and simpler. Supervisor acts as quality control.

**Weaknesses**: Supervisor is a single point of failure and a bottleneck. Supervisor quality determines system quality. If the supervisor misroutes, everything downstream is wrong.

**Implementation notes**:
- Supervisor prompt is critical. It needs clear routing criteria, not vague instructions.
- Use structured output for the supervisor's routing decision: `{worker: "research", reason: "...", params: {...}}`.
- Consider using a cheaper/faster model for routing, expensive model for workers.
- Limit the number of worker invocations per request to control cost and latency.

**Error handling**: If a worker fails, the supervisor can retry with the same or different worker, adjust parameters, or return a partial result with explanation. The supervisor should have explicit instructions for each failure mode.

## Parallel Fan-out/Fan-in

```
        ┌→ [Agent A] →┐
[Split] →→ [Agent B] →→ [Aggregate] → [Output]
        └→ [Agent C] →┘
```

**When to use**: Independent sub-tasks that can run concurrently. Examples: multi-source search, parallel analysis from different perspectives, batch processing.

**Strengths**: Significant latency reduction through parallelism. Natural fit for embarrassingly parallel tasks.

**Weaknesses**: Aggregation is hard — combining outputs from multiple agents into coherent results requires careful design. All agents must complete before aggregation (or you need timeout handling).

**Implementation notes**:
- Split function must produce truly independent sub-tasks. If agents need each other's output, this isn't the right pattern.
- Aggregation agent/function is where complexity lives. Options: simple concatenation, weighted voting, LLM-based synthesis, or structured merge.
- Set per-agent timeouts. If one agent is slow, you can either wait, skip it, or return partial results.
- Consider result quality variance — parallel agents may produce outputs of different quality.

**Error handling**: If K of N agents fail, decide policy: require all (fail if any fails), require majority, or best-effort (return whatever succeeded). Make this configurable.

## Hierarchical

```
[Orchestrator]
├── [Team Lead A]
│   ├── [Worker A1]
│   └── [Worker A2]
└── [Team Lead B]
    ├── [Worker B1]
    └── [Worker B2]
```

**When to use**: Very complex tasks with natural hierarchy. The top level decomposes into major work streams, each managed independently. Examples: large software projects, comprehensive research reports, complex planning tasks.

**Strengths**: Scales to very complex tasks. Each level only needs to understand its direct reports. Mirrors how human organizations work.

**Weaknesses**: Most complex to implement and debug. Deep hierarchies add latency (each level is a full LLM call). Information loss at each level. Expensive in tokens.

**Implementation notes**:
- Limit depth to 3 levels maximum. Beyond that, information loss and latency become prohibitive.
- Each level should have clear authority boundaries — what can a team lead decide vs. what must escalate to the orchestrator?
- Implement progress reporting: lower levels report status upward so the orchestrator can detect stalls.
- Use summarization at each level to manage context — don't forward raw outputs up the hierarchy.

**Error handling**: Hierarchical error propagation. Workers report to team leads, team leads report to orchestrator. Each level decides: retry, reassign, escalate, or accept partial result.

## Debate/Adversarial

```
[Proposer A] ←→ [Proposer B]
       ↓              ↓
      [Judge] → [Output]
```

**When to use**: High-stakes decisions where verification matters. Code review, fact-checking, complex reasoning where you want to catch errors. Also useful for creative tasks where diversity of thought helps.

**Strengths**: Catches errors that single-agent approaches miss. Produces more robust outputs. Natural quality assurance mechanism.

**Weaknesses**: Expensive — at minimum 3x the cost of single agent. Slow — multiple rounds of debate add latency. Agents can converge on the same wrong answer (especially if using the same model). Can be adversarial in unproductive ways.

**Implementation notes**:
- Use different system prompts (or ideally different models) for proposers to get genuine diversity.
- Limit debate rounds (2-3 is usually sufficient). Diminishing returns beyond that.
- Judge should have clear evaluation criteria, not just "pick the better one."
- Consider structured critique: require proposers to identify specific flaws, not general disagreement.
- Track convergence — if proposers agree after round 1, skip further rounds.

**Error handling**: If proposers never converge, the judge makes the final call. If the judge can't decide, escalate to a human or return both options with the judge's analysis.

## Swarm

**When to use**: Rarely. Only when tasks are truly dynamic and unpredictable, agent capabilities are heterogeneous, and workload distribution changes at runtime. Examples: large-scale content moderation with varying content types, dynamic customer service with routing based on real-time agent performance.

**Strengths**: Highly adaptive. Can handle unpredictable workloads. Self-organizing.

**Weaknesses**: Hardest to implement, debug, and reason about. Non-deterministic behavior. Requires sophisticated coordination mechanisms. Often over-engineered — simpler patterns usually suffice.

**Implementation notes**:
- Implement capability advertisement: each agent publishes what it can do. A matchmaker assigns tasks based on capabilities and availability.
- Use message queues for task distribution. Agents pull tasks they're capable of handling.
- Implement health checks and circuit breakers. Remove unhealthy agents from the pool.
- This is essentially building a distributed system. Apply distributed systems principles: eventual consistency, idempotency, graceful degradation.

**Recommendation**: Unless you're building a platform that serves many different task types at scale, you almost certainly don't need this. Start with supervisor/worker and only move to swarm if supervisor becomes the bottleneck.
