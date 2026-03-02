# Framework Guide

Comparison of multi-agent frameworks with selection criteria and implementation guidance.

## Framework Overview

### LangGraph

**What it is**: Graph-based orchestration framework from LangChain. Agents are nodes, edges define flow with conditional routing.

**Best for**: Complex workflows with conditional branching, cycles, and human-in-the-loop. Production systems needing persistence and streaming.

**Strengths**:
- First-class support for cycles and conditional edges
- Built-in state persistence (checkpointing)
- Streaming support for real-time updates
- Human-in-the-loop patterns built in
- Strong typing with TypedDict state
- LangSmith integration for observability

**Weaknesses**:
- Steep learning curve — graph paradigm takes time to internalize
- Verbose for simple pipelines
- Tightly coupled to LangChain ecosystem
- Debugging graph execution can be non-obvious

**When to choose**: Complex workflows with conditional logic, need for persistence/resume, production systems needing observability via LangSmith.

**Key concepts**:
```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    current_step: str
    results: dict

graph = StateGraph(AgentState)
graph.add_node("researcher", research_agent)
graph.add_node("writer", writing_agent)
graph.add_node("reviewer", review_agent)
graph.add_conditional_edges("reviewer", route_after_review, {
    "approved": END,
    "revision_needed": "writer"
})
```

### CrewAI

**What it is**: Role-based multi-agent framework. Define agents with roles, goals, and backstories. Agents collaborate on tasks.

**Best for**: Role-based collaboration scenarios. Quick prototyping of multi-agent workflows. Teams familiar with the "crew" mental model.

**Strengths**:
- Intuitive API — define agents like team members
- Built-in task delegation and collaboration
- Memory and context sharing built in
- Quick to prototype
- Good documentation and examples

**Weaknesses**:
- Less control over execution flow than LangGraph
- Opinionated about agent structure — harder to customize
- Less mature for production deployments
- Debugging inter-agent communication can be opaque

**When to choose**: Rapid prototyping, role-based collaboration, teams wanting quick results without graph theory.

**Key concepts**:
```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Senior Researcher",
    goal="Find comprehensive, accurate information",
    backstory="Expert research analyst with 10 years experience",
    tools=[search_tool, scrape_tool]
)

writer = Agent(
    role="Technical Writer", 
    goal="Create clear, engaging content from research",
    backstory="Award-winning technical writer",
    tools=[write_tool]
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential  # or Process.hierarchical
)
```

### AutoGen (Microsoft)

**What it is**: Conversation-based multi-agent framework. Agents communicate through messages in group chats.

**Best for**: Conversational agent systems, debate/discussion patterns, code generation with execution feedback.

**Strengths**:
- Natural conversation-based interaction
- Built-in code execution sandbox
- Good for human-AI collaboration
- Flexible group chat patterns

**Weaknesses**:
- Conversation-based model can be unpredictable
- Less structured than graph-based approaches
- Harder to enforce strict execution order
- Agent termination conditions need careful tuning

**When to choose**: Conversational multi-agent scenarios, code generation with iterative refinement, research discussions.

### Custom Implementation

**What it is**: Build your own orchestration layer using raw LLM API calls.

**Best for**: Simple patterns (2-3 agents, linear flow), when you need full control, when frameworks add unnecessary complexity.

**Strengths**:
- Full control over every aspect
- No framework dependencies
- Minimal abstraction overhead
- Easiest to debug — you wrote every line

**Weaknesses**:
- You build everything: retries, logging, state management, error handling
- No community patterns to lean on
- Maintenance burden grows with complexity

**When to choose**: Simple 2-3 agent pipelines, teams with strong engineering capacity, when framework overhead isn't justified, learning purposes.

**Key pattern**:
```python
async def orchestrate(task: str) -> str:
    # Step 1: Research
    research = await call_agent(
        system="You are a researcher...",
        message=task,
        output_schema=ResearchResult
    )
    
    # Step 2: Draft
    draft = await call_agent(
        system="You are a writer...",
        message=f"Based on this research: {research.json()}, write...",
        output_schema=Draft
    )
    
    # Step 3: Review
    review = await call_agent(
        system="You are a reviewer...",
        message=f"Review this draft: {draft.json()}",
        output_schema=ReviewResult
    )
    
    if review.approved:
        return draft.content
    else:
        # Retry with feedback
        return await revise(draft, review.feedback)
```

## Selection Decision Tree

```
Is your workflow simple (2-3 agents, linear)?
├── YES → Custom implementation
└── NO → Do you need conditional branching or cycles?
    ├── YES → LangGraph
    └── NO → Do you prefer role-based mental model?
        ├── YES → CrewAI
        └── NO → Do you want conversation-based interaction?
            ├── YES → AutoGen
            └── NO → LangGraph (most flexible)
```

## Framework-Agnostic Best Practices

Regardless of framework choice:

1. **Abstract the framework**: Don't let framework-specific code leak into business logic. Wrap framework calls behind your own interfaces so you can swap frameworks later.

2. **Version your agent prompts**: Treat system prompts like code — version control, review, test.

3. **Centralize configuration**: Agent configs (model, temperature, max_tokens, tools) should be in config files, not hardcoded.

4. **Implement circuit breakers**: If an agent fails repeatedly, stop calling it and use a fallback. Don't burn tokens retrying a broken agent.

5. **Plan for framework migration**: Frameworks evolve fast. Design your system so that swapping frameworks requires changing the orchestration layer, not rewriting agents.
