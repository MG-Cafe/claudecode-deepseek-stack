# Agent Tool Usage (Required)

You have access to the **Agent** tool for spawning subagents. You MUST use it. It is your most powerful tool for complex work. Treat it as a first-class tool equal to Read, Write, Bash, and Grep.

## When to Spawn Agents

Spawn an Agent when ANY of these apply:

- Task touches 3+ files → Agent
- Task requires research before implementation → Agent(research), then Agent(implement)
- Task has independent subtasks that can run in parallel → multiple Agent calls in one response
- You need to preserve main context window → Agent offloads the heavy reading
- User says: implement, build, create feature, refactor, follow the plan, multi-file, parallel → Agent

## How to Use the Agent Tool

```
Agent({
  description: "3-5 word summary",
  prompt: "Full context briefing. State the goal, what files to touch, what to produce.",
  subagent_type: "general-purpose"  // or: Explore, Plan, content-writer, etc.
})
```

For parallel independent work, emit MULTIPLE Agent tool calls in a single response.

## Anti-Patterns (Do NOT Do These)

- Do NOT read 10+ files yourself when an Agent could do it → spawn Agent(subagent_type="Explore")
- Do NOT implement multi-file changes inline → spawn Agent with implementation instructions
- Do NOT skip the Agent tool because "it seems simpler to do it directly" — context preservation matters
- Do NOT ask the user "should I use an agent?" — just use it when the criteria above are met

## Available Subagent Types

| Type | Use For |
|------|---------|
| `general-purpose` | Default — research, implementation, multi-step tasks |
| `Explore` | Codebase search, file discovery, "where is X defined" |
| `Plan` | Architecture planning, implementation strategy |
| `content-writer` | High-quality content creation |
| `smart-searcher` | Fast file/skill discovery (runs on cheapest model) |

## Pattern: Research Then Implement

For non-trivial tasks, use two sequential agents:
1. Agent(description="Research X", subagent_type="Explore") → returns findings
2. Read findings, then Agent(description="Implement X", prompt="Based on research: [findings]. Implement...") 

## Pattern: Parallel Independent Work

When subtasks don't depend on each other, spawn all agents in ONE response:
```
// Single response with multiple Agent calls:
Agent({ description: "Update component A", prompt: "..." })
Agent({ description: "Update component B", prompt: "..." })
Agent({ description: "Add tests for C", prompt: "..." })
```

This is faster and preserves your context window.
