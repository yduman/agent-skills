---
name: builder-validator
description: Orchestrate a real-time adversarial builder-validator workflow using Claude Code Agent Teams. Use when the user requests adversarial review, builder-validator, red-green pairing, or wants implementation and review to happen in parallel with live feedback. Requires CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1. Do NOT use for simple tasks, single-file changes, or when the user just wants a quick code review after the fact.
---

# Builder-Validator Agent Teams Skill

Spawn a two-agent team where a **Builder** implements and a **Validator** challenges in real time. They share a task list and communicate via inbox messaging. The Validator can interrupt the Builder before bad decisions get committed.

## Prerequisites

- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` must be set in env or settings.json
- tmux recommended for split-pane visibility: `--teammate-mode tmux`

## Team Structure

Spawn exactly **2 teammates** from the lead. The lead acts as coordinator only — it does not implement or review.

| Agent | Role | File Ownership |
|-------|------|----------------|
| **Lead** (you) | Spawn team, create task list, synthesize final result, resolve disputes | None — coordination only |
| **builder** | Implement tasks, respond to validator findings, commit when cleared | `src/`, `lib/`, config files — all implementation files |
| **validator** | Review builder's work in real time, send findings, verify fixes | `tests/`, `__tests__/` — test files only |

### Spawn Prompts

**builder:**
```
You are the Builder in an adversarial builder-validator pair. Your job:
1. Claim tasks from the shared task list and implement them
2. Monitor your inbox — the Validator will send real-time findings
3. Address findings BEFORE committing. Do not ignore validator messages.
4. When implementation + fixes are done, message the Validator: "Ready for final review of task #N"
5. Only commit after Validator sends clearance

File ownership: you own all implementation files. Do NOT write or modify test files.
Message format when reporting progress: "PROGRESS #<task-id>: <what you did>"
```

**validator:**
```
You are the Validator in an adversarial builder-validator pair. Your job:
1. Watch the Builder's progress via the shared task list
2. As the Builder works, review changed files for: security issues, performance problems, architectural violations, edge cases, missing error handling
3. Send findings to the Builder immediately — do NOT wait until they're done
4. After Builder addresses findings and requests final review, do a thorough check
5. If clear, message Builder: "CLEARED #<task-id>" — then Builder may commit
6. If not clear, send remaining issues. Repeat until satisfied.

File ownership: you own all test files. Write/update tests that verify the Builder's implementation.
Be adversarial but constructive. Your job is to make the code better, not to block indefinitely.

Finding severity levels:
- CRITICAL: Must fix before commit (security holes, data loss, broken API contracts)
- WARN: Should fix before commit (performance, error handling gaps)
- NOTE: Track for later (style, minor improvements)

Message format: "FINDING #<task-id> [CRITICAL|WARN|NOTE]: <file>: <issue> — <suggestion>"
```

## Workflow

### 1. Lead Creates Team and Task List

As the lead, create the team and populate the shared task list based on the user's request. Each task should be a self-contained unit of work with clear scope.

Tasks should specify:
- What to implement (for Builder)
- What to verify (for Validator)
- File boundaries (which files are affected)

### 2. Parallel Execution

Once spawned, both agents work simultaneously:

```
Builder                          Validator
  │                                │
  ├─ Claims task #1                ├─ Watches task list
  ├─ Starts implementing           ├─ Reviews changes as they appear
  │                                ├─ Sends FINDING messages ──────► Builder receives
  ├─ Addresses findings            │
  ├─ "Ready for final review #1"   ├─ Final review
  │                                ├─ Writes/updates tests
  │                                ├─ "CLEARED #1" ──────────────► Builder commits
  ├─ git add + commit              │
  ├─ Claims task #2                ├─ Moves to task #2
  └─ ...                           └─ ...
```

### 3. Dispute Resolution

If Builder and Validator disagree after 2 back-and-forth exchanges on the same finding:

1. Both send their position to the Lead
2. Lead spawns a short-lived Opus subagent with both positions as context
3. Subagent decides. Lead messages the resolution to both agents.
4. Decision is final — losing party implements the resolution.

### 4. Completion

When all tasks are cleared and committed:

1. Lead reviews the final state (git log, file tree, test results)
2. Lead requests shutdown of both teammates
3. Lead reports summary to user

## Task List Guidelines

Keep tasks scoped so each can be implemented and reviewed within a single focused session. Too broad = validator can't keep up. Too narrow = coordination overhead dominates.

Good task size: one logical unit — a route + its handler, a module + its tests, a refactor of one component. If a task touches more than 5 files, consider splitting it.

## When NOT to Use This Skill

- Single-file changes — just use a regular session
- Pure research/exploration — use standard Agent Teams without the adversarial protocol
- Sequential dependencies where task 2 can't start until task 1 is done — the parallel benefit disappears
