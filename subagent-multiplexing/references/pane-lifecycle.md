# Pane Lifecycle Management

Detailed strategies for deciding when to keep panes open and when to close them. This is a critical decision that affects workspace cleanliness, debugging ability, and resource usage.

## Quick Decision Matrix

| Scenario | Decision | Why |
|----------|----------|-----|
| Agent succeeded, output captured | **Close** | Frees resources, keeps workspace clean |
| Agent failed or errored | **Keep** | Preserve error info for debugging |
| Agent timed out | **Keep** | Investigate why it hung |
| Background task (build, test) | **Close after polling** | Agent reported result; pane served its purpose |
| Interactive task (REPL, server) | **Keep if interactive** | User may interact; closing kills the session |
| Large/complex output | **Keep pane, also save output to file** | Fallback if file I/O fails; pane is scrollable history |
| Subagent still running | **Don't decide yet** | Poll first to get final status |

## Detailed Strategies

### Strategy 1: Output Capture + Close

When the agent has succeeded and its output is safely captured (e.g., written to a file, returned, or processed).

**When**:
- All test suites passed and you have a test report
- Code generation completed and files are written
- Analysis succeeded and results are in `/tmp/output.json`

**Implementation**:
```typescript
const analysisId = spawn_pane("analyze", `
  node analyze.js --output /tmp/analysis.json
`);

const result = poll_pane(analysisId);

if (result.exit_code === 0) {
  // Verify output file exists
  if (fs.existsSync('/tmp/analysis.json')) {
    const analysis = JSON.parse(fs.readFileSync('/tmp/analysis.json'));
    console.log(`Analysis captured: ${analysis.components.length} components`);
    
    // Output is safe. Close pane.
    kill_pane(analysisId);
  } else {
    // File doesn't exist; keep pane for debugging
    console.error("Output file not found despite exit code 0. Keeping pane open.");
  }
} else {
  // Error; keep pane
  console.error("Analysis failed. Keeping pane open.");
}
```

**Pros**:
- Clean workspace (panes are resources)
- Clear signal: "this task is done and results are safe"

**Cons**:
- If file I/O fails, you've lost the pane output as fallback

### Strategy 2: Error Panes (Keep for Debugging)

When an agent fails, keep the pane open so you can:
- Read error messages
- Manually retry commands
- Explore the working directory

**When**:
- Exit code != 0
- Agent timed out
- Agent produced partial results (e.g., 2/3 tests passed)

**Implementation**:
```typescript
const testId = spawn_pane("tests", "npm test");
const result = poll_pane(testId);

if (result.exit_code !== 0) {
  console.error(`Tests failed (exit code ${result.exit_code})`);
  console.log("Pane kept open. Jump to 'tests' pane to inspect output.");
  // Don't kill_pane
} else {
  console.log("Tests passed!");
  kill_pane(testId);
}
```

**User experience**:
- User sees "Tests failed. Jump to 'tests' pane to inspect."
- User can open the pane, scroll through output, re-run failed tests manually
- User closes the pane when done debugging

**Pro**:
- Preserve full history and terminal state for manual debugging

**Con**:
- Workspace accumulates panes over time; user must manage cleanup

### Strategy 3: Long-Running Tasks (Close After Polling)

For background tasks (long compilation, test run), the pane is just a progress indicator. Once you've polled the final result, the pane is no longer needed.

**When**:
- Long-running build or test suite
- Agent spawned in background while main work continues
- Agent completed successfully and result is in memory

**Implementation**:
```typescript
const buildId = spawn_pane("build", "cargo build --release");

// Main agent does other work...
await refactorCode();

// When ready, poll build
const buildResult = poll_pane(buildId, 120);

if (buildResult.exit_code === 0) {
  console.log(`✓ Build succeeded in ${buildResult.elapsed_seconds}s`);
  kill_pane(buildId); // close; result is captured
} else {
  console.log(`✗ Build failed. Keeping pane open for inspection.`);
  // keep pane
}
```

**Rationale**:
- `poll_pane()` returns the full result (status, exit code, output)
- Once you have the result in memory, the pane is redundant for successful tasks
- For failures, you keep the pane to preserve error state

### Strategy 4: Interactive Tasks (Conditional Close)

Interactive tasks (REPL, dev server, SSH session) should stay open if the user might interact with them.

**When**:
- Started a `npm run dev` server
- Opened an interactive REPL
- Spawned a long-lived daemon

**Implementation**:
```typescript
const serverId = spawn_pane("dev-server", "npm run dev");

// Server is running interactively
console.log("Dev server started in 'dev-server' pane. User can interact.");
console.log("Press Ctrl+C in the pane to stop the server.");

// Don't poll or close; server runs indefinitely

// Later, if you need to shut down:
send_keys(serverId, "C-c"); // graceful shutdown
const shutdownResult = poll_pane(serverId, 10);
kill_pane(serverId);
```

**Rationale**:
- Interactive tasks are long-lived by design
- User may jump to the pane and run commands
- Polling/closing would interrupt the workflow

### Strategy 5: Balanced Approach (Safe Default)

A practical strategy that balances cleanliness and debuggability:

1. **On success**: Close immediately
2. **On failure**: Keep for debugging
3. **On timeout**: Keep (time to investigate hangs)
4. **For background tasks**: Close after polling (result is captured)

**Implementation**:
```typescript
async function runAgentAndDecideLifecycle(paneId, taskDescription) {
  const result = poll_pane(paneId, 300);
  
  if (result.exit_code === 0) {
    console.log(`✓ ${taskDescription} succeeded.`);
    kill_pane(paneId);
  } else if (result.status === "timeout") {
    console.log(`⏱ ${taskDescription} timed out. Keeping pane open.`);
    // keep pane
  } else {
    console.log(`✗ ${taskDescription} failed. Keeping pane open for inspection.`);
    // keep pane
  }
}

// Usage:
const id1 = spawn_pane("task-1", "npm test");
await runAgentAndDecideLifecycle(id1, "Tests");

const id2 = spawn_pane("task-2", "build.sh");
await runAgentAndDecideLifecycle(id2, "Build");
```

**Rationale**:
- **Success → Close**: Reduce clutter
- **Failure/timeout → Keep**: Preserve debugging info
- Simple logic, easy to understand and maintain

---

## Pane Lifecycle Over Time

```
Spawn                    Poll                         Decision
  ↓                       ↓                             ↓
  │ [running]            │ [completed]                 │
  │ pane is live ─────→ [result ready] ───────→ [keep or kill?]
  │                      │                             │
  └──────────────────────┘                             ├─→ Success + output captured → KILL
                                                       ├─→ Failure → KEEP
                                                       ├─→ Timeout → KEEP
                                                       └─→ Interactive task → KEEP

Once closed (KILL):
  • Resources freed (file descriptors, memory)
  • Output lost (can't scroll back)
  • Process terminated

Once kept (KEEP):
  • Pane remains live (can interact)
  • Output preserved in scrollback
  • Resources held until manually closed
```

---

## Resource Management

### When to Care About Pane Count

**Rule of thumb**: Close panes aggressively if you're spawning 10+.

**Why**:
- Each pane uses OS file descriptors (~3-5 per pane)
- Typical OS limit: 1024-4096 open files per process
- 10 panes: ~50 FDs. Acceptable.
- 100 panes: ~500 FDs. Still OK but getting tight.
- 200+ panes: Risk hitting OS limit. Consider closing idle panes.

### Scrollback Memory

By default, each pane keeps ~100KB of output in memory (configurable).

- 10 panes: ~1MB scrollback. Negligible.
- 100 panes: ~10MB scrollback. Acceptable.
- 500+ panes: ~50MB. Not huge, but worth closing.

**To reduce scrollback**:
```bash
export CMUX_CLAUDE_MAX_OUTPUT=50000  # 50KB per pane instead of 100KB
```

### When Pane Count Matters

You're spawning 50+ panes? Consider:

1. **Closing successful panes immediately** (don't keep them open)
2. **Batching work** (dispatch 5 workers with 10 tasks each, not 50 panes)
3. **Reducing scrollback** (see above)

---

## Pane Lifecycle and Error Investigation

### Preserving Error Context

When an agent fails, the pane contains:
- Last command run
- Exit code and signal (if killed)
- Error messages
- Partial output (if interrupted)
- Working directory state

**To investigate**:
1. Open the pane (press `p` in pane list)
2. Scroll up to see the full error
3. Manually re-run the command if needed
4. Check the working directory (`ls`, `pwd`)
5. Once done, close the pane (`exit` or Ctrl+D)

### Batching Error Inspection

If multiple panes failed, you can't view all simultaneously (only one pane in focus).

**Strategy**: Capture output before closing.

```typescript
const failedPanes = [pane1, pane2, pane3];

for (const paneId of failedPanes) {
  const output = get_pane_output(paneId, 100); // last 100 lines
  fs.writeFileSync(`/tmp/error-${paneId}.log`, output);
}

// Now you have all error logs. Review them:
for (const paneId of failedPanes) {
  const log = fs.readFileSync(`/tmp/error-${paneId}.log`, 'utf-8');
  console.log(`=== Error in ${paneId} ===`);
  console.log(log);
}
```

---

## Lifecycle Policies (Template)

### Policy: "Aggressive Cleanup"
**Best for**: Workspaces that spawn many panes (e.g., task queue with 50+ tasks).

```typescript
// Kill all panes immediately on success
// Keep only explicitly error-marked panes

const successfulPanes = [];
const failedPanes = [];

for (const id of paneIds) {
  const result = poll_pane(id);
  if (result.exit_code === 0) {
    successfulPanes.push(id);
    kill_pane(id); // immediate cleanup
  } else {
    failedPanes.push(id);
    // keep for debugging
  }
}

console.log(`${successfulPanes.length} successful (closed), ${failedPanes.length} failed (open)`);
```

**Pros**: Clean workspace, minimal resources.  
**Cons**: Can't manually inspect successful panes after completion.

### Policy: "Conservative (Keep All)"
**Best for**: High-value tasks where you want to preserve context.

```typescript
// Keep all panes open regardless of outcome
// Let user decide when to close

const panes = list_panes();
console.log(`${panes.length} panes still open. Review at your leisure.`);
```

**Pros**: Maximum debuggability.  
**Cons**: Workspace accumulates panes; user must manually clean up.

### Policy: "Balanced (Recommended)"
**Best for**: Most workflows.

```typescript
// Close on success (output is captured)
// Keep on error (preserve context)
// Keep on timeout (investigate hangs)

for (const id of paneIds) {
  const result = poll_pane(id);
  if (result.exit_code === 0) {
    kill_pane(id);
  } else {
    console.log(`Pane ${id} failed. Keeping open for inspection.`);
  }
}
```

**Pros**: Good balance of cleanliness and debuggability.  
**Cons**: Some panes stay open; requires user awareness.

---

## When Pane Lifecycle Decisions Change

### Scenario: Success, but Needed Later

```
T0: Spawn analysis agent → completes successfully → Close pane
T1: (5 minutes later) Realize you need to tweak the analysis

Problem: Pane is closed. Can't inspect or re-run.

Solution: Before closing, save complete output.

// Better:
const analysisId = spawn_pane("analyze", "node analyze.js");
const result = poll_pane(analysisId);

if (result.exit_code === 0) {
  // Save full output before closing
  const output = get_pane_output(analysisId, 10000); // all available output
  fs.writeFileSync('/tmp/analysis-output.log', output);
  kill_pane(analysisId);
}
```

### Scenario: Kept Pane Accumulation

```
T0: Spawn 10 agents, 3 fail, 7 succeed
    → 3 panes are open (success panes closed, failures kept)

T1-10: Manually review the 3 failures, fix issues, re-run

T15: Now you have 15 panes open (original 3 + 3 re-runs + 9 new agents)

Problem: Workspace is crowded. Hard to find the active pane.

Solution: Periodically clean up. Either:
- Use a naming convention ("old-attempt-1", "retry-1") and manually close old ones
- After successful re-run, close the old failure pane
- Close panes explicitly in batches once you're confident in your fixes
```

---

## Summary Checklist

Before spawning an agent, decide its lifecycle:

- [ ] Will output be captured (file, return value, memory)?
- [ ] If it fails, how long will I need the pane open?
- [ ] How many panes will I have total? (if 50+, close aggressively)
- [ ] Is this interactive (user might interact)?
- [ ] Do I need output history if I close the pane? (save to file first if yes)

Then implement:

```typescript
const id = spawn_pane(name, command);
const result = poll_pane(id);

// Decide
if (result.exit_code === 0 && outputCaptured) {
  kill_pane(id); // clean up
} else if (result.exit_code !== 0) {
  console.log("Failed. Pane kept for debugging.");
  // keep open
} else {
  // other cases...
}
```

See `orchestration-guide.md` and `subagent-patterns.md` for more examples.
