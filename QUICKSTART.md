# QUICKSTART: Spawn Subagents with cmux Panes

End-to-end guide: spawn multiple Claude subagents, each with its own pane, monitor live, aggregate results.

**Time**: 10-15 minutes to understand, 5 minutes to adapt to your problem.

## Prerequisites

- Claude Code CLI installed
- cmux installed (`brew install cmux`)
- Understand what subagents are (see "What are Subagents?" below)

## What Are Subagents?

A subagent is a separate Claude session spawned from your main session. It:
- **Runs independently** in parallel with other agents
- **Has its own context** (no interference with other agents)
- **Solves one focused problem** (e.g., "analyze performance", "review security")

You spawn a subagent using Claude Code's `-a` flag:
```bash
claude -a my-agent-name
```

## The Problem We're Solving

Without multiplexing:
```
Spawn Agent 1 → [invisible, waiting] → Get results
Spawn Agent 2 → [invisible, waiting] → Get results
Spawn Agent 3 → [invisible, waiting] → Get results
Total time: 15 minutes (serial)
Visibility: Zero
```

With multiplexing:
```
Spawn Agent 1 → pane-1 (live output) ─┐
Spawn Agent 2 → pane-2 (live output) ─┼─ Monitor all live
Spawn Agent 3 → pane-3 (live output) ─┘ Get results in parallel
Total time: ~5 minutes (3x faster)
Visibility: Real-time, jump to any pane
```

## Core Pattern (5 Steps)

### Step 1: Define Your Subagents

Each subagent is an independent task. Define them as objects:

```typescript
const agents = [
  {
    name: "research-performance",
    prompt: "Analyze the performance bottlenecks in src/core/...",
    // or use a command if you have a script
    // command: "claude -a research-performance"
  },
  {
    name: "research-security",
    prompt: "Review for security vulnerabilities in src/core/...",
  },
  {
    name: "research-architecture",
    prompt: "Map the architecture and design patterns...",
  }
];
```

### Step 2: Spawn Subagents in Panes

For each agent, create a pane and run the subagent in it:

```typescript
const paneIds = {};

for (const agent of agents) {
  const command = `claude -a ${agent.name} << 'EOF'
${agent.prompt}
EOF`;

  console.log(`Spawning ${agent.name}...`);
  paneIds[agent.name] = spawn_pane(agent.name, command);
}

console.log(`Spawned ${agents.length} agents. Panes created.`);
```

**What's happening**:
- `spawn_pane()` creates a new terminal window (cmux pane)
- The command runs a Claude subagent with the given prompt
- Each agent's output goes to its pane (live, visible)
- All agents run in parallel (not waiting for each other)

### Step 3: Monitor Progress Live

While agents are working, you can monitor them:

```typescript
// Check progress every 10 seconds
let active = true;
let iteration = 0;

while (active) {
  iteration++;
  const panes = list_panes();
  const agentPanes = panes.filter(p => agents.some(a => a.name === p.name));
  
  console.log(`\n[${iteration}] Agent status:`);
  for (const pane of agentPanes) {
    const status = pane.status === 'running' ? '▶' : pane.status === 'done' ? '✓' : '✗';
    console.log(`  ${status} ${pane.name}: ${pane.elapsed_seconds}s`);
  }
  
  // Check if all done
  active = agentPanes.some(p => p.status === 'running');
  
  if (active) {
    await new Promise(resolve => setTimeout(resolve, 10000)); // wait 10s
  }
}
```

You can also **jump to any pane** in your terminal to see live output:
```
Press: p (pane list) → arrow keys to select → Enter to jump
```

### Step 4: Collect Results

Once all agents complete, gather their outputs:

```typescript
const results = {};

for (const [agentName, paneId] of Object.entries(paneIds)) {
  const paneResult = poll_pane(paneId, 0); // don't wait, they're done
  
  const output = get_pane_output(paneId, 100); // last 100 lines
  
  results[agentName] = {
    status: paneResult.exit_code === 0 ? 'success' : 'failed',
    duration: paneResult.elapsed_seconds,
    output: output.substring(0, 500) // first 500 chars
  };
  
  // Clean up pane (optional; keep if you want to inspect)
  if (paneResult.exit_code === 0) {
    kill_pane(paneId);
  }
}

console.log("Results:", JSON.stringify(results, null, 2));
```

### Step 5: Synthesize and Act

Combine results and decide what to do next:

```typescript
const allSucceeded = Object.values(results).every(r => r.status === 'success');

if (allSucceeded) {
  console.log("✓ All agents succeeded!");
  
  // Combine findings
  const summary = {
    performance: results['research-performance'].output,
    security: results['research-security'].output,
    architecture: results['research-architecture'].output
  };
  
  // Save to file for reference
  fs.writeFileSync('/tmp/research-findings.json', JSON.stringify(summary, null, 2));
  
  // Next step: analyze findings, make decisions, etc.
  console.log("Findings saved to /tmp/research-findings.json");
} else {
  console.log("✗ Some agents failed. Check panes for errors.");
  // Keep failed panes open for debugging
}
```

## Complete Example: Research Sprint

Here's the full pattern in one piece:

```typescript
async function researchSprint() {
  // Step 1: Define agents
  const agents = [
    {
      name: "research-perf",
      prompt: `Analyze the performance characteristics of the codebase.
Focus on: bottlenecks, slow algorithms, memory usage.
Report findings as JSON with structure: { findings: [...], severity: 'low'|'medium'|'high' }`
    },
    {
      name: "research-security",
      prompt: `Review the codebase for security vulnerabilities.
Focus on: input validation, injection points, access control.
Report findings as JSON with structure: { vulnerabilities: [...], severity: 'low'|'medium'|'critical' }`
    },
    {
      name: "research-architecture",
      prompt: `Map the system architecture and design patterns.
Focus on: component structure, dependencies, scalability implications.
Report findings as JSON with structure: { patterns: [...], recommendations: [...] }`
    }
  ];

  console.log(`Starting research sprint with ${agents.length} agents...`);

  // Step 2: Spawn all agents in parallel
  const paneIds = {};
  for (const agent of agents) {
    const command = `claude -a ${agent.name} << 'EOF'
${agent.prompt}
EOF`;
    paneIds[agent.name] = spawn_pane(agent.name, command);
    console.log(`  ✓ Spawned ${agent.name}`);
  }

  console.log(`\nAll agents spawned. Monitoring progress...`);

  // Step 3: Monitor progress
  let remainingTime = 600; // 10 minutes max
  while (remainingTime > 0) {
    const panes = list_panes();
    const agentPanes = panes.filter(p => agents.some(a => a.name === p.name));
    
    const running = agentPanes.filter(p => p.status === 'running').length;
    const done = agentPanes.filter(p => p.status === 'done').length;
    const errors = agentPanes.filter(p => p.status === 'error').length;
    
    console.log(`Progress: Running: ${running}, Done: ${done}, Errors: ${errors}`);
    
    if (running === 0) {
      break; // All done
    }
    
    await new Promise(resolve => setTimeout(resolve, 10000)); // check every 10 seconds
    remainingTime -= 10;
  }

  console.log(`\nCollecting results...`);

  // Step 4: Collect results
  const results = {};
  for (const [agentName, paneId] of Object.entries(paneIds)) {
    const paneResult = poll_pane(paneId, 0);
    results[agentName] = {
      status: paneResult.exit_code === 0 ? 'success' : 'failed',
      duration: paneResult.elapsed_seconds,
      output: get_pane_output(paneId, 50) // last 50 lines
    };
    
    if (results[agentName].status === 'success') {
      kill_pane(paneId); // clean up
    }
  }

  // Step 5: Synthesize
  console.log(`
    ═══════════════════════════════════════════════════════════
    RESEARCH SPRINT COMPLETE
    ═══════════════════════════════════════════════════════════`);
  
  for (const [agentName, result] of Object.entries(results)) {
    const status = result.status === 'success' ? '✓' : '✗';
    console.log(`    ${status} ${agentName}: ${result.duration}s`);
  }
  
  console.log(`    ═══════════════════════════════════════════════════════════`);

  // Save findings
  fs.writeFileSync('/tmp/research-findings.json', JSON.stringify(results, null, 2));
  console.log(`\n    Findings saved to /tmp/research-findings.json`);

  return results;
}

// Run it
researchSprint().catch(console.error);
```

## Orchestration Patterns

The research sprint above is a **fan-out/fan-in** pattern. Here are other common patterns:

### Pattern 1: Fan-Out / Fan-In (Parallel)
Spawn N independent agents → Wait for all → Combine results.

**When**: All agents are independent (research, testing, analysis).

**Time**: ~(max single agent duration)

**Example**: 3 researchers take 5 min each → all done in ~5 min (not 15).

**Template**:
```typescript
// Spawn all
const ids = agents.map(a => spawn_pane(a.name, a.command));

// Wait for all
await Promise.all(ids.map(id => poll_pane(id)));

// Combine
const results = {};
for (const id of ids) {
  const output = get_pane_output(id, 100);
  results[id] = output;
}
```

### Pattern 2: Pipeline (Sequential)
Spawn agent A → Wait → Spawn agent B with A's output → Wait → Spawn agent C.

**When**: Output of one agent feeds into the next (analyze → generate → test).

**Time**: Sum of all agent durations

**Example**: Analyze (5 min) → Generate (3 min) → Test (2 min) = 10 min total.

**Template**:
```typescript
// Agent A
const aId = spawn_pane("analyzer", analyzeCmd);
const aResult = poll_pane(aId);
if (aResult.exit_code !== 0) throw new Error("Analysis failed");

// Agent B (uses A's output)
const bId = spawn_pane("generator", generateCmd + " --input /tmp/analysis.json");
const bResult = poll_pane(bId);
if (bResult.exit_code !== 0) throw new Error("Generation failed");

// Agent C (uses B's output)
const cId = spawn_pane("tester", testCmd + " --input /tmp/generated.js");
const cResult = poll_pane(cId);
```

### Pattern 3: Work-Pool (Batch Processing)
Spawn N workers, distribute M tasks as workers complete.

**When**: 10+ independent tasks (e.g., implement 10 features, process 20 files).

**Time**: ~(M tasks / N workers)

**Example**: 10 features, 3 workers → done in ~3-4 cycles.

**Template**: See `cmux-multiplexing/scripts/task_queue_dispatcher.js`.

## Adapting to Your Problem

### Scenario 1: I have 3 different agents (research, review, test)

Use **Fan-Out / Fan-In**:
```typescript
const agents = [
  { name: "researcher", prompt: "..." },
  { name: "reviewer", prompt: "..." },
  { name: "tester", prompt: "..." }
];
// Follow the "Complete Example" above
```

### Scenario 2: Agent B depends on Agent A's output

Use **Pipeline**:
```typescript
// Run A
const aId = spawn_pane("agent-a", cmdA);
const aResult = poll_pane(aId);

// Run B with A's output
const bId = spawn_pane("agent-b", cmdB + " --analysis /tmp/a-output.json");
const bResult = poll_pane(bId);
```

### Scenario 3: I have 10+ tasks to distribute

Use **Work-Pool**:
```typescript
const { dispatchTaskQueue } = require('./cmux-multiplexing/scripts/task_queue_dispatcher.js');

const tasks = [
  { id: "feat-1", command: "claude -a implement-feature --feature auth" },
  { id: "feat-2", command: "claude -a implement-feature --feature logging" },
  // ... 10+ tasks
];

const results = await dispatchTaskQueue(tasks, { workerCount: 3 });
```

## Common Mistakes

❌ **Spawning agents one-by-one with waits in between**
```typescript
// WRONG - serial, slow
const a = spawn_pane("a", cmdA);
await poll_pane(a);
const b = spawn_pane("b", cmdB);
await poll_pane(b);
```

✅ **Spawn all, then wait all**
```typescript
// RIGHT - parallel
const a = spawn_pane("a", cmdA);
const b = spawn_pane("b", cmdB);
await Promise.all([poll_pane(a), poll_pane(b)]);
```

---

❌ **Forgetting to check exit code**
```typescript
// WRONG - crashes if agent failed
const result = poll_pane(id);
const data = JSON.parse(get_pane_output(id, 100));
```

✅ **Check exit code first**
```typescript
// RIGHT
const result = poll_pane(id);
if (result.exit_code !== 0) {
  console.error("Agent failed");
  return; // or retry
}
const data = JSON.parse(get_pane_output(id, 100));
```

---

❌ **Not naming panes descriptively**
```typescript
// WRONG
spawn_pane("p1", cmd);
spawn_pane("p2", cmd);
```

✅ **Use descriptive names**
```typescript
// RIGHT
spawn_pane("research-perf", cmd);
spawn_pane("research-sec", cmd);
```

## Next Steps

1. **Copy the "Complete Example"** above
2. **Adapt the agent prompts** to your problem
3. **Run it** and watch the panes live
4. **For complex systems**, read `cmux-multiplexing/references/orchestration-guide.md`

## More Patterns & Templates

See `cmux-multiplexing/references/subagent-patterns.md` for 6 complete templates:
1. Parallel Test Runner
2. Research Sprint
3. Code Generation Pipeline
4. Task Queue Dispatcher
5. Background Builder + Foreground Work
6. Parallel Analysis + Decision

## API Reference (Quick)

**Spawn a pane:**
```typescript
const paneId = spawn_pane(name, command, cwd);
```

**Wait for completion:**
```typescript
const result = poll_pane(paneId, timeoutSeconds);
// result: { status, exit_code, output, elapsed_seconds }
```

**Get live output:**
```typescript
const output = get_pane_output(paneId, numLines);
```

**List all panes:**
```typescript
const panes = list_panes();
// [ { id, name, status, elapsed_seconds, ... } ]
```

**Close a pane:**
```typescript
kill_pane(paneId);
```

**Send keyboard input (for interactive panes):**
```typescript
send_keys(paneId, "C-c"); // Ctrl+C
send_keys(paneId, "exit()", true); // type and Enter
```

Full reference: `cmux-multiplexing/SKILL.md`.

## Questions?

- **"How do I pass data between agents?"** → `cmux-multiplexing/references/orchestration-guide.md` → "State Synchronization"
- **"What if an agent fails?"** → `cmux-multiplexing/references/orchestration-guide.md` → "Error Propagation"
- **"Should I close the pane or keep it?"** → `cmux-multiplexing/references/pane-lifecycle.md`
- **"Can I scale to 50+ tasks?"** → Yes, use work-pool pattern (see `cmux-multiplexing/scripts/task_queue_dispatcher.js`)

---

**Ready?** Copy the complete example, run it, adapt to your problem. You'll have a parallel agent system in minutes.
