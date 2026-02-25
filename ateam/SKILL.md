---
name: ateam
description: >
  Orchestrate a full agile team of AI agents to implement features, fix bugs, or refactor code.
  Spins up a coordinated team with a Team Lead, Backend Engineers, Frontend Engineers, an optional
  Security Expert, and a Release Manager. Use when the user says "spin up the ATeam", "implement X
  with the ATeam", or any variation requesting the ATeam for development work. Supports greenfield
  features, bug fixes, and refactors. Agents work in parallel on isolated git worktrees with
  automated testing, optional security review, and branch merging into a PR.
---

# ATeam — Agile Team Orchestrator

Orchestrate a team of specialized agents through: **plan → implement+test → [optional security review] → release PR**.

## Team Composition

| Role | Count | Agent Type | Isolation |
|------|-------|------------|-----------|
| Team Lead | 1 | `general-purpose` | main worktree |
| Backend Engineer | 1–3 | `general-purpose` | isolated worktree each |
| Frontend Engineer | 1–3 | `general-purpose` | isolated worktree each |
| Security Expert | 1 | `general-purpose` | main worktree (**disabled by default**) |
| Release Manager | 1 | `general-purpose` | **own isolated worktree** |

## Security Expert — Opt-in

The Security Expert is **disabled by default**. It is only spawned and included in the workflow when the user explicitly requests it. Trigger phrases include:

- "run security audits during development"
- "with security review"
- "include security"
- `$ARGUMENTS` contains `--with-security`

If none of these triggers are present, **skip step 4 entirely** and go directly from implementation to release. Do not spawn the `security-expert` agent at all.

## Worktree Strategy

All engineers and the release manager work in **isolated git worktrees** created via `git worktree add`. This keeps the main worktree clean for the team lead. All worktrees are created inside the `.claude/worktrees/` directory within the repository root.

- **Engineers:** Each creates a worktree at `.claude/worktrees/<agent-name>-<name>` and works on a `feature/<name>` branch inside it. Engineers **never push** s their branches — all work stays local.
- **Release Manager:** Creates a worktree at `.claude/worktrees/release` on an `integration/<goal-slug>` branch. Merges all engineer branches **locally** (not from remote), verifies checks, then pushes the integration branch and opens a PR to the target branch.

After the release manager has merged all feature branches, engineers' worktrees and local branches are deleted. The release manager's worktree is cleaned up during shutdown.

## Workflow

### 1. Initialize the Team

Create the team and all agents using the `TeamCreate` and `Task` tools:

```
TeamCreate: team_name="ateam", description="ATeam: <brief goal>"
```

Spawn agents via the `Task` tool with `team_name: "ateam"`:

| Name | Role prompt | Spawned? |
|------|-------------|----------|
| `team-lead` | See references/team-lead.md | Always |
| `backend-1`, `backend-2`, `backend-3` | See references/engineer.md (backend variant) | Always (count based on scope) |
| `frontend-1`, `frontend-2`, `frontend-3` | See references/engineer.md (frontend variant) | Always (count based on scope) |
| `security-expert` | See references/security-expert.md | **Only if user opted in** |
| `release-manager` | See references/release-manager.md | Always |

All agents must receive the role prompt from the corresponding reference file as their initial task prompt. Append the specific goal/feature description after the role prompt.

**Important:** Include in each engineer's prompt the repository root path so they can create their worktree relative to it.

### 2. Plan (Team Lead)

The team lead:

1. Analyzes the codebase and requirements.
2. Breaks work into tasks via `TaskCreate` — each task tagged for `backend` or `frontend`.
3. Assigns tasks to engineers via `TaskUpdate` with `owner`.
4. Sets up `blockedBy`/`blocks` dependencies where needed.

**Strict rule:** Never assign frontend tasks to backend engineers or vice versa.

### 3. Implement + Test (Engineers, parallel)

Each engineer:

1. Claims their assigned task.
2. Creates an **isolated git worktree** inside `.claude/worktrees/` using `git worktree add`.
3. Creates a branch named `feature/<short-meaningful-name>`.
4. Implements the code and writes all necessary automated tests.
5. Runs manual verification if automated tests are insufficient.
6. Before finishing, ensures **all typecheck, lint, and test commands pass**.
7. **Commits locally** (does NOT push to remote).
8. Reports completion to the team lead via `SendMessage`, including their **branch name** and **worktree path**.

Engineers work in parallel — maximize concurrency. Branches remain local until the release manager merges them.

### 4. Security Review (Security Expert) — ONLY IF OPTED IN

**Skip this step entirely if the user did not request security review.**

Once implementation tasks are complete:

1. Team lead sends a message to `security-expert` to begin review.
2. Security expert reviews the changed code for vulnerabilities.
3. Reports findings back to team lead.
4. If critical issues are found, team lead assigns fix tasks back to the relevant engineers.
5. Repeat until the security expert gives the all-clear.

### 5. Release (Release Manager)

Once implementation is complete (and security review passes, if applicable):

1. Team lead sends a message to `release-manager` with all `feature/` branch names and the **target branch** (defaults to `main`).
2. Release manager creates an isolated worktree at `.claude/worktrees/release` on an `integration/<goal-slug>` branch.
3. Merges all feature branches **locally** (from local refs, not remote) into the integration branch.
4. If merge conflicts arise, resolves them carefully.
5. After all feature branches are merged, **cleans up engineer worktrees and deletes their local branches** (`git worktree remove` + `git branch -D`).
6. Runs **all typecheck, lint, and test commands** to confirm nothing is broken.
7. Pushes the integration branch and **opens a PR** to the target branch using `gh pr create`.
8. Reports the PR URL to the team lead.

### 6. Shutdown

After successful release:

1. Team lead sends `shutdown_request` to all agents.
2. Release manager cleans up its own worktree (`git worktree remove .claude/worktrees/release`). Engineer worktrees were already cleaned up during the release step.
3. Clean up with `TeamDelete`.

## Decision Making

The team lead makes all architectural and implementation decisions autonomously. When ambiguity arises from the user's request, the team lead resolves it based on codebase conventions and best practices — do not escalate to the user unless the decision fundamentally changes scope.

## Branch Naming

- **Engineer branches:** `feature/<short-meaningful-name>` (e.g., `feature/user-auth-api`, `feature/dashboard-filters`)
- **Integration branch:** `integration/<goal-slug>` (e.g., `integration/user-auth-feature`)

## Error Handling

- **Test failure:** Engineer fixes and re-runs before reporting done.
- **Merge conflict:** Release manager resolves in the integration worktree, then re-runs all checks.
- **Security issue:** Team lead assigns a fix task to the responsible engineer, then re-triggers security review.
- **Blocked task:** Engineer notifies team lead, who reassigns or unblocks.

## References

Role-specific prompts for each agent (read and include when spawning):

- [Team Lead](references/team-lead.md)
- [Engineer](references/engineer.md) — covers both backend and frontend variants
- [Security Expert](references/security-expert.md) — **only spawn if opted in**
- [Release Manager](references/release-manager.md)
