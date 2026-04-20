# Orchestra

AI Agent Orchestrator — beads task DAG + tmux sessions + Claude Code workers.

```
                    ┌──────────────┐
          ┌────────│ ORCHESTRATOR │────────┐
          │        │  (Claude AI) │        │
          │        └──────┬───────┘        │
          │               │                │
     send-keys       send-keys        send-keys
     signals         signals           signals
          │               │                │
          ▼               ▼                ▼
     ┌────────┐     ┌────────┐       ┌────────┐
     │EXPLORE │     │  PLAN  │       │  EDIT  │
     │worker  │     │ worker │       │ worker │
     └────────┘     └────────┘       └────────┘
```

The orchestrator is a Claude Code agent that coordinates worker agents. Each worker runs in its own tmux window with specialized skills. Communication is hub-and-spoke — workers never talk to each other, everything flows through the orchestrator.

## How It Works

1. **You talk to the orchestrator** — it's a Claude agent running in tmux
2. **Orchestrator creates a task DAG** in beads (dependency-aware issue tracker)
3. **Workers are launched** as Claude Code instances in tmux windows
4. **Workers signal completion** via hooks — orchestrator gets notified automatically
5. **Orchestrator chains results** — explore findings feed into plan, plan feeds into edit

## Install

```bash
# Clone
git clone https://github.com/santhosh-m1718/orchestra.git
cd orchestra

# Install globally (requires uv)
uv tool install --from . orchestra

# Dependencies
brew install tmux       # session management
# beads (bd) — https://github.com/rrigby/beads
# Claude Code — https://claude.ai/code
```

## Quick Start

```bash
# Start the orchestrator (drops you into tmux with Claude)
orchestra start

# You're now talking to Mayushii (the orchestrator agent)
# Ask it to do something:
#   "fix the auth bug in the backend repo"
#   "explore the codebase and explain the architecture"
#   "add input validation to the API"
```

The orchestrator will:
- Create tasks in beads with dependencies
- Launch worker agents with the right roles and skills
- Monitor progress and chain results
- Report back to you when done

## Architecture

```
orchestra/                  # Python package (CLI toolbox)
├── store.py                # SQLite — orchestrators, sessions, messages
├── tmux.py                 # Thin tmux wrapper (send-keys, capture, wait)
├── lifecycle.py            # Start/stop/send workers, workspace setup
├── skills.py               # Skill catalog + LLM selection + symlink injection
├── hooks.py                # Claude Code hooks + implementations
└── cli.py                  # Typer CLI

roles/                      # Prompt templates for agent roles
├── explore.md              # Investigation, research, code search
├── plan.md                 # Architecture, breakdown, design
├── edit.md                 # Write code, commit, test
└── verify.md               # Test, lint, review

orchestrator/
└── SKILL.md                # The brain — teaches orchestrator to use the CLI
```

### Three Storage Layers

| Layer | What | Why |
|-------|------|-----|
| **Beads** | Task DAG (dependencies, status, descriptions) | Dependency-aware, agent-optimized |
| **SQLite** | Crew state (sessions, messages, heartbeats) | Fast, concurrent, proven by JEFF |
| **Workspaces** | Per-worker directories with CLAUDE.md + hooks | Claude Code auto-loads context |

## CLI Reference

### Orchestrator
```bash
orchestra start                          # start + auto-attach to tmux
orchestra start --model claude-sonnet-4-6  # use a different model
orchestra start --no-attach              # headless mode
orchestra stop                           # stop everything
orchestra status                         # dashboard with idle times
orchestra stalls                         # find stuck workers
```

### Workers
```bash
orchestra worker start orch-XX --role explore --skills debug,backend
orchestra worker start orch-XX --role edit --repo /path/to/repo
orchestra worker start orch-XX --role explore --auto-skills  # LLM picks skills
orchestra worker send orch-XX "hint" --type nudge            # light touch
orchestra worker send orch-XX "new direction" --type divert  # interrupt + redirect
orchestra worker list                                         # show all workers
orchestra worker output orch-XX                               # capture pane output
orchestra worker stop orch-XX
```

### Skills
```bash
orchestra skill list                     # show available skills
orchestra skill select "fix auth bug" --role explore  # LLM selection
```

### Inter-Agent Communication
```bash
orchestra crew ask orch-XX "should I use JWT or sessions?"  # worker → orchestrator
```

## Message Types

| Type | Effect | Use Case |
|------|--------|----------|
| **nudge** | Sends text to worker terminal | Share findings, hints |
| **status** | Sends as `/btw` (no context pollution) | Quick status checks |
| **normal** | Full conversation message | New instructions |
| **divert** | Ctrl-C + new message | Redirect approach |

## Hooks

Workers have three Claude Code hooks that call back into `orchestra`:

| Hook | When | What It Does |
|------|------|-------------|
| **SessionStart** | Worker boots | Injects task details + pending messages |
| **PostToolUse** | Every tool call | Heartbeat — touches `last_seen` in SQLite |
| **Stop** | Worker exits | Checks beads status, signals orchestrator |

## Skills

Skills are loaded from an external repo (default: `~/.orchestra/skills/`). Each skill is a directory with a `SKILL.md` following the [Agent Skills Standard](https://agentskills.io).

```bash
# Clone a skills repo
git clone https://github.com/cbx1/skills ~/.orchestra/skills

# Or set a custom path
export ORCHESTRA_SKILLS_REPO=/path/to/your/skills
```

Skills are **symlinked** into worker workspaces so Claude Code auto-loads them. The orchestrator can use LLM-powered selection to pick the right skills per task.

## Design Principles

- **Orchestrator is an LLM, not a Python loop** — the Claude agent decides decomposition, sequencing, and recovery
- **Beads for coordination, files for payloads** — task DAG in beads, rich output in workspace files
- **Hub-and-spoke communication** — workers never talk to each other
- **Skills as agent DNA** — same role + different skills = different specialist
- **Event-driven signals** — hooks, not polling

## Inspired By

- [JEFF](https://github.com/NeerajG03/jeff) — Go-based agent workspace manager (tmux, personas, crews)
- [Agent Skills Standard](https://agentskills.io) — portable skill format
- [Beads](https://github.com/rrigby/beads) — dependency-aware issue tracker
