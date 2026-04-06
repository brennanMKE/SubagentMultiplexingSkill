# cmux-subagent-orchestration

Orchestrate multiple Claude agents in parallel, each with its own terminal pane. Real-time visibility, coordinated workflows, flexible lifecycle management.

## What This Is

A Claude Code skill that enables powerful multi-agent coordination using cmux (terminal multiplexer):
- **Spawn agents in parallel** — each gets its own pane
- **Real-time visibility** — watch what's happening, jump to any pane
- **Flexible coordination** — fan-out, pipeline, work-pool patterns
- **Smart pane management** — auto-cleanup on success, preserve errors for debugging

## Quick Start

### Install

Copy the `cmux-multiplexing` directory to your Claude Code skills folder:
```bash
cp -r cmux-multiplexing ~/.claude/skills/
```

Or manually clone/download and place in `~/.claude/skills/cmux-multiplexing/`.

### Use in Claude Code

The skill triggers automatically when you mention orchestrating agents, running tests in parallel, or managing multiple tasks. You can also explicitly reference it:

```
I need to run 3 test suites in parallel and compare results. Use cmux-subagent-orchestration.
```

### Core Pattern

```typescript
// Spawn multiple agents in parallel
const unitId = spawn_pane("test-unit", "npm run test:unit");
const integId = spawn_pane("test-integration", "npm run test:integration");
const e2eId = spawn_pane("test-e2e", "npm run test:e2e");

// Wait for all
const [unit, integ, e2e] = await Promise.all([
  poll_pane(unitId),
  poll_pane(integId),
  poll_pane(e2eId)
]);

// Synthesize results
const allPassed = [unit, integ, e2e].every(r => r.exit_code === 0);
console.log(allPassed ? "✓ All tests passed" : "✗ Some tests failed");
```

## Documentation

### SKILL.md (start here)
The main skill documentation. Covers:
- Core concept (why parallel panes matter)
- Key principles
- 4 main orchestration workflow patterns
- Full API reference
- Quick tips

**When to read**: First-time users, getting oriented on patterns.

### references/orchestration-guide.md (architectural principles)
Deep dive into the **why** behind multi-agent systems:
- **Isolation and Context** — Why independent panes scale better
- **Orchestration Models** — Fan-out, pipeline, work-pool, background work
- **Error Propagation** — Handling failures gracefully
- **Resource Constraints** — When to fan-out vs. when to reduce parallelism
- **State Synchronization** — How agents communicate (files, APIs, events)
- **Monitoring and Control** — Polling strategies, real-time checks
- **Decision Trees** — Choose patterns based on your task

**When to read**: Designing complex workflows, making architectural trade-offs, scaling to many agents.

### references/subagent-patterns.md (ready-to-use templates)
Six concrete workflow templates with complete code:

1. **Parallel Test Runner** — Run unit, integration, E2E tests simultaneously
2. **Research Sprint** — Multiple agents investigate different aspects in parallel
3. **Code Generation Pipeline** — Analyze → Generate → Test (sequential stages)
4. **Task Queue Dispatcher** — Work-pool pattern for 10+ independent tasks
5. **Background Builder + Foreground Work** — Long compilation while main agent works
6. **Parallel Analysis + Decision** — Multiple agents analyze, main decides

**When to read**: You have a specific workflow in mind, copy the template and adapt.

### references/pane-lifecycle.md (pane management strategies)
Detailed guidance on when to keep/close panes:
- **Quick Decision Matrix** — Lookup table for common scenarios
- **Strategies** — Output capture, error preservation, resource management
- **Lifecycle Policies** — Aggressive cleanup, conservative, balanced (recommended)
- **Resource Considerations** — When pane count matters, file descriptor limits

**When to read**: Debugging pane accumulation, optimizing for resource constraints, deciding lifecycle policies.

## Helper Scripts

Three ready-to-use JavaScript utilities in `scripts/`:

### orchestrate_parallel_tasks.js
Spawn N tasks in parallel, poll all, synthesize results.

```typescript
const { orchestrateParallelTasks } = require('./orchestrate_parallel_tasks');

const tasks = [
  { name: "test-unit", command: "npm run test:unit" },
  { name: "test-integration", command: "npm run test:integration" },
  { name: "test-e2e", command: "npm run test:e2e" }
];

const results = await orchestrateParallelTasks(tasks, {
  timeout: 300,
  closeOnSuccess: true
});
```

### task_queue_dispatcher.js
Maintain a queue, spawn N workers, assign tasks as workers complete.

```typescript
const { dispatchTaskQueue } = require('./task_queue_dispatcher');

const tasks = [
  { id: "feat-1", command: "npm run implement -- --feature auth" },
  { id: "feat-2", command: "npm run implement -- --feature logging" },
  // ... 10+ tasks
];

const results = await dispatchTaskQueue(tasks, { workerCount: 3 });
```

### pane_monitor.js
Real-time dashboard of running panes, error detection, reporting.

```typescript
const { monitorPanes, waitForPaneError } = require('./pane_monitor');

// Live dashboard
await monitorPanes({ duration: 300, interval: 5 });

// Wait for specific error
const failedPane = await waitForPaneError(/timeout|fatal error/, { timeout: 60 });
```

## Common Workflows

### Parallel Testing
Run 3 test suites simultaneously, get results faster.

**Template**: See `references/subagent-patterns.md` → Parallel Test Runner

**Time saved**: ~3x (all tests in parallel wall-clock time)

### Research Sprint
Multiple agents investigate performance, security, architecture, dependencies—all in parallel.

**Template**: See `references/subagent-patterns.md` → Research Sprint

**Time saved**: ~4x (comprehensive assessment in 1/4 the time)

### Code Generation Pipeline
Analyze codebase → generate code → test result. Each stage is an independent agent.

**Template**: See `references/subagent-patterns.md` → Code Generation Pipeline

**Benefit**: Each stage gets accurate context from the previous stage

### Batch Feature Implementation
10 features to implement? Spawn 3 agents, let them work through the queue.

**Template**: See `references/subagent-patterns.md` → Task Queue Dispatcher

**Time saved**: ~3x (10 features with 3 agents in 3-4 cycles)

### Background Build + Foreground Work
Start a 2-minute cargo build in the background. Main agent refactors, documents, reviews code.

**Template**: See `references/subagent-patterns.md` → Background Builder + Foreground Work

**Benefit**: Stay productive while waiting; no idle time

## API Reference

### Core Functions

#### spawn_pane(name, command, cwd?, shell?)
Start a new pane and run a command.

```typescript
const id = spawn_pane("test-unit", "npm run test:unit");
```

#### poll_pane(paneId, timeout_seconds?)
Wait for a pane to complete. Returns exit code, status, elapsed time.

```typescript
const result = poll_pane(id, 300); // wait up to 5 minutes
if (result.exit_code === 0) console.log("✓ Success");
```

#### list_panes()
Get current status of all active panes.

```typescript
const panes = list_panes();
panes.forEach(p => console.log(`${p.name}: ${p.status}`));
```

#### get_pane_output(paneId, lines?)
Fetch recent output without waiting. Useful for checking progress.

```typescript
const output = get_pane_output(id, 20); // last 20 lines
if (output.includes("ERROR")) console.error("Problem detected");
```

#### kill_pane(paneId)
Terminate a pane and its process.

```typescript
kill_pane(id); // stop and cleanup
```

#### send_keys(paneId, keys, enter?)
Send keyboard input to an interactive pane.

```typescript
send_keys(serverId, "C-c"); // Ctrl+C to stop
send_keys(replId, "exit()", true); // type "exit()" and press Enter
```

For complete reference, see `SKILL.md`.

## Architecture Principles

See `references/orchestration-guide.md` for depth. Quick version:

1. **Isolation**: Each agent in its own pane = clean context, independent debugging
2. **Fan-out**: Spawn all in parallel, don't wait between spawns
3. **Coordination**: Agents share state via files/APIs, not tight coupling
4. **Error budgets**: Decide early which failures are tolerable
5. **Pane lifecycle**: Close on success (clean workspace), keep on error (debug info)

## Pane Lifecycle

**Decision tree**:
```
Did agent succeed? 
├─ YES, output captured? 
│  └─ Close pane (clean workspace)
└─ NO (error/timeout)
   └─ Keep pane (preserve context)
```

See `references/pane-lifecycle.md` for detailed strategies.

## Best Practices

1. **Name panes clearly** — `"test-unit"` not `"p1"`. Helps when reading pane list.
2. **Spawn all at once** — Don't spawn A, wait, spawn B. Spawn A and B simultaneously.
3. **Check progress early** — Use `get_pane_output()` to spot errors before final poll.
4. **Fail fast** — If one agent fails and blocks others, kill it immediately.
5. **Save output before closing** — For important tasks, write final output to file before closing pane.
6. **Use helper scripts** — Don't rewrite orchestration logic; use `orchestrate_parallel_tasks.js` or `task_queue_dispatcher.js`.

## Limitations

- **Interactive output**: Some interactive tools may not capture output cleanly
- **Pane count**: OS limits on open files (~1000-4096); with 200+ panes, consider closing aggressively
- **Layout control**: cmux controls layout, not this skill; this skill manages command lifecycle only

## FAQ

**Q: Can I use this without Claude Code?**  
A: No, this skill requires Claude Code and cmux to be installed. The skill spawns subagents, which is a Claude Code feature.

**Q: What if I don't have cmux installed?**  
A: Install it first: `brew install cmux` (macOS) or your distro's package manager.

**Q: Can agents share state?**  
A: Yes, via files (shared output files, JSON queues) or APIs. See `references/orchestration-guide.md` → State Synchronization.

**Q: How many agents can I run simultaneously?**  
A: Depends on your machine's resources. CPU-bound: ~number of cores. Network-bound: 10+. Memory-bound: 2-5. Match your workload.

**Q: What happens if an agent takes too long?**  
A: `poll_pane()` times out. You can then `kill_pane()` and move on, or wait longer. See `references/pane-lifecycle.md` for strategies.

**Q: Can I monitor panes while agents are running?**  
A: Yes, use `list_panes()` and `get_pane_output()` to check progress. Use `pane_monitor.js` for a live dashboard.

## Examples in the Codebase

Check `references/subagent-patterns.md` for six complete, copy-paste-ready examples:
1. Parallel Test Runner
2. Research Sprint
3. Code Generation Pipeline
4. Task Queue Dispatcher
5. Background Builder + Foreground Work
6. Parallel Analysis + Decision

Each includes full code, explanation, and pros/cons.

## Learning Path

1. **New to this?** Read `SKILL.md` (20 min)
2. **Have a specific workflow?** Find the template in `references/subagent-patterns.md` (5 min to adapt)
3. **Designing a complex system?** Read `references/orchestration-guide.md` (30 min for depth)
4. **Pane management questions?** See `references/pane-lifecycle.md` (10 min lookup)
5. **Trouble with pane accumulation?** Consult the lifecycle policies in `references/pane-lifecycle.md`

## Contributing

Ideas for improvements? The skill is designed to be extended:
- Add more helper scripts to `scripts/`
- Add reference docs to `references/`
- Share new workflow templates

## See Also

- [cmux documentation](https://github.com/manaflow-ai/cmux)
- Claude Code documentation (agent spawning, subagents)
- `references/orchestration-guide.md` for architectural deep dives

---

**Made for parallel work.** Spawn, orchestrate, synthesize. Faster coding through multi-agent coordination.
