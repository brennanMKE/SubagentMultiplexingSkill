# Orchestration Guide: Multi-Agent Systems with cmux

This guide explains the **architectural principles** behind effective multi-agent orchestration. Read this when designing complex workflows or trying to decide whether to fan-out or pipeline your agents.

## Table of Contents
1. [Isolation and Context](#isolation-and-context)
2. [Orchestration Models](#orchestration-models)
3. [Error Propagation](#error-propagation)
4. [Resource Constraints](#resource-constraints)
5. [State Synchronization](#state-synchronization)
6. [Monitoring and Control](#monitoring-and-control)
7. [Decision Trees](#decision-trees)

---

## Isolation and Context

### Why Independent Panes Matter

Each pane is a **independent shell environment**. This is a feature, not a limitation.

**Benefits:**
- **Clean context**: One agent doesn't pollute another's namespace (environment vars, shell state)
- **Focused debugging**: If agent A fails, you inspect its pane in isolation; agent B is unaffected
- **Scaling**: You can add more agents without exponential state complexity
- **True parallelism**: Each pane is a real OS process; they run simultaneously on multi-core systems

**Tradeoff:**
- **Shared state requires explicit coordination**: If agent A generates a file and agent B needs it, you must explicitly pass the file path or rely on the filesystem as a coordination point

### Mental Model: Agents as Microservices

Think of agents like microservices. Each is:
- **Autonomous**: Has its own context, can make decisions independently
- **Loosely coupled**: Doesn't need to know about the others' internals
- **Communicates via contracts**: File paths, return values, exit codes

This model scales better than tightly-coupled, synchronous execution.

---

## Orchestration Models

### Fan-Out / Fan-In (Embarrassingly Parallel)

**When**: Multiple independent tasks that don't depend on each other.

```
Main Agent
├─ Spawn Agent A (task 1)
├─ Spawn Agent B (task 2)
├─ Spawn Agent C (task 3)
└─ Poll all → Synthesize results
```

**Best for:**
- Running 3 test suites in parallel
- Multiple code analysis checks (security, style, performance)
- Parallel research (agent investigates performance, another security, another architecture)

**Advantages:**
- **Throughput**: N tasks in ~1/(N) the time (if all take similar duration)
- **Fault isolation**: One agent fails, others complete
- **Simple mental model**: Easy to implement and debug

**Pitfall to avoid:**
- Don't spawn agents one-by-one with polling in between. Spawn all first, then poll all. Otherwise you lose parallelism.

```typescript
// ❌ WRONG (serial)
for (let task of tasks) {
  const id = spawn_pane(`task-${i}`, cmd);
  const result = poll_pane(id); // waits before next spawn!
}

// ✅ RIGHT (parallel)
const ids = tasks.map((cmd, i) => spawn_pane(`task-${i}`, cmd));
const results = ids.map(id => poll_pane(id)); // poll all in parallel
```

### Pipeline (Dependency Chain)

**When**: Output of stage N is input to stage N+1.

```
Main Agent
├─ Spawn Agent A (analysis)
├─ Poll A → parse results
├─ Spawn Agent B (code generation, receives A's output)
├─ Poll B → validate output
└─ Spawn Agent C (testing) → finalize
```

**Best for:**
- Analyze codebase → generate migration → test migration
- Parse requirements → design architecture → implement
- Explore data → clean data → analyze → report

**Advantages:**
- **Each stage has accurate context**: Agent B knows what Agent A found before running
- **Fail fast**: If stage 2 fails, no point running stage 3
- **Workflow clarity**: Easy to understand dependencies

**Pitfall to avoid:**
- Making stages too small. If each stage is "run one command," you'll have overhead from spawning many panes. Consider batching related commands into one agent.

### Background Work + Main Agent

**When**: One long-running task doesn't block main workflow.

```
Main Agent
├─ Spawn Agent A (long-running: compilation, testing)
├─ While A runs, continue:
│  ├─ Code review
│  ├─ Refactoring
│  └─ Documentation
└─ When ready, poll A for final result
```

**Best for:**
- `cargo build --release` taking 2 minutes while you refactor
- Long-running test suite while you implement the next feature
- Data processing while you prepare the analysis report

**Advantages:**
- **Productivity**: No waiting around
- **Context switching**: You stay engaged instead of idle

**Pitfall to avoid:**
- Don't assume the background task completes successfully. Always poll before using its result.

### Task Queue / Work Pool

**When**: Variable number of small, independent tasks; spawn worker agents on-demand.

```
Main Agent (Dispatcher)
├─ Load task queue: [task-1, task-2, ..., task-10]
├─ Spawn 3 worker agents
├─ Distribute: task-1 to Worker-A, task-2 to Worker-B, task-3 to Worker-C
├─ Poll for first completion
├─ Assign task-4 to (first available worker)
└─ Repeat until queue is empty
```

**Best for:**
- Implementing 10 similar features (each agent takes 1-2)
- Batch processing (10 files to analyze, spawn 3 agents in rotation)
- Scaling to variable workload (if you have 100 tasks, use 5 workers; if 10 tasks, use 2)

**Advantages:**
- **Adaptive parallelism**: Match worker count to workload
- **Easier than fixed pipelines**: Each worker does the same thing
- **Scalable**: Add/remove workers without rewriting logic

**Pitfall to avoid:**
- Make sure each task is truly independent. If task-3 depends on output of task-1, use Pipeline instead.
- Don't spawn more workers than tasks. If you have 3 tasks and spawn 5 workers, you're wasting resources.

---

## Error Propagation

### Local Failure (One Agent Fails)

**Scenario**: Agent B fails; Agents A and C are still running.

**Decision tree:**
1. Is Agent B's failure blocking the main task?
   - **YES**: Kill A and C, investigate B, pivot
   - **NO**: Let A and C finish, handle B's failure independently
2. Can you salvage partial results from B?
   - **YES**: Fetch B's pane output, extract usable parts, continue
   - **NO**: Abort B entirely, move on

**Example**:
```typescript
const [unitResult, integResult, e2eResult] = await Promise.all([
  poll_pane(unitId),
  poll_pane(integId),
  poll_pane(e2eId)
]);

if (unitResult.exit_code !== 0) {
  // Unit tests failed—this is bad, probably abort
  console.log("Unit tests failed. Aborting.");
  kill_pane(integId);
  kill_pane(e2eId);
} else if (integResult.exit_code !== 0) {
  // Integration tests failed, but units passed
  console.log("Integration tests failed. Units passed. Proceeding with caution.");
}
```

### Cascading Failure (Multi-Stage Pipeline)

**Scenario**: Stage 2 depends on Stage 1; Stage 1 fails.

**Strategy**: Don't spawn Stage 2 if Stage 1 failed. This prevents wasted work and cleaner error messages.

```typescript
const stage1Id = spawn_pane("stage-1-analyze", analyzeCmd);
const stage1Result = poll_pane(stage1Id);

if (stage1Result.exit_code !== 0) {
  console.log("Analysis failed. Not proceeding to code generation.");
  return; // stop here
}

// Safe to proceed
const stage2Id = spawn_pane("stage-2-generate", generateCmd);
const stage2Result = poll_pane(stage2Id);
```

### Error Budgets

**Concept**: Plan how many failures you can tolerate.

- **Testing**: If 1 of 3 test suites fails, continue but flag it. Usually acceptable.
- **Code generation**: If code gen fails, abort. Not acceptable to proceed.
- **Analysis**: If one analysis fails, continue with others. Aggregate results that succeeded.

Ask yourself: **"If agent X fails, can I still deliver value?"** If yes, continue on failure. If no, stop.

---

## Resource Constraints

### When to Reduce Parallelism

**Memory-bound agents**: Each agent loads a large dataset into memory. Running 5 simultaneously causes OOM.

**Solution**: Pipeline instead of fan-out, or use work-pool with fewer workers.

**CPU-bound agents**: Parallel agents doing heavy computation. With 8 cores, 10 agents just thrash the scheduler.

**Solution**: Match worker count to core count. Spawn 8 agents, not 10.

**Network-bound agents**: Multiple agents making API calls. API has rate limits.

**Solution**: Serialize or throttle, or negotiate rate-limit handling with the API.

### Pane Overhead

Each pane consumes OS resources (file descriptors, memory for shell state, scrollback buffer).

**Typical limits**:
- Most systems: 1000+ open file descriptors per session
- Scrollback memory: ~100KB per pane (configurable)

**When it matters**: You're spawning 100+ panes. At that point, consider:
- Reducing scrollback (`export CMUX_CLAUDE_MAX_OUTPUT=50000`)
- Killing panes aggressively after polling
- Using work-pool pattern (max N panes alive at once, not 100)

### Network-Bound vs CPU-Bound vs I/O-Bound

**Network-bound agents** (API calls, file downloads):
- Spawn liberally (10+ agents). Network latency dominates; multiple agents leverage waiting time.
- Example: 10 agents each making 1-second API calls. In parallel: ~1 second. Serially: ~10 seconds.

**CPU-bound agents** (analysis, compilation):
- Spawn cautiously. Spawn ~(number of cores) agents.
- Example: 8-core system. 8 agents doing heavy computation. More than 8 just context-switches.

**I/O-bound agents** (reading/writing files):
- Spawn moderately. Disk throughput is often shared.
- Example: 4 agents writing large files to same disk. More than 4 may thrash disk.

---

## State Synchronization

### Using Files as Contracts

Agents can't directly call each other. They communicate via **files and exit codes**.

**Pattern: One agent writes output, next reads it**

```typescript
// Agent A: Analyze
const analyzeId = spawn_pane("analyzer", `
  node analyze.js > /tmp/analysis.json
`);
const analyzeResult = poll_pane(analyzeId);

if (analyzeResult.exit_code !== 0) {
  console.log("Analysis failed. Not proceeding.");
  return;
}

// Agent B: Read A's output and generate code
const analysis = JSON.parse(fs.readFileSync("/tmp/analysis.json"));
const generateId = spawn_pane("generator", `
  node generate.js --input /tmp/analysis.json --output /tmp/generated.js
`);
const generateResult = poll_pane(generateId);
```

**Benefits**: No tight coupling. Agents don't need to know about each other; they just read/write files.

### Using APIs for State

For more complex coordination (e.g., work queue, shared metrics), use a lightweight API or database.

**Pattern: Redis or file-based queue**

```typescript
// Dispatcher writes task queue to file
fs.writeFileSync("/tmp/task-queue.json", JSON.stringify(tasks));

// Each worker reads queue, claims a task, updates queue
// (or uses Redis for atomic operations)
```

**When to use**: If you're coordinating 5+ agents doing related work. Otherwise, files are simpler.

### Locking and Atomicity

**Scenario**: Multiple agents reading/writing the same file.

**Risk**: Race conditions (Agent A reads, Agent B modifies, Agent A writes stale data).

**Solution**:
- Assign each agent a unique output file. Merge at the end.
- Use a database/API with atomic transactions (Redis, etc.)
- Use OS-level file locking (careful and complex)

**Simpler approach**: Give each agent a unique scope.

```typescript
// ❌ Risky: multiple agents writing same file
const results = spawn_pane("agent-1", "process input.json > output.json");
const results2 = spawn_pane("agent-2", "process input.json >> output.json");

// ✅ Safe: each agent owns its output
const id1 = spawn_pane("agent-1", "process input.json > output-1.json");
const id2 = spawn_pane("agent-2", "process input.json > output-2.json");
// Then: merge output-1.json and output-2.json yourself
```

---

## Monitoring and Control

### Polling Strategy

**Option 1: Poll all at once**
```typescript
const [r1, r2, r3] = await Promise.all([
  poll_pane(id1, 300),
  poll_pane(id2, 300),
  poll_pane(id3, 300)
]);
```
**Pros**: Simple, all agents wait until slowest finishes.
**Cons**: Longest duration is max(agent durations).

**Option 2: Poll as available (early completion)**
```typescript
const ids = [id1, id2, id3];
const results = [];
while (ids.length > 0) {
  const panes = list_panes();
  const completed = ids.filter(id => {
    const p = panes.find(pane => pane.id === id);
    return p && p.status !== "running";
  });
  for (const id of completed) {
    results.push(poll_pane(id, 0)); // don't wait, already done
    ids.splice(ids.indexOf(id), 1);
  }
  // small sleep before next poll
}
```
**Pros**: React to early completions; don't wait for slowest.
**Cons**: More complex; requires polling in a loop.

### Checking in on Running Panes

Don't wait blind. Use `get_pane_output()` to spot-check for errors:

```typescript
// Spawn all panes
const ids = [spawn_pane(...), spawn_pane(...), spawn_pane(...)];

// While waiting, check for obvious failures
for (let i = 0; i < 10; i++) {
  await sleep(10000); // check every 10 seconds
  for (const id of ids) {
    const output = get_pane_output(id, 20); // last 20 lines
    if (output.includes("FATAL") || output.includes("panic")) {
      console.log(`Pane ${id} has error. Killing related work.`);
      kill_pane(id);
      // maybe kill siblings too
    }
  }
}
// Now poll for real completion
const results = ids.map(id => poll_pane(id));
```

---

## Decision Trees

### Choosing an Orchestration Model

```
Q: Do the tasks depend on each other?
├─ NO → Fan-out/fan-in
│  └─ Can they run in parallel?
│     ├─ YES → Spawn all at once, poll all
│     └─ NO → Queue/work-pool (serialize within workers)
│
└─ YES, linear chain (A→B→C)
   └─ Pipeline (spawn A, wait, spawn B, wait, spawn C)
   
Q: One task is much longer than others?
├─ YES → Background work + main agent
│  └─ Keep main agent active while long task runs
│
└─ NO → Fan-out or pipeline
```

### Choosing Pane Lifecycle (keep open or close?)

```
Q: Did the agent complete successfully?
├─ YES
│  ├─ Was the result captured (output saved, result returned)?
│  │  ├─ YES → Close pane (clean workspace)
│  │  └─ NO → Keep pane (fallback for inspection)
│  │
│  └─ Is this a long-running background task (server, REPL)?
│     ├─ YES → Ask user (or keep if interactive)
│     └─ NO → Close pane
│
└─ NO (error or timeout)
   └─ Keep pane (debugging info is valuable)
```

---

## Real-World Example: Comprehensive Code Review

Workflow:
1. **Analyzer agent**: Read codebase, generate architecture report
2. **3 review agents** (in parallel): Security, performance, style
3. **Synthesizer agent**: Aggregate reports, produce final review

```
Dispatcher
├─ Pane 1: Analyzer → architecture.json
├─ (wait for Analyzer)
├─ Pane 2-4: 3 reviewers (parallel, input: architecture.json)
├─ (wait for all 3)
└─ Pane 5: Synthesizer (input: all 3 reviews) → final-report.md
```

**Code sketch**:
```typescript
// Stage 1: Analyze
const analyzeId = spawn_pane("analyze", "node analyze.js > /tmp/arch.json");
await poll_pane(analyzeId);

// Stage 2: Parallel reviews (fan-out)
const reviewIds = [
  spawn_pane("review-security", "node review-security.js < /tmp/arch.json"),
  spawn_pane("review-perf", "node review-perf.js < /tmp/arch.json"),
  spawn_pane("review-style", "node review-style.js < /tmp/arch.json")
];
await Promise.all(reviewIds.map(id => poll_pane(id)));

// Stage 3: Synthesize
const synthesizeId = spawn_pane("synthesize", "node synthesize.js > /tmp/final-review.md");
await poll_pane(synthesizeId);

console.log("Review complete. See /tmp/final-review.md");
```

---

## Summary

- **Independent agents in panes** = scalable, fault-isolated, readable
- **Choose orchestration model based on dependencies**: fan-out (parallel), pipeline (sequential), work-pool (variable load), background (long-running)
- **Coordinate via files/APIs**, not shared memory
- **Error handling**: Decide early what failures are tolerable
- **Monitoring**: Use `get_pane_output()` to spot-check while waiting
- **Pane lifecycle**: Close successful tasks (clean workspace), keep errors (debug info)

More questions? See `subagent-patterns.md` for concrete templates, or `pane-lifecycle.md` for detailed lifecycle strategies.
