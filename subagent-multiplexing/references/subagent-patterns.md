# Subagent Orchestration Patterns

Ready-to-use templates for common multi-agent workflows. Copy, adapt, and extend these patterns for your needs.

## 1. Parallel Test Runner

Spawn unit, integration, and E2E tests simultaneously. Collect results and synthesize a report.

**When to use**: Testing a change that affects multiple layers. You want fast feedback on all test suites.

**Pane layout**:
```
├─ test-unit
├─ test-integration
└─ test-e2e
```

**Template**:
```typescript
async function runTestsInParallel() {
  // Spawn all test suites in parallel
  const unitTestId = spawn_pane("test-unit", "npm run test:unit");
  const integTestId = spawn_pane("test-integration", "npm run test:integration");
  const e2eTestId = spawn_pane("test-e2e", "npm run test:e2e");

  // Poll all in parallel
  const [unitResult, integResult, e2eResult] = await Promise.all([
    poll_pane(unitTestId, 300),
    poll_pane(integTestId, 300),
    poll_pane(e2eTestId, 300)
  ]);

  // Synthesize results
  const results = {
    unit: { passed: unitResult.exit_code === 0, duration: unitResult.elapsed_seconds },
    integration: { passed: integResult.exit_code === 0, duration: integResult.elapsed_seconds },
    e2e: { passed: e2eResult.exit_code === 0, duration: e2eResult.elapsed_seconds }
  };

  const allPassed = Object.values(results).every(r => r.passed);
  const totalTime = Math.max(...Object.values(results).map(r => r.duration));

  console.log(`
    ═══════════════════════════════════
    TEST SUMMARY
    ═══════════════════════════════════
    Unit Tests:      ${results.unit.passed ? '✓' : '✗'} (${results.unit.duration}s)
    Integration:     ${results.integration.passed ? '✓' : '✗'} (${results.integration.duration}s)
    E2E Tests:       ${results.e2e.passed ? '✓' : '✗'} (${results.e2e.duration}s)
    ─────────────────────────────────────
    Overall:        ${allPassed ? '✓ ALL PASSED' : '✗ SOME FAILED'}
    Total time:     ${totalTime}s (wall-clock)
  `);

  // Clean up
  if (allPassed) {
    kill_pane(unitTestId);
    kill_pane(integTestId);
    kill_pane(e2eTestId);
  }
  // If any failed, leave panes open for debugging

  return allPassed;
}
```

**Key ideas**:
- Spawn all at once, don't wait between spawns
- Use `Promise.all()` to wait for all simultaneously
- Synthesize results into a clear report
- Only close panes if all passed (keep error panes for debugging)

---

## 2. Research Sprint (Multi-Agent Investigation)

Multiple agents investigate different aspects of a codebase in parallel: performance, security, architecture, dependencies.

**When to use**: You're onboarding to a new codebase or need a comprehensive assessment before major changes.

**Pane layout**:
```
├─ research-perf
├─ research-security
├─ research-arch
└─ research-deps
```

**Template**:
```typescript
async function researchCodebase() {
  // Spawn 4 research agents in parallel
  const perfId = spawn_pane("research-perf", `
    node -e "console.log('Analyzing performance...');
    /* your perf analysis script */
    fs.writeFileSync('/tmp/perf-report.json', JSON.stringify(report));
    console.log('Perf analysis complete.');"
  `);
  
  const securityId = spawn_pane("research-security", `
    node -e "console.log('Scanning for security issues...');
    /* your security analysis script */
    fs.writeFileSync('/tmp/security-report.json', JSON.stringify(report));
    console.log('Security scan complete.');"
  `);
  
  const archId = spawn_pane("research-arch", `
    node -e "console.log('Mapping architecture...');
    /* your architecture analysis script */
    fs.writeFileSync('/tmp/arch-report.json', JSON.stringify(report));
    console.log('Architecture map complete.');"
  `);
  
  const depsId = spawn_pane("research-deps", `
    node -e "console.log('Analyzing dependencies...');
    /* your dependency analysis script */
    fs.writeFileSync('/tmp/deps-report.json', JSON.stringify(report));
    console.log('Dependency analysis complete.');"
  `);

  // Poll all
  const [perfResult, secResult, archResult, depsResult] = await Promise.all([
    poll_pane(perfId, 600),
    poll_pane(securityId, 600),
    poll_pane(archId, 600),
    poll_pane(depsId, 600)
  ]);

  // Load and summarize results
  const reports = {
    perf: JSON.parse(fs.readFileSync('/tmp/perf-report.json')),
    security: JSON.parse(fs.readFileSync('/tmp/security-report.json')),
    architecture: JSON.parse(fs.readFileSync('/tmp/arch-report.json')),
    dependencies: JSON.parse(fs.readFileSync('/tmp/deps-report.json'))
  };

  // Generate executive summary
  console.log(`
    ═══════════════════════════════════
    RESEARCH SPRINT RESULTS
    ═══════════════════════════════════
    Performance:  ${reports.perf.issues_found} issues
    Security:     ${reports.security.issues_found} vulnerabilities
    Architecture: ${reports.architecture.patterns_found} key patterns
    Dependencies: ${reports.dependencies.outdated_count} outdated packages
    ═══════════════════════════════════
  `);

  // Agents succeeded, close panes
  kill_pane(perfId);
  kill_pane(securityId);
  kill_pane(archId);
  kill_pane(depsId);

  return reports;
}
```

**Key ideas**:
- Each agent writes its output to a separate JSON file (no race conditions)
- All agents run in parallel, even if investigation takes 10 minutes
- Results are combined into an executive summary
- Panes are closed after successful completion

---

## 3. Code Generation Pipeline (Analyzer → Generator → Tester)

Analyze codebase → generate migration/code → test result. Stages are sequential but each stage is an independent agent.

**When to use**: Migrating code (e.g., old auth to new, REST to GraphQL), generating boilerplate, or applying complex transformations.

**Pane layout**:
```
├─ analyze
├─ generate (waits for analyze)
└─ test (waits for generate)
```

**Template**:
```typescript
async function codeGenPipeline() {
  // STAGE 1: Analyze current code
  console.log("Stage 1: Analyzing current code...");
  const analyzeId = spawn_pane("analyze", `
    node analyze.js --input src/ --output /tmp/analysis.json
  `);
  const analyzeResult = await poll_pane(analyzeId, 300);
  
  if (analyzeResult.exit_code !== 0) {
    console.error("Analysis failed. Aborting pipeline.");
    return false;
  }

  const analysis = JSON.parse(fs.readFileSync('/tmp/analysis.json'));
  console.log(`Found ${analysis.components.length} components to migrate.`);

  // STAGE 2: Generate new code based on analysis
  console.log("Stage 2: Generating new code...");
  const generateId = spawn_pane("generate", `
    node generate.js --analysis /tmp/analysis.json --output src/generated/
  `);
  const genResult = await poll_pane(generateId, 300);
  
  if (genResult.exit_code !== 0) {
    console.error("Code generation failed. Keeping pane open for debugging.");
    return false;
  }

  console.log("Code generation complete.");

  // STAGE 3: Test generated code
  console.log("Stage 3: Testing generated code...");
  const testId = spawn_pane("test", `
    npm run test -- src/generated/
  `);
  const testResult = await poll_pane(testId, 300);

  if (testResult.exit_code === 0) {
    console.log("✓ All tests passed! Pipeline successful.");
    kill_pane(analyzeId);
    kill_pane(generateId);
    kill_pane(testId);
    return true;
  } else {
    console.log("✗ Tests failed. Keeping panes open for debugging.");
    return false;
  }
}
```

**Key ideas**:
- Each stage waits for the previous one to complete
- Fail fast: if Stage 1 fails, don't spawn Stage 2
- Output from one stage is input to the next (file-based contract)
- On error, keep panes open for manual inspection
- On success, clean up all panes

---

## 4. Task Queue Dispatcher (Work Pool)

Main agent maintains a queue of independent tasks. Spawns N worker agents, assigns tasks as workers complete.

**When to use**: You have 10+ independent tasks (e.g., implement 10 features, process 20 files). Spawn 3-5 worker agents and keep them busy.

**Pane layout**:
```
├─ worker-1 (task 1, then task 4, then task 7)
├─ worker-2 (task 2, then task 5, then task 8)
└─ worker-3 (task 3, then task 6, then task 9)
```

**Template**:
```typescript
async function taskQueueDispatcher(tasks, workerCount = 3) {
  const queue = [...tasks]; // copy
  const workers = [];
  const results = {};

  // Initialize workers
  for (let i = 0; i < workerCount; i++) {
    workers.push({
      id: `worker-${i}`,
      paneId: null,
      busy: false,
      tasksCompleted: 0
    });
  }

  while (queue.length > 0 || workers.some(w => w.busy)) {
    // Assign tasks to idle workers
    for (const worker of workers) {
      if (!worker.busy && queue.length > 0) {
        const task = queue.shift();
        console.log(`Assigning ${task.name} to ${worker.id}`);
        
        worker.paneId = spawn_pane(
          worker.id,
          task.command,
          task.cwd
        );
        worker.busy = true;
        worker.currentTask = task.name;
      }
    }

    // Check for completed workers
    const panes = list_panes();
    for (const worker of workers) {
      if (worker.busy) {
        const pane = panes.find(p => p.id === worker.paneId);
        if (pane && pane.status !== "running") {
          // Worker finished its task
          const result = poll_pane(worker.paneId, 0);
          results[worker.currentTask] = {
            status: result.exit_code === 0 ? 'success' : 'failed',
            duration: result.elapsed_seconds
          };
          console.log(`✓ ${worker.currentTask} complete (${result.elapsed_seconds}s)`);
          
          worker.busy = false;
          worker.tasksCompleted++;
          kill_pane(worker.paneId);
        }
      }
    }

    // Small sleep to avoid busy-waiting
    if (queue.length > 0 || workers.some(w => w.busy)) {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }

  // Summary
  const successful = Object.values(results).filter(r => r.status === 'success').length;
  console.log(`
    ═══════════════════════════════════
    TASK QUEUE COMPLETE
    ═══════════════════════════════════
    Total tasks:   ${tasks.length}
    Successful:    ${successful}
    Failed:        ${tasks.length - successful}
    Workers used:  ${workerCount}
    ═══════════════════════════════════
  `);

  return results;
}

// Example usage:
const tasks = [
  { name: "feature-auth", command: "node implement-feature.js --feature auth" },
  { name: "feature-logging", command: "node implement-feature.js --feature logging" },
  { name: "feature-cache", command: "node implement-feature.js --feature cache" },
  // ... 10+ tasks
];

await taskQueueDispatcher(tasks, 3); // 3 workers
```

**Key ideas**:
- Maintain a task queue and a pool of workers
- As workers finish, assign them new tasks
- Match worker count to typical task duration and queue size
- Close panes after each task completes (keep workspace clean)
- Provide summary of successes/failures

---

## 5. Background Builder + Foreground Work

Start a long-running compilation/test in the background. Main agent continues with other work. Poll periodically.

**When to use**: You're waiting for a 2+ minute compilation, and you have code review, documentation, or design work to do.

**Pane layout**:
```
├─ build (long-running: cargo build --release)
└─ [main agent: refactoring, documentation, analysis]
```

**Template**:
```typescript
async function backgroundBuildWithForegroundWork() {
  // Start long-running build in background
  console.log("Spawning build pane (cargo build --release)...");
  const buildId = spawn_pane("build", "cargo build --release");
  console.log(`Build pane spawned. Build will run in background.`);

  // Main agent continues with other work
  console.log("Proceeding with code review and refactoring...");
  await doCodeReview(); // your work function
  await refactorModule(); // your work function

  // Periodically check build status
  console.log("Checking build progress...");
  for (let i = 0; i < 6; i++) { // check every 30 seconds, up to 3 minutes
    await new Promise(resolve => setTimeout(resolve, 30000));
    
    const panes = list_panes();
    const buildPane = panes.find(p => p.id === buildId);
    
    if (buildPane && buildPane.status === "done") {
      console.log(`Build complete! Exit code: ${buildPane.exit_code}`);
      break;
    } else if (buildPane && buildPane.status === "error") {
      console.error("Build failed!");
      const output = get_pane_output(buildId, 50);
      console.log("Last 50 lines of build output:");
      console.log(output);
      break;
    } else {
      console.log(`Build still running (${buildPane.elapsed_seconds}s)...`);
    }
  }

  // Final poll to get result
  console.log("Waiting for final build result...");
  const buildResult = poll_pane(buildId, 60); // wait up to 1 min more

  if (buildResult.exit_code === 0) {
    console.log("✓ Build succeeded!");
    kill_pane(buildId); // clean up
    return true;
  } else {
    console.log("✗ Build failed. Keeping pane open for inspection.");
    return false;
  }
}
```

**Key ideas**:
- Spawn long-running task immediately, don't wait
- Main agent does productive work while waiting
- Periodically check status with `list_panes()` + `get_pane_output()` (non-blocking)
- Final `poll_pane()` blocks until completion
- Keep panes open on failure, close on success

---

## 6. Parallel Analysis + Decision

Multiple agents analyze the same thing independently, then main agent decides based on consensus.

**When to use**: You want multiple perspectives on a risky change (security, performance, architecture impact) and want to make a data-driven decision.

**Pane layout**:
```
├─ analyze-security
├─ analyze-perf
└─ analyze-compat
```

**Template**:
```typescript
async function parallelAnalysisDecision() {
  const target = "src/core/auth/"; // what to analyze
  
  // Spawn 3 independent analysis agents
  const secAnalysisId = spawn_pane("analyze-security", `
    node security-analysis.js --target ${target} --output /tmp/sec-risk.json
  `);
  
  const perfAnalysisId = spawn_pane("analyze-perf", `
    node perf-analysis.js --target ${target} --output /tmp/perf-risk.json
  `);
  
  const compatAnalysisId = spawn_pane("analyze-compat", `
    node compat-analysis.js --target ${target} --output /tmp/compat-risk.json
  `);

  // Wait for all
  await Promise.all([
    poll_pane(secAnalysisId, 300),
    poll_pane(perfAnalysisId, 300),
    poll_pane(compatAnalysisId, 300)
  ]);

  // Load results
  const secRisk = JSON.parse(fs.readFileSync('/tmp/sec-risk.json'));
  const perfRisk = JSON.parse(fs.readFileSync('/tmp/perf-risk.json'));
  const compatRisk = JSON.parse(fs.readFileSync('/tmp/compat-risk.json'));

  // Aggregate and decide
  const totalRiskScore = secRisk.score + perfRisk.score + compatRisk.score;
  const avgRisk = totalRiskScore / 3;

  const decision = avgRisk < 3 ? "SAFE_TO_PROCEED" : "HIGH_RISK_INVESTIGATE";

  console.log(`
    ═══════════════════════════════════
    ANALYSIS SUMMARY
    ═══════════════════════════════════
    Security risk:       ${secRisk.score}/10 - ${secRisk.summary}
    Performance risk:    ${perfRisk.score}/10 - ${perfRisk.summary}
    Compatibility risk:  ${compatRisk.score}/10 - ${compatRisk.summary}
    ─────────────────────────────────────
    Average risk:        ${avgRisk.toFixed(1)}/10
    Decision:            ${decision}
    ═══════════════════════════════════
  `);

  // Clean up
  kill_pane(secAnalysisId);
  kill_pane(perfAnalysisId);
  kill_pane(compatAnalysisId);

  return decision === "SAFE_TO_PROCEED";
}
```

**Key ideas**:
- Agents analyze independently, no cross-talk
- Each writes its own output file
- Main agent aggregates results and decides
- Clear decision criteria (thresholds, scoring)

---

## Common Mistakes to Avoid

❌ **Spawning agents one-by-one with waits in between**
```typescript
// WRONG
const id1 = spawn_pane("task-1", "cmd1");
await poll_pane(id1); // waits before task-2 even starts!
const id2 = spawn_pane("task-2", "cmd2");
await poll_pane(id2);
```

✅ **Spawn all, then wait all**
```typescript
// RIGHT
const id1 = spawn_pane("task-1", "cmd1");
const id2 = spawn_pane("task-2", "cmd2");
await Promise.all([poll_pane(id1), poll_pane(id2)]);
```

---

❌ **Not checking if agents succeeded**
```typescript
// WRONG: proceed even if analysis failed
const id = spawn_pane("analyze", "node analyze.js > /tmp/out.json");
await poll_pane(id);
const result = JSON.parse(fs.readFileSync('/tmp/out.json')); // crash if file doesn't exist
```

✅ **Check exit code before using results**
```typescript
// RIGHT
const id = spawn_pane("analyze", "node analyze.js > /tmp/out.json");
const result = await poll_pane(id);
if (result.exit_code !== 0) {
  console.error("Analysis failed. Aborting.");
  return;
}
const data = JSON.parse(fs.readFileSync('/tmp/out.json'));
```

---

❌ **Spawning too many agents without matching resources**
```typescript
// WRONG: 50 agents on 8-core system
for (let i = 0; i < 50; i++) {
  spawn_pane(`agent-${i}`, "heavy-computation");
}
```

✅ **Match agent count to resources**
```typescript
// RIGHT: 8 agents on 8-core system
const agentCount = 8; // or use os.cpus().length
for (let i = 0; i < agentCount; i++) {
  spawn_pane(`agent-${i}`, "heavy-computation");
}
```

---

## Further Reading

- See `orchestration-guide.md` for **why** these patterns work
- See `pane-lifecycle.md` for detailed **pane management** strategies
- Return to `SKILL.md` for the **full API reference**
