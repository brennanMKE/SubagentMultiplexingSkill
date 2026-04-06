# CMUX Subagent Skill

## Overview

This document defines a reusable **skill** for spawning subagents in new `cmux` panes. It is designed for agent frameworks like `pi` or Claude-style systems where agents can delegate tasks to subprocesses.

---

## Purpose

Enable an agent to:

- Spawn independent subagents
- Run them in isolated `cmux` panes
- Execute tasks in parallel
- Stream results asynchronously

---

## Skill Definition

```yaml
name: spawn_cmux_subagent
description: Spawn a new subagent in a dedicated cmux pane
inputs:
  name:
    type: string
    description: Human-readable name of the subagent
  agent:
    type: string
    description: Agent profile to use
  task:
    type: string
    description: Task to execute
  model:
    type: string
    required: false
  tools:
    type: array
    required: false
  fork:
    type: boolean
    default: false

execution:
  type: shell
  command: |
    cmux new-pane -n "${name}" "pi run \\
      --agent ${agent} \\
      --task \"${task}\" \\
      ${model:+--model ${model}} \\
      ${tools:+--tools ${tools}} \\
      ${fork:+--fork}"
```

---

## Behavior

### What happens when invoked

1. A new `cmux` pane is created
2. A subagent process is launched inside the pane
3. The subagent executes independently
4. Output is visible in real time
5. Results are returned asynchronously

---

## Example Usage

### Single subagent

```js
spawn_cmux_subagent({
  name: "Auth Analyzer",
  agent: "scout",
  task: "Analyze authentication flow"
});
```

### Parallel subagents

```js
["auth", "db", "api"].forEach(topic => {
  spawn_cmux_subagent({
    name: `Scout: ${topic}`,
    agent: "scout",
    task: `Analyze ${topic} module`
  });
});
```

---

## Optional Enhancements

### 1. Pane Layout Control

```bash
cmux split-pane -h
cmux split-pane -v
```

### 2. Named Sessions

```bash
cmux new-session -s agents
```

### 3. Logging

Pipe output to log files:

```bash
| tee logs/${name}.log
```

---

## Advanced Version (with logging)

```yaml
execution:
  type: shell
  command: |
    cmux new-pane -n "${name}" "pi run \\
      --agent ${agent} \\
      --task \"${task}\" \\
      ${model:+--model ${model}} \\
      ${tools:+--tools ${tools}} \\
      ${fork:+--fork} \\
      2>&1 | tee logs/${name}.log"
```

---

## Notes

- Requires `cmux` installed and running
- Assumes `pi` CLI or equivalent agent runner
- Each pane is an isolated execution environment

---

## Minimal Alternative (no pi)

```yaml
execution:
  type: shell
  command: |
    cmux new-pane -n "${name}" "bash -lc '${task}'"
```

---

## Summary

This skill provides:

- True parallel agent execution
- Visual monitoring via panes
- Clean isolation between tasks

Use it as a foundation for building multi-agent systems with real process-level separation.

