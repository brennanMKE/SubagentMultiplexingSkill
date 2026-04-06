# Working with Sandbox Constraints

When orchestrating subagents in parallel, each agent runs in an isolated sandbox context. This provides security benefits but creates a practical constraint: **subagents cannot write to user directories** that the main orchestrator has access to.

## The Problem

In Claude Code:
- The main agent (orchestrator) has write permissions to `/Users/brennan/Developer/...` and other user directories
- Spawned subagents run in restricted sandbox contexts with limited I/O permissions
- Subagents can **read** files but cannot **write** to user directories
- This blocks patterns where agents generate code/output that should land in the project

```
❌ What fails:
  Main Agent → spawns Subagent A
  Subagent A tries: fs.writeFile('/Users/brennan/project/output.js', ...)
  → Permission denied (sandbox constraint)
```

## The Solution: Temp-Write → Orchestrator-Copy Pattern

The reliable workaround is to have agents write to their local temp context, then have the main orchestrator copy results to the final location.

```
✅ What works:
  Main Agent → spawns Subagent A (cwd: /tmp/work-a)
  Subagent A writes: output.js (in its local context)
  Subagent A reports: "output.js written to temp location"
  Main Agent reads the output
  Main Agent writes to: /Users/brennan/project/output.js (using its permissions)
```

## Implementation Pattern

### Setup Phase: Prepare Temp Directories

The main orchestrator creates unique temp directories for each subagent:

```javascript
const os = require("os");
const path = require("path");

// Create isolated temp directories for each subagent
const tempDirs = {
  analyzer: path.join(os.tmpdir(), `subagent-analyzer-${Date.now()}`),
  generator: path.join(os.tmpdir(), `subagent-generator-${Date.now()}`),
  tester: path.join(os.tmpdir(), `subagent-tester-${Date.now()}`),
};

// Create the directories
Object.values(tempDirs).forEach(dir => {
  spawn_pane("setup", `mkdir -p ${dir}`);
});
```

### Spawn Phase: Point Agents to Temp

When spawning subagents, set their `cwd` to their temp directory:

```javascript
// Each agent writes to its own temp location
const analyzerPane = spawn_pane(
  "analyzer",
  "node analyze.js > analysis.json",
  tempDirs.analyzer  // ← agents write here
);

const generatorPane = spawn_pane(
  "generator",
  "node generate.js > generated.js",
  tempDirs.generator
);
```

### Harvest Phase: Main Agent Retrieves & Copies

After agents complete, the main orchestrator reads from temp and writes to final location:

```javascript
const fs = require("fs");

// Poll all agents
const analysisResult = poll_pane(analyzerPane);
const generationResult = poll_pane(generatorPane);

// Main agent reads from subagent temp locations
const analysisContent = fs.readFileSync(
  path.join(tempDirs.analyzer, "analysis.json"),
  "utf-8"
);
const generatedCode = fs.readFileSync(
  path.join(tempDirs.generator, "generated.js"),
  "utf-8"
);

// Main agent writes to the actual project
fs.writeFileSync("/Users/brennan/project/src/analysis.json", analysisContent);
fs.writeFileSync("/Users/brennan/project/src/generated.js", generatedCode);

console.log("✓ Results harvested from temp and written to project");
```

### Cleanup Phase: Remove Temp Directories

```javascript
const { execSync } = require("child_process");

Object.values(tempDirs).forEach(dir => {
  execSync(`rm -rf ${dir}`);
});
```

## Complete Example: Parallel Code Generation

Here's a full orchestration workflow:

```javascript
const os = require("os");
const path = require("path");
const fs = require("fs");

async function orchestrateCodeGeneration() {
  const projectRoot = "/Users/brennan/Developer/my-project";
  
  // 1. Setup: Create temp directories
  const tempDirs = {
    analyzer: path.join(os.tmpdir(), `analyze-${Date.now()}`),
    generator: path.join(os.tmpdir(), `generate-${Date.now()}`),
    tester: path.join(os.tmpdir(), `test-${Date.now()}`),
  };
  
  Object.values(tempDirs).forEach(dir => {
    execSync(`mkdir -p ${dir}`);
  });
  
  console.log("📁 Temp directories created");
  
  // 2. Spawn: Multiple agents in parallel
  const analyzerPane = spawn_pane(
    "analyzer",
    `cd ${projectRoot} && node scripts/analyze.js > ${tempDirs.analyzer}/analysis.json`,
    tempDirs.analyzer
  );
  
  const generatorPane = spawn_pane(
    "generator",
    `node scripts/generate.js --config ${tempDirs.analyzer}/analysis.json > ${tempDirs.generator}/output.js`,
    tempDirs.generator
  );
  
  console.log("🚀 Subagents spawned");
  
  // 3. Wait: Poll all panes
  const analysis = poll_pane(analyzerPane, 60);
  const generated = poll_pane(generatorPane, 120);
  
  if (analysis.status !== "done" || generated.status !== "done") {
    console.error("❌ Subagents failed");
    return;
  }
  
  console.log("✅ Subagents completed");
  
  // 4. Harvest: Main agent reads from temp
  const analysisJson = fs.readFileSync(
    path.join(tempDirs.analyzer, "analysis.json"),
    "utf-8"
  );
  
  const generatedCode = fs.readFileSync(
    path.join(tempDirs.generator, "output.js"),
    "utf-8"
  );
  
  // 5. Write: Main agent writes to project (it has permissions)
  fs.writeFileSync(
    path.join(projectRoot, "src/generated.js"),
    generatedCode
  );
  
  fs.writeFileSync(
    path.join(projectRoot, ".analysis.json"),
    analysisJson
  );
  
  console.log("💾 Results written to project");
  
  // 6. Cleanup: Remove temp
  Object.values(tempDirs).forEach(dir => {
    execSync(`rm -rf ${dir}`);
  });
  
  console.log("🧹 Temp directories cleaned");
}
```

## Best Practices

### 1. Use Descriptive Temp Paths
Include timestamps and agent names so you can debug if cleanup fails:
```javascript
const tempDir = path.join(
  os.tmpdir(),
  `subagent-${agentName}-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`
);
```

### 2. Store Output Paths in Agent Reports
Have subagents report where they wrote output:
```javascript
// In subagent script
const output = { result: "...", location: outputPath };
console.log(JSON.stringify(output));
```

Then orchestrator can parse and retrieve:
```javascript
const output = JSON.parse(agentOutput);
const result = fs.readFileSync(output.location, "utf-8");
```

### 3. Avoid Hardcoded Paths in Subagent Scripts
Pass temp directories as arguments:
```javascript
spawn_pane(
  "generator",
  `node generate.js --output ${tempDirs.generator}/result.js`,
  tempDirs.generator
);
```

### 4. Handle Cleanup Gracefully
Use try-finally to ensure temp cleanup:
```javascript
try {
  // orchestration work
} finally {
  Object.values(tempDirs).forEach(dir => {
    try {
      execSync(`rm -rf ${dir}`);
    } catch (e) {
      console.warn(`Failed to clean ${dir}:`, e.message);
    }
  });
}
```

### 5. Log the Harvest Phase
Make the copy-to-project step explicit and visible:
```javascript
console.log(`\n📋 Harvesting results from subagents...\n`);

// For each output, show: temp location → project location
Object.entries(outputs).forEach(([name, { temp, final }]) => {
  console.log(`  ${name}: ${temp} → ${final}`);
  fs.copyFileSync(temp, final);
});

console.log(`\n✅ All results written to project\n`);
```

## When This Pattern Works Best

| Scenario | Fit | Notes |
|----------|-----|-------|
| Code generation (analyzer → generator → tester) | ✅ Excellent | Agents naturally produce discrete outputs |
| Parallel testing | ✅ Excellent | Test results stay in temp, main agent aggregates |
| Research/analysis | ✅ Excellent | Each agent writes report to temp, main synthesizes |
| Long-running builds | ⚠️ Moderate | Output is often binary; may need streaming |
| Interactive tasks | ❌ Poor | Temp isolation doesn't work for live servers/REPLs |

## Limitations & Workarounds

| Issue | Workaround |
|-------|-----------|
| Agents can't update existing project files mid-task | Have main agent handle updates after all agents complete |
| Large outputs exceed temp space | Stream to main agent via stdout/stderr instead of files |
| Agents need access to project files | Copy needed files to temp at spawn time |
| Cleanup fails (temp space fills) | Monitor `/tmp` size; consider weekly cleanup jobs |

## Related Patterns

- **Fan-Out/Fan-In**: Spawn N agents writing to N temp dirs, harvest results in one batch
- **Pipeline**: Agent A writes to temp → main reads → passes to Agent B via argument
- **Task Queue**: Main dispatcher uses temp dirs as scratch space for work assignments

See `orchestration-guide.md` for more patterns.
