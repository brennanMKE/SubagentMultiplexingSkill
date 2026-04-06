# Documentation Index

Quick navigation for the cmux-subagent-orchestration skill.

## By Question / Use Case

**"I'm new. Where do I start?"**
→ `SKILL.md` (overview + API) → `README.md` (getting started)

**"I need to run tests in parallel"**
→ `references/subagent-patterns.md` → "Parallel Test Runner" (copy-paste template)

**"I need to do X, Y, Z in parallel but they're all different"**
→ `references/subagent-patterns.md` → "Parallel Analysis + Decision" (template for arbitrary tasks)

**"I have 10+ tasks to distribute among 3 workers"**
→ `references/subagent-patterns.md` → "Task Queue Dispatcher" (work-pool pattern)

**"I need analyzer → generator → tester (sequential stages)"**
→ `references/subagent-patterns.md` → "Code Generation Pipeline"

**"I'm waiting for a long compilation. How do I work in parallel?"**
→ `references/subagent-patterns.md` → "Background Builder + Foreground Work"

**"I need to research performance, security, and architecture simultaneously"**
→ `references/subagent-patterns.md` → "Research Sprint"

**"I don't understand when to fan-out vs. pipeline vs. work-pool"**
→ `references/orchestration-guide.md` → "Orchestration Models" section

**"How do I coordinate agents if they need to share data?"**
→ `references/orchestration-guide.md` → "State Synchronization"

**"What do I do if an agent fails?"**
→ `references/orchestration-guide.md` → "Error Propagation"

**"Should I close this pane or keep it open?"**
→ `references/pane-lifecycle.md` → "Quick Decision Matrix"

**"I'm spawning 50+ panes and things are slow"**
→ `references/pane-lifecycle.md` → "Resource Management"

**"I want live monitoring of all my agents"**
→ `scripts/pane_monitor.js` (live dashboard)

**"I want a ready-to-use orchestration function"**
→ `scripts/orchestrate_parallel_tasks.js` (fan-out/fan-in helper) or `scripts/task_queue_dispatcher.js` (work-pool helper)

---

## By Topic

### Concepts & Architecture
- **Overview**: `SKILL.md` → "Core Concept"
- **Orchestration patterns**: `SKILL.md` → "Workflows" (quick version) OR `references/orchestration-guide.md` (detailed)
- **When to use this skill**: `README.md` → "What This Is" and `SKILL.md` description
- **Isolation principle**: `references/orchestration-guide.md` → "Isolation and Context"
- **Scaling agents**: `references/orchestration-guide.md` → "Resource Constraints"

### API Reference
- **Full API**: `SKILL.md` → "API Reference"
- **Quick lookup**: `README.md` → "API Reference" (summarized)
- **Function details**: `SKILL.md` (complete documentation per function)

### Workflow Templates
- **All templates**: `references/subagent-patterns.md` (6 complete examples)
- **Parallel execution**: `references/subagent-patterns.md` → "Parallel Test Runner", "Research Sprint", "Parallel Analysis + Decision"
- **Sequential execution**: `references/subagent-patterns.md` → "Code Generation Pipeline"
- **Work distribution**: `references/subagent-patterns.md` → "Task Queue Dispatcher"
- **Background work**: `references/subagent-patterns.md` → "Background Builder + Foreground Work"

### Decision Making
- **Choose orchestration model**: `references/orchestration-guide.md` → "Decision Trees"
- **Choose pane lifecycle**: `references/pane-lifecycle.md` → "Quick Decision Matrix"
- **Error handling strategy**: `references/orchestration-guide.md` → "Error Propagation"
- **Resource planning**: `references/orchestration-guide.md` → "Resource Constraints"

### Practical Implementation
- **Helper scripts**: `scripts/` directory (3 utilities)
- **Avoid mistakes**: `references/subagent-patterns.md` → "Common Mistakes to Avoid"
- **Best practices**: `README.md` → "Best Practices"

### Troubleshooting
- **Pane accumulation**: `references/pane-lifecycle.md` → "When Pane Count Matters"
- **Failed agents**: `references/pane-lifecycle.md` → "Preserving Error Context" or `references/orchestration-guide.md` → "Error Propagation"
- **Monitoring agents**: `scripts/pane_monitor.js` (live dashboard) or `references/orchestration-guide.md` → "Monitoring and Control"
- **Resource limits**: `references/pane-lifecycle.md` → "Resource Management"

---

## File Structure

```
cmux-multiplexing/
├── SKILL.md                              # Main skill doc (overview + API)
├── README.md                             # Getting started + quick reference
├── INDEX.md                              # This file
├── references/
│   ├── orchestration-guide.md            # Deep architectural principles
│   ├── subagent-patterns.md              # 6 ready-to-use templates
│   └── pane-lifecycle.md                 # Pane management strategies
└── scripts/
    ├── orchestrate_parallel_tasks.js     # Fan-out/fan-in helper
    ├── task_queue_dispatcher.js          # Work-pool helper
    └── pane_monitor.js                   # Live monitoring dashboard
```

---

## Quick Lookup Table

| I want to... | File | Section |
|---|---|---|
| Get started | `README.md` | Quick Start |
| Understand the concept | `SKILL.md` | Core Concept + Key Principles |
| See code examples | `references/subagent-patterns.md` | Pick a pattern |
| Learn design principles | `references/orchestration-guide.md` | Orchestration Models |
| Handle errors | `references/orchestration-guide.md` | Error Propagation |
| Manage panes | `references/pane-lifecycle.md` | Quick Decision Matrix |
| Read the API | `SKILL.md` | API Reference |
| Use a helper | `scripts/` | Pick a script |
| Avoid mistakes | `references/subagent-patterns.md` | Common Mistakes |
| Monitor agents | `scripts/pane_monitor.js` | (built-in) |
| Scale to 50+ tasks | `references/subagent-patterns.md` | Task Queue Dispatcher |

---

## Reading Paths

### Path 1: 30-Minute Crash Course
1. `README.md` (5 min)
2. `SKILL.md` → "Core Concept" + "Key Principles" + "Workflows" (10 min)
3. `references/subagent-patterns.md` → pick one template and read (10 min)
4. Skim `references/pane-lifecycle.md` → "Quick Decision Matrix" (5 min)

### Path 2: Deep Understanding (90 minutes)
1. `README.md` (5 min)
2. `SKILL.md` (15 min)
3. `references/orchestration-guide.md` (30 min)
4. `references/subagent-patterns.md` (20 min)
5. `references/pane-lifecycle.md` (15 min)
6. Review `scripts/` (5 min)

### Path 3: "Just Give Me a Template" (5 minutes)
1. `references/subagent-patterns.md`
2. Find your use case
3. Copy code, adapt, go

### Path 4: Specific Problem Solving
- "Tests are slow" → `references/subagent-patterns.md` → "Parallel Test Runner"
- "Agent failed" → `references/orchestration-guide.md` → "Error Propagation"
- "Too many panes" → `references/pane-lifecycle.md` → "Resource Management"
- "Not sure what to do" → `references/orchestration-guide.md` → "Decision Trees"

---

## Related Topics (Outside This Skill)

- **Claude Code subagents**: [Claude Code docs](https://claude.com/claude-code)
- **cmux terminal multiplexer**: [cmux GitHub](https://github.com/manaflow-ai/cmux)
- **Terminal multiplexing in general**: tmux, screen, iterm2 docs (background knowledge)

---

## Glossary

**Pane**: A separate terminal window managed by cmux. Each agent gets its own pane.

**Spawn**: Create a new pane and run a command in it.

**Poll**: Wait for a pane to complete and get the result.

**Fan-out**: Spawn N independent agents in parallel.

**Fan-in**: Wait for all N agents and combine results.

**Pipeline**: Spawn agent A, wait, spawn agent B with A's output, etc.

**Work-pool**: Maintain a queue of tasks and dispatch to N workers.

**Orchestrate**: Coordinate multiple agents (decide who runs, when, and what to do with results).

**Pane lifecycle**: Whether to close a pane (clean workspace) or keep it (preserve debugging info).

---

## Version Info

**Skill**: cmux-subagent-orchestration  
**Last Updated**: 2026-04-06  
**Status**: Production-ready

---

## Getting Help

- **Quick questions**: Check the "Quick Lookup Table" above
- **Can't find what you need**: Search by keyword in the INDEX
- **Want templates**: See `references/subagent-patterns.md`
- **Architectural questions**: See `references/orchestration-guide.md`
- **Pane/resource issues**: See `references/pane-lifecycle.md`
