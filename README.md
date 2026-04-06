# subagent-multiplexing

Orchestrate multiple subagent tasks in parallel using cmux panes. Create visible panes, send commands to them, monitor output live, coordinate results. All tasks run concurrently while main agent stays responsive.

Teaches orchestration patterns: fan-out/fan-in, pipelines, background workers, work queues.

## The Problem

By default, Claude Code subagents run sequentially:
```
Spawn Agent 1 → Wait (can't do anything) → Get results 
→ Spawn Agent 2 → Wait (can't do anything) → Get results
→ Spawn Agent 3 → Wait (can't do anything) → Get results

Total time: 15 minutes (serial)
Main agent: Blocked the entire time
```

## The Solution

**Spawn all agents at once. They run in parallel panes. Main agent stays responsive.**

```
Spawn Agent 1 (pane 1) ↘
Spawn Agent 2 (pane 2) → All run in parallel
Spawn Agent 3 (pane 3) ↗

Total time: ~5 minutes (wall-clock)
Main agent: Free to do other work, monitor progress, coordinate results
```

## Core Concept

The main agent calls the `Agent` tool multiple times **without waiting**. Each subagent:
- Runs in its own cmux pane (visible, monitorable)
- Works independently in parallel
- Completes at its own pace
- Result comes back to main agent when done

Think of it like delegating work: "I'm spawning 3 researchers. They'll report back when done. Meanwhile, I'll organize the final synthesis."

## Quick Start

```bash
# Spawn 3 agents in parallel panes (all at once, no waiting)
cmux new-pane -n "Performance Analysis" "python profile_code.py"
cmux new-pane -n "Security Audit" "python security_scan.py"
cmux new-pane -n "Architecture Review" "python review_architecture.py"

# All three agents now run simultaneously in visible panes
# You can see them in: cmux list-panes

# Monitor progress live
cmux read-screen --pane pane:1 --lines 30
cmux read-screen --pane pane:2 --lines 30
cmux read-screen --pane pane:3 --lines 30

# When ready, collect results and synthesize
cmux read-screen --pane pane:1 > /tmp/perf_result
cmux read-screen --pane pane:2 > /tmp/security_result
cmux read-screen --pane pane:3 > /tmp/arch_result

# Analyze all three results
cat /tmp/*_result | synthesize.sh
```

Each agent runs in its own visible pane. You monitor with cmux commands. All run concurrently.

## Core Patterns

1. **Fan-Out/Fan-In** — Spawn N agents in parallel, collect all results, synthesize
   - Best for: Parallel testing, analysis, parallel research
   - Time: ~time of 1 task (instead of N serial)

2. **Pipeline** — Spawn A, wait, spawn B with A's output, wait, spawn C
   - Best for: Analysis → generation → testing (dependencies)
   - Benefit: Each stage has exact output from previous

3. **Background Worker** — Spawn long task, continue with other work
   - Best for: Builds, long tests, deployments (while you code)
   - Benefit: Zero idle time

4. **Work Queue** — Distribute M tasks to N workers as they complete
   - Best for: Batch processing, feature implementation
   - Time: ~M/N (instead of M serial)

See `subagent-multiplexing/SKILL.md` for detailed cmux command examples for each pattern.

## How It Works

1. **Create panes with `cmux new-pane`** — Spawn N panes with agent commands in them
   ```bash
   cmux new-pane -n "Agent 1" "command_1"
   cmux new-pane -n "Agent 2" "command_2"
   ```

2. **Agents run in parallel** — All panes execute simultaneously and independently

3. **Monitor with cmux commands** — Watch progress live
   ```bash
   cmux list-panes              # See all running panes
   cmux read-screen --pane X    # Read output from a pane
   ```

4. **Collect results** — When agents complete, read their output
   ```bash
   cmux read-screen --pane pane:1 > /tmp/result1
   ```

5. **Synthesize** — Analyze all results and decide next steps

## Best Practices

### 1. Give panes descriptive names

```bash
# ✅ Clear
cmux new-pane -n "Unit Tests" "npm test:unit"
cmux new-pane -n "Integration Tests" "npm test:integration"

# ❌ Vague
cmux new-pane -n "Test 1" "npm test"
```

Clear names appear in `cmux list-panes` and help you track what's running.

### 2. Spawn all panes quickly, then monitor

```bash
# ✅ Good: All spawn immediately (no waiting between them)
cmux new-pane -n "A" "cmd_a"
cmux new-pane -n "B" "cmd_b"
cmux new-pane -n "C" "cmd_c"

# Then monitor with cmux commands
cmux list-panes
cmux read-screen --pane pane:1
```

### 3. Use cmux commands to monitor and collect

```bash
# Monitor live
cmux read-screen --pane pane:1 --lines 30

# Collect results when done
cmux read-screen --pane pane:1 > /tmp/result1
cmux read-screen --pane pane:2 > /tmp/result2
cmux read-screen --pane pane:3 > /tmp/result3

# Synthesize all results
cat /tmp/result* | analyze.sh
```

### 4. One pane failing doesn't block others

```bash
# Spawn A, B, C in parallel
# If A exits with error, B and C keep running
# Collect all results (even if A failed)

cmux read-screen --pane pane:1 > /tmp/result_a
cmux read-screen --pane pane:2 > /tmp/result_b
cmux read-screen --pane pane:3 > /tmp/result_c

# Synthesize all three
cat /tmp/result_* | analyze.sh
```

## Sandbox Constraints

When subagents run in isolated contexts, they may have limited write permissions to your project.

**Workaround**: Have agents write to temp directories, then main agent copies to final location.

See `references/sandbox-constraints.md` for details and working patterns.

## Patterns & Templates

See the `references/` directory:

- **orchestration-guide.md** — Deep architectural patterns and principles
- **subagent-patterns.md** — Ready-to-use templates
- **pane-lifecycle.md** — Strategies for managing pane visibility/cleanup
- **sandbox-constraints.md** — Handling isolated agent I/O

## Next Steps

1. Read `QUICKSTART.md` for step-by-step integration
2. Review `subagent-multiplexing/SKILL.md` for patterns and concepts
3. Check `subagent-multiplexing/references/` for deep architectural guidance
4. Look at the pattern templates in `references/subagent-patterns.md` for your use case

## Troubleshooting

### Panes not appearing?

Check that cmux is running and you're using the right workspace:

```bash
cmux current-workspace    # Check active workspace
cmux list-panes           # List all panes in current workspace
cmux tree                 # See full workspace hierarchy
```

### How do I watch what an agent is doing live?

Use `cmux read-screen` to check output anytime:

```bash
# One-time check
cmux read-screen --pane pane:1 --lines 20

# Watch continuously (tail-like)
watch -n 1 'cmux read-screen --pane pane:1 --lines 20'
```

### How do agents share data?

Use files or environment variables. The main session reads/writes files that agents can access:

```bash
# Agent writes to file
cmux new-pane -n "Agent" "python script.py > /tmp/output.json"

# Main session reads when ready
cat /tmp/output.json | jq .
```

### What if a pane closes unexpectedly?

The pane stays visible in `cmux list-panes` and you can read its final output with `cmux read-screen`. Use that output to diagnose what went wrong.

## Architecture

```
Your Code
├── README.md (this file)
├── QUICKSTART.md (getting started)
├── LICENSE (MIT)
├── .gitignore
├── install.sh (installation script)
└── subagent-multiplexing/
    ├── SKILL.md (core skill - patterns, concepts, API)
    ├── INDEX.md (navigation & quick lookup)
    └── references/
        ├── orchestration-guide.md (architectural patterns)
        ├── subagent-patterns.md (ready-to-use templates)
        ├── pane-lifecycle.md (pane management)
        └── sandbox-constraints.md (handling isolated I/O)
```

## Key Insights

1. **`cmux new-pane` is your core tool** — Creates a visible pane for each agent. That's how agents become parallelizable.
2. **Parallelism is cheap** — Spawn panes liberally. Serial waiting for one agent to finish before spawning the next is expensive.
3. **Visibility matters** — Panes let you see what's happening. Use `cmux read-screen` and `cmux list-panes` to monitor.
4. **Coordination is via files** — Agents write results to `/tmp/`. Main session reads them and decides next steps.
5. **Fail gracefully** — One pane's failure doesn't affect others. Spawn all, collect all, synthesize partial results.

## Next Steps

1. Read `QUICKSTART.md` for a step-by-step integration guide
2. Browse `examples/` for working implementations
3. Review `references/orchestration-guide.md` for deeper patterns
4. Build your first multi-agent system (start with 2-3 agents)

## See Also

- `QUICKSTART.md` — Integration guide
- `subagent-multiplexing/SKILL.md` — Full reference
- `subagent-multiplexing/references/` — Deep documentation
- `examples/` — Working code

## License

MIT
