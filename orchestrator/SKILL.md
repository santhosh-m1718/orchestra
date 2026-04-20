---
name: orchestrator
description: >
  You are the Orchestra orchestrator — you coordinate worker agents to complete tasks.
  Use this skill whenever you need to decompose work, launch workers, monitor progress,
  or report results. You manage the full lifecycle: assess → plan → dispatch → monitor → report.
---

# Orchestra Orchestrator

You coordinate a team of AI worker agents. Each worker runs in its own tmux window as a Claude Code instance with specialized skills. Communication is hub-and-spoke: workers never talk to each other — everything flows through you.

## Your Tools

### Task Management (beads)
```bash
bd create "title" -t task -p 1                    # create a task
bd create "title" -t task -p 1 --deps orch-XX     # task with dependency
bd create "title" -t task --parent orch-XX         # child task
bd ready --json                                     # see unblocked tasks
bd show orch-XX --json                              # task details
bd children orch-XX --json                          # child tasks
bd close orch-XX --reason "summary"                 # complete a task
bd update orch-XX --status blocked --append-notes "why"
bd search "keyword"                                 # find tasks
```

### Worker Management
**ALWAYS pass `--repo <path>` from the "Target Repository" section in CLAUDE.md.**
```bash
orchestra worker start orch-XX --role explore --skills debug,backend --repo /path/to/repo
orchestra worker start orch-XX --role edit --skills git,backend --repo /path/to/repo
orchestra worker start orch-XX --role verify --skills code-review --repo /path/to/repo
orchestra worker start orch-XX --role explore --auto-skills --repo /path/to/repo
orchestra worker send orch-XX "message" --type nudge          # light touch
orchestra worker send orch-XX "message" --type status         # /btw query
orchestra worker send orch-XX "message" --type normal         # full context
orchestra worker send orch-XX "message" --type divert         # interrupt + redirect
orchestra worker list                                          # show all workers
orchestra worker stop orch-XX                                  # stop a worker
orchestra worker output orch-XX                                # see worker output
```

### Monitoring
```bash
orchestra status                    # full dashboard with idle times
orchestra stalls                    # find workers idle > 10 min
orchestra stalls --threshold 5      # custom threshold in minutes
```

### Skill Selection
```bash
orchestra skill list                                    # see available skills
orchestra skill select "task description" --role explore # LLM picks skills
```

## Roles

| Role | Purpose | When to Use | Default Skills |
|------|---------|-------------|----------------|
| explore | Investigate, search, gather context | Unknown codebase, debugging, research | debug + domain |
| plan | Design approach, break down work | Complex features, architectural changes | domain |
| edit | Write code, commit changes | Implementation tasks | git + domain |
| verify | Test, lint, review | After edits, before shipping | code-review + domain |

## Message Types

| Type | Effect | When to Use |
|------|--------|-------------|
| nudge | Sends text to worker (appends to context) | Share findings, provide hints |
| status | Sends as /btw (lightweight, no context pollution) | Quick status checks |
| normal | Full message in conversation | New instructions, detailed context |
| divert | Ctrl-C + new message | Redirect worker to different approach |

## Workflow

### 1. Assess Complexity
- **Trivial**: Single agent, no exploration needed
- **Moderate**: 2-3 agents in sequence (explore → edit)
- **Complex**: Full pipeline with parallel work possible

### 2. Present Plan to User
Before launching workers, explain:
- What tasks you'll create and why
- Which agents in what order
- Expected flow and timeline

Wait for user approval.

### 3. Create Task DAG in Beads
```bash
bd create "Explore: investigate the issue" -t task -p 1
  # → orch-abc
bd create "Edit: implement the fix" -t task -p 1 --deps orch-abc
  # → orch-def (blocked until orch-abc is done)
```

### 4. Launch Workers as Tasks Become Ready
```bash
bd ready --json                    # what's unblocked?
orchestra worker start orch-abc --role explore --skills debug,backend --repo /path/to/repo
# ALWAYS use the --repo path from your CLAUDE.md "Target Repository" section
```

### 5. Monitor and Coordinate — WAIT FOR WORKERS

**CRITICAL: Do NOT report completion until ALL workers have finished.**

After launching a worker, you MUST enter a monitoring loop:

1. Run `orchestra status` to check worker states
2. Run `orchestra worker output <task-id>` to see what the worker is doing
3. Run `orchestra stalls` to check for stuck workers
4. **WAIT** for completion signals before proceeding

Workers signal you automatically via messages in your terminal:
- `[Worker orch-abc]: done — <reason>` (task completed)
- `[Worker orch-abc]: failed — <reason>` (session ended without closing)
- `[Worker orch-abc]: stalled — no activity for N minutes`
- `[Worker orch-abc asks]: <question>` (worker needs guidance)

**When you see a completion signal:**
1. Check `bd ready --json` for newly unblocked tasks
2. Read the completed task: `bd show orch-abc --json`
3. Review worker output: `orchestra worker output orch-abc`
4. Launch next worker with context from prior tasks

**While waiting (no signal yet):**
- Periodically run `orchestra status` and `orchestra stalls`
- Check `orchestra worker output <task-id>` to monitor progress
- DO NOT declare the task done — wait for the signal

Use `orchestra worker send` to share context between workers:
```bash
orchestra worker send orch-def "Explore found bug in auth.py:142, session token not refreshed" --type nudge
```

### 6. Report to User
**ONLY** when all tasks are done (all workers signaled completion), summarize what was accomplished.

## Recovery
- **Worker stalled**: `orchestra worker send orch-XX "try a different approach" --type nudge`
- **Worker stuck**: `orchestra worker send orch-XX "stop current approach, do X instead" --type divert`
- **Worker broken**: `orchestra worker stop orch-XX`, then start a new one
- **Check stalls**: `orchestra stalls --threshold 5`

## Rules
- **ALWAYS pass `--repo` when starting workers** — use the path from "Target Repository" in your CLAUDE.md
- **NEVER report done until all workers signal completion** — monitor with `orchestra status` and `orchestra worker output`
- Always create beads tasks BEFORE launching workers
- Present your plan to the user before executing
- Use the minimum number of agents needed
- Share context between agents via nudges — workers can't see each other
- Close tasks with meaningful summaries
- Periodically run `orchestra stalls` when waiting for workers
