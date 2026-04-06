---
name: subagent-multiplexing
description: How to spawn multiple subagent tasks in parallel using cmux panes. Use this skill whenever you need to: run multiple independent agents concurrently in visible panes, parallelize testing/analysis/code generation, run a pipeline where one agent feeds into the next, or distribute work across worker agents. Teaches orchestration patterns with real cmux commands—fan-out/fan-in, pipelines, background workers, work queues.
---

# subagent-multiplexing

Orchestrate multiple Claude Code subagents in parallel by spawning them in dedicated cmux panes. Each pane is visible and monitorable. All run concurrently while main agent stays responsive.

## What This Skill Does

This skill teaches you how to:
- **Create multiple cmux panes** with `cmux new-pane` for each subagent task
- **Send subagent commands/prompts to panes** using `cmux send`
- **Monitor pane output live** using `cmux read-screen` and `cmux list-panes`
- **Coordinate results** by reading pane output and synthesizing findings
- **Orchestrate patterns** — Fan-out/fan-in, pipelines, background workers, work distribution

**Key insight**: Create panes → send commands → monitor output → coordinate results. All run in parallel.

---

## Core Concept: Parallel vs Sequential

### ❌ Sequential (Main agent blocked)

```bash
# Run first task and wait for completion
cmux send --surface surface:1 "task 1 command"
# ... wait for output ...
cmux read-screen --surface surface:1

# Then run second task (main agent blocked the whole time)
cmux send --surface surface:1 "task 2 command"
# ... wait for output ...
cmux read-screen --surface surface:1

# Three tasks × 5 min each = 15 minutes total
```

### ✅ Parallel (Main agent responsive)

```bash
# Get the Claude pane (where you type commands) — Claude stays here
CLAUDE_SURFACE=$(cmux identify --no-caller | grep -o "surface:[0-9]*")

# Split right from Claude pane to create three new subagent panes stacked on right side
S1=$(cmux new-split right --surface "$CLAUDE_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)
S2=$(cmux new-split down --surface "$S1" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)
S3=$(cmux new-split down --surface "$S2" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Send all three task commands at once (non-blocking)
cmux send --surface "$S1" "task 1 command"
cmux send-key --surface "$S1" Return

cmux send --surface "$S2" "task 2 command"
cmux send-key --surface "$S2" Return

cmux send --surface "$S3" "task 3 command"
cmux send-key --surface "$S3" Return

# All three run simultaneously on the right side
# Main agent continues immediately (~5 minutes wall-clock time)

# Monitor progress anytime
cmux read-screen --surface "$S1"
cmux read-screen --surface "$S2"
cmux read-screen --surface "$S3"

# Collect results when ready
```

---

## When to Use This Skill

**✅ Perfect for:**
- **Parallel analysis** — Multiple agents investigating security, performance, architecture simultaneously
- **Parallel testing** — Unit, integration, E2E tests running in separate panes
- **Code generation pipeline** — Analyzer → Generator → Validator (sequential stages, each depends on previous)
- **Research sprints** — Scouts investigating different areas in parallel
- **Work distribution** — Dispatcher assigns tasks to worker panes as they complete

**❌ Avoid when:**
- **Single focused task** — One agent is simpler and clearer
- **Tight sequential dependencies** — Use Pipeline pattern (sequential) not Fan-Out
- **Very quick tasks** — If each task <1 second, parallelization overhead exceeds benefit
- **Constant inter-pane communication** — Sequential is simpler

---

## Orchestration Patterns

### Pattern 1: Fan-Out / Fan-In (Most Common)

Create N panes. Send independent tasks to all panes simultaneously. Collect all results in parallel.

**Use for**: Parallel testing, analysis, code review, research sprints.

**How it works**:
1. Create N panes with `cmux new-pane`
2. Send all N task commands immediately (all at once)
3. Monitor progress with `cmux read-screen` and `cmux list-panes`
4. As each pane completes, read its output
5. Synthesize all results together

**Example**:

```bash
# PHASE 1: Create three subagent panes on the right side
# This keeps Claude on the left, subagents stacked on the right

# Get Claude pane
CLAUDE_SURFACE=$(cmux identify --no-caller | grep -o "surface:[0-9]*")

# Split right to create first subagent pane
SURFACE1=$(cmux new-split right --surface "$CLAUDE_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Split down twice to create two more subagents stacked below the first
SURFACE2=$(cmux new-split down --surface "$SURFACE1" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)
SURFACE3=$(cmux new-split down --surface "$SURFACE2" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# PHASE 2: Send all three task commands at once (parallel)
cmux send --surface $SURFACE1 "npm test:unit && echo 'UNIT_TESTS_DONE'"
cmux send-key --surface $SURFACE1 Return

cmux send --surface $SURFACE2 "npm test:integration && echo 'INTEGRATION_TESTS_DONE'"
cmux send-key --surface $SURFACE2 Return

cmux send --surface $SURFACE3 "npm test:e2e && echo 'E2E_TESTS_DONE'"
cmux send-key --surface $SURFACE3 Return

# All three tests run in parallel immediately

# PHASE 3: Monitor progress
echo "Monitoring pane 1:"
while ! cmux read-screen --surface $SURFACE1 | grep -q "UNIT_TESTS_DONE"; do
  echo "  Still running..."
  sleep 2
done

echo "Monitoring pane 2:"
while ! cmux read-screen --surface $SURFACE2 | grep -q "INTEGRATION_TESTS_DONE"; do
  sleep 2
done

echo "Monitoring pane 3:"
while ! cmux read-screen --surface $SURFACE3 | grep -q "E2E_TESTS_DONE"; do
  sleep 2
done

# PHASE 4: Collect results
RESULT1=$(cmux read-screen --surface $SURFACE1)
RESULT2=$(cmux read-screen --surface $SURFACE2)
RESULT3=$(cmux read-screen --surface $SURFACE3)

# PHASE 5: Synthesize
if echo "$RESULT1" | grep -q "PASSED" && \
   echo "$RESULT2" | grep -q "PASSED" && \
   echo "$RESULT3" | grep -q "PASSED"; then
  echo "✅ All tests passed"
else
  echo "❌ Some tests failed"
fi

# PHASE 6: Cleanup (closes the right-side subagent panes)
cmux close-surface --surface $SURFACE1
cmux close-surface --surface $SURFACE2
cmux close-surface --surface $SURFACE3
```

**Why it's powerful**:
- 3 tasks in parallel = ~5 min (instead of 15 min serial)
- If one pane fails, others still complete — see full picture
- Real-time visibility in visible panes
- Main agent stays responsive throughout

---

### Pattern 2: Pipeline (Sequential Stages with Dependencies)

Create panes sequentially. Run Stage 1, wait for completion, read output, spawn Stage 2 with Stage 1's output, etc.

**Use for**: Analysis → Code Generation → Testing (each stage depends on previous output)

**How it works**:
1. Create pane 1, send task, wait for completion
2. Read pane 1 output
3. Create pane 2, send task using pane 1's output
4. Wait for pane 2 completion
5. Continue to next stage

**Example**:

```bash
# Get Claude pane
CLAUDE_SURFACE=$(cmux identify --no-caller | grep -o "surface:[0-9]*")

# STAGE 1: Create analyzer pane on the right side
ANALYZER_SURFACE=$(cmux new-split right --surface "$CLAUDE_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Send analysis command
cmux send --surface $ANALYZER_SURFACE "echo 'Analyzing...' && find . -name '*.js' | wc -l > /tmp/analysis.txt && echo 'ANALYSIS_DONE'"
cmux send-key --surface $ANALYZER_SURFACE Return

# Wait for analyzer to complete
echo "Waiting for analysis..."
while ! cmux read-screen --surface $ANALYZER_SURFACE | grep -q "ANALYSIS_DONE"; do
  sleep 1
done

# Read analyzer output
ANALYSIS=$(cat /tmp/analysis.txt)
echo "Analysis result: $ANALYSIS files found"

# STAGE 2: Create generator pane (split down from analyzer)
GENERATOR_SURFACE=$(cmux new-split down --surface "$ANALYZER_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Send generation command with analyzer output
cmux send --surface $GENERATOR_SURFACE "echo 'Generating code for $ANALYSIS files...' && sleep 2 && echo 'GENERATION_DONE'"
cmux send-key --surface $GENERATOR_SURFACE Return

# Wait for generator to complete
echo "Waiting for code generation..."
while ! cmux read-screen --surface $GENERATOR_SURFACE | grep -q "GENERATION_DONE"; do
  sleep 1
done

echo "Generation complete"

# STAGE 3: Create tester pane (split down from generator)
TESTER_SURFACE=$(cmux new-split down --surface "$GENERATOR_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

cmux send --surface $TESTER_SURFACE "echo 'Running tests on generated code...' && sleep 2 && echo 'TESTS_DONE'"
cmux send-key --surface $TESTER_SURFACE Return

while ! cmux read-screen --surface $TESTER_SURFACE | grep -q "TESTS_DONE"; do
  sleep 1
done

echo "Pipeline complete"
```

**Why it's powerful**:
- Each stage has exact output from previous stage
- No wasted assumptions or rework
- Clear dependency chain

---

### Pattern 3: Background Worker

Create a pane, send a long-running task, continue with other work immediately.

**Use for**: Builds, long tests, deployments (while main agent does other work)

**How it works**:
1. Create pane
2. Send long-running task command
3. Return immediately (don't wait)
4. Continue with other work in main session
5. Check pane progress anytime
6. Collect results when ready

**Example**:

```bash
# Get Claude pane
CLAUDE_SURFACE=$(cmux identify --no-caller | grep -o "surface:[0-9]*")

# Create build pane on the right side
BUILD_SURFACE=$(cmux new-split right --surface "$CLAUDE_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Send long-running build command
cmux send --surface $BUILD_SURFACE "cargo build --release 2>&1 | tee /tmp/build.log && echo 'BUILD_COMPLETE'"
cmux send-key --surface $BUILD_SURFACE Return

# Return immediately — main agent continues with other work
echo "Build started in background (pane: $BUILD)"
echo "Main agent continues with other tasks..."

# Do other work while build runs
sleep 2
echo "Checking build progress..."
BUILD_OUTPUT=$(cmux read-screen --surface $BUILD_SURFACE)
echo "$BUILD_OUTPUT" | tail -5

# When ready, wait for build to complete
echo "Waiting for build to finish..."
while ! cmux read-screen --surface $BUILD_SURFACE | grep -q "BUILD_COMPLETE"; do
  sleep 5
  echo "  Still building..."
done

echo "Build finished! Collecting results..."
FINAL_BUILD=$(cmux read-screen --surface $BUILD_SURFACE)
echo "$FINAL_BUILD"
```

**Why it's powerful**:
- Zero idle time for main agent
- Long task doesn't block other work
- Can monitor progress anytime

---

### Pattern 4: Work Queue / Task Distribution

Create N worker panes. Send first batch of tasks. As each worker completes, assign next task.

**Use for**: Batch processing, implementing multiple features, scaling to variable workload.

**How it works**:
1. Create N worker panes
2. Send first N tasks (one per worker)
3. Monitor all panes
4. When a worker finishes, send it the next task
5. Continue until all tasks done

**Example**:

```bash
# Task list
TASKS=("build_auth" "build_api" "build_db" "add_tests" "add_docs" "add_deploy")

# Get Claude pane and create 2 worker panes on the right side
CLAUDE_SURFACE=$(cmux identify --no-caller | grep -o "surface:[0-9]*")

# Split right to create first worker pane
WORKER1_SURFACE=$(cmux new-split right --surface "$CLAUDE_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Split down to create second worker pane below first
WORKER2_SURFACE=$(cmux new-split down --surface "$WORKER1_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Assign first tasks
cmux send --surface $WORKER1_SURFACE "echo 'Task: ${TASKS[0]}' && sleep 3 && echo 'DONE'"
cmux send-key --surface $WORKER1_SURFACE Return

cmux send --surface $WORKER2_SURFACE "echo 'Task: ${TASKS[1]}' && sleep 3 && echo 'DONE'"
cmux send-key --surface $WORKER2_SURFACE Return

# Dispatcher loop
TASK_IDX=2
while [ $TASK_IDX -lt ${#TASKS[@]} ]; do
  # Check if worker 1 is done
  if cmux read-screen --surface $WORKER1_SURFACE | grep -q "DONE"; then
    echo "Worker 1 done, assigning task: ${TASKS[$TASK_IDX]}"
    cmux send --surface $WORKER1_SURFACE "echo 'Task: ${TASKS[$TASK_IDX]}' && sleep 3 && echo 'DONE'"
    cmux send-key --surface $WORKER1_SURFACE Return
    ((TASK_IDX++))
  fi
  
  # Check if worker 2 is done
  if [ $TASK_IDX -lt ${#TASKS[@]} ] && cmux read-screen --surface $WORKER2_SURFACE | grep -q "DONE"; then
    echo "Worker 2 done, assigning task: ${TASKS[$TASK_IDX]}"
    cmux send --surface $WORKER2_SURFACE "echo 'Task: ${TASKS[$TASK_IDX]}' && sleep 3 && echo 'DONE'"
    cmux send-key --surface $WORKER2_SURFACE Return
    ((TASK_IDX++))
  fi
  
  sleep 1
done

echo "All tasks distributed"
```

**Why it's powerful**:
- N workers handle M tasks (M > N)
- ~M/N wall-clock time (instead of M serial)
- Scales with workload
- Workers stay busy

---

## Key Principles

**1. Create panes upfront, send commands immediately**
```bash
# ✅ Correct: Create all panes, then send all commands
PANE1=$(cmux new-pane --workspace workspace:1)
PANE2=$(cmux new-pane --workspace workspace:1)
PANE3=$(cmux new-pane --workspace workspace:1)

cmux send --surface $SURFACE1 "task 1"
cmux send --surface $SURFACE2 "task 2"
cmux send --surface $SURFACE3 "task 3"

# ❌ Wrong: Creating and sending one at a time (sequential)
PANE1=$(cmux new-pane --workspace workspace:1)
cmux send --surface $SURFACE1 "task 1"
# ... wait ...
PANE2=$(cmux new-pane --workspace workspace:1)
cmux send --surface $SURFACE2 "task 2"
```

**2. Monitor output with `cmux read-screen`**
```bash
# Get current state of a pane
cmux read-screen --surface surface:1 --lines 30

# Check for completion markers
if cmux read-screen --surface surface:1 | grep -q "DONE"; then
  echo "Task complete"
fi
```

**3. Each pane is independent**
- Separate process, separate output
- One pane failing doesn't affect others
- Collect results from all panes even if some fail

**4. Visibility is built-in**
- Panes are visible in the workspace
- You can jump to any pane to see real-time output
- `cmux list-panes` shows all active panes
- `cmux read-screen` captures current output

**5. Coordinate via files and output parsing**
```bash
# Pane writes status to file
cmux send --surface $SURFACE1 "command > /tmp/result.txt && echo 'DONE'"

# Main agent reads file and checks status
while ! grep -q "DONE" /tmp/result.txt; do sleep 1; done
RESULT=$(cat /tmp/result.txt)
```

---

## Common Use Cases

### Parallel Code Review

```bash
# Three reviewers, three panes, reviewing in parallel
SECURITY=$(cmux new-pane --workspace workspace:1)
PERF=$(cmux new-pane --workspace workspace:1)
ARCH=$(cmux new-pane --workspace workspace:1)

cmux send --surface surface:1 "Review code for security vulnerabilities"
cmux send --surface surface:2 "Profile code for performance bottlenecks"
cmux send --surface surface:3 "Review architecture and design patterns"

# All three review simultaneously
```

### Parallel Test Execution

```bash
# Three test suites, three panes
UNIT=$(cmux new-pane --workspace workspace:1)
INTEG=$(cmux new-pane --workspace workspace:1)
E2E=$(cmux new-pane --workspace workspace:1)

cmux send --surface $UNIT_SURFACE "npm test:unit"
cmux send --surface $INTEG_SURFACE "npm test:integration"
cmux send --surface $E2E_SURFACE "npm test:e2e"

# All three run in parallel (~5 min instead of 15)
```

---

## Tips for Success

**1. Give panes descriptive names by using descriptive commands**
```bash
# ✅ Clear what each pane does
cmux send --surface $SURFACE1 "# Unit Tests\nnpm test:unit"
cmux send --surface $SURFACE2 "# Integration Tests\nnpm test:integration"

# This shows in pane output and workspace
```

**2. Use completion markers**
```bash
# Add `&& echo 'DONE'` to commands so you can detect completion
cmux send --surface $SURFACE1 "npm test && echo 'UNIT_TESTS_DONE'"

# Poll for the marker
while ! cmux read-screen --surface $SURFACE1 | grep -q "UNIT_TESTS_DONE"; do
  sleep 1
done
```

**3. Store results in `/tmp/` for later processing**
```bash
cmux send --surface $SURFACE1 "npm test > /tmp/unit-tests.log && echo 'DONE'"
# ... wait for completion ...
cat /tmp/unit-tests.log | analyze.sh
```

**4. One pane failing doesn't stop others**
```bash
# If pane 1 fails, panes 2 and 3 still complete
# Collect results from all panes
RESULT1=$(cmux read-screen --surface $SURFACE1)
RESULT2=$(cmux read-screen --surface $SURFACE2)
RESULT3=$(cmux read-screen --surface $SURFACE3)

# Synthesize even with partial failures
```

**5. Close panes when done**
```bash
cmux close-surface --surface $SURFACE1
cmux close-surface --surface $SURFACE2
cmux close-surface --surface $SURFACE3
```

---

## Reference: Key cmux Commands

```bash
# Create subagent panes on the right side (Claude stays on left)
CLAUDE_SURFACE=$(cmux identify --no-caller | grep -o "surface:[0-9]*")

# Split right to create first pane on right side
PANE1=$(cmux new-split right --surface "$CLAUDE_SURFACE" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Split down to create more panes stacked below
PANE2=$(cmux new-split down --surface "$PANE1" --workspace workspace:1 | grep -o "surface:[0-9]*" | head -1)

# Send commands to panes
cmux send --surface $PANE1 "your command here"
cmux send-key --surface $PANE1 Return

# Monitor panes
cmux list-panes --workspace workspace:1
cmux read-screen --surface $PANE1 --lines 50

# Clean up
cmux close-surface --surface $PANE1
```

---

## See Also

- `references/orchestration-guide.md` — Deep architectural patterns
- `references/subagent-patterns.md` — Ready-to-use workflow templates
- `references/pane-lifecycle.md` — Managing pane visibility and cleanup
- `references/sandbox-constraints.md` — Handling isolated agent I/O
