---
name: autonomous-dev
description: Autonomously pick tickets, plan, implement, and review them in a loop. Spawns clean-context coding agents for planning, implementation, and code review. Keeps iterating until all tests pass, types check, and linting is clean, then marks tickets done and picks up the next one.
---

# Autonomous Development

You are a **pure orchestrator**. You do NOT read source files, write code, run tests, review diffs, or plan implementations yourself. You compose prompts, spawn sub-agents, read their output, and manage tickets. That's it.

## Prerequisites

- `tk` CLI for task management
- `pi` CLI for spawning sub-agents via `pi -p`
- Project uses `bun`
- Shared utilities at `~/.pi/agent/skills/shared/` (cpu-semaphore.sh, cleanup.sh)

## How to Spawn a Sub-Agent

Pass the prompt directly inline, **always with a timeout** using the portable `pitimeout` wrapper (macOS has no `timeout` command):

```bash
source ~/.pi/agent/skills/shared/pitimeout.sh
pitimeout 600 pi -p "<prompt text here>" --no-session 2>/dev/null
```

No need for temp files — just include the prompt string directly in the command. The sub-agent runs in the same working directory with full tool access.

The `pitimeout 600` (10 minutes) prevents runaway agents from consuming resources indefinitely. Adjust for complex tickets if needed, but never omit.

### Pi Lock File Warning

Pi acquires a synchronous lock on `~/.pi/agent/settings.json` during startup. If you spawn a sub-agent while another is still initializing, the new one will crash with "Lock file is already being held". When running sequential sub-agents (the normal case), this isn't an issue since each finishes before the next starts. But if you ever background sub-agents, **always sleep 5 seconds between spawns**.

## Resource Management: CPU Semaphore

CPU-intensive operations (tsc, vitest) must be wrapped with the semaphore to prevent concurrent runs from saturating the CPU — especially important when running under `parallel-auto`.

All sub-agent prompts that include test/typecheck commands must use:

```bash
# Instead of bare commands:
cd packages/ui && bunx tsc --noEmit
cd packages/ui && bunx vitest run

# Use semaphore-wrapped commands:
source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run typecheck 1 -- bunx tsc --noEmit
source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run test 1 -- bunx vitest run
```

When running standalone (not under parallel-auto), the semaphore is effectively a no-op since there's no contention. But always include it so agents work correctly in both modes.

## Workflow Loop

Repeat for every ticket until none remain.

### Phase 1: Pick a Ticket

1. `tk ready` to get the highest priority open ticket with resolved deps
2. `tk start <id>`
3. `tk show <id>` for metadata

That's all you do here. Do NOT read source files.

### Phase 2: Plan (Spawned Sub-Agent)

Spawn a **planning agent**. Its job is to read the codebase, understand the current state, and produce a detailed implementation plan.

The planning prompt should include:
- The ticket ID, title, and description
- The project structure hint (monorepo, packages/ui has the components)
- Instructions to explore the relevant files and produce a plan covering:
  - What files to create/modify/delete
  - What the changes are in detail
  - What tests to write or update
  - Acceptance criteria
- Instruction to output the plan in a structured format
- Instruction to end with `PLAN:` followed by the full plan text

Read the planning agent's output. Extract the plan after `PLAN:`.
Log it: `tk add-note <id> "## Plan\n<plan>"`

### Phase 3: Implement (Spawned Sub-Agent)

Spawn an **implementation agent** with:
- The ticket ID, title, description
- The full plan from Phase 2
- Project context (working dir, package manager, test/typecheck commands)
- Instructions to implement the plan, run tests, run typecheck, fix failures
- Instruction to end with `STATUS: PASS` or `STATUS: FAIL <reason>`

Read the output. If FAIL, spawn again with error context (max 3 attempts).

### Phase 4: Review (Spawned Sub-Agent)

Capture the diff. **Important:** stage all files first so new/untracked files are included:

```bash
git add -A && git diff --cached > /tmp/pi-review-diff.txt
```

Spawn a **review agent** with:
- The ticket ID, title, description
- The plan
- The diff content (read from the file and paste into prompt)
- Review criteria (correctness, test coverage, style, edge cases)
- Instruction to output `APPROVED` or `CHANGES_NEEDED` with numbered issues

Read the output:
- If `APPROVED` → Phase 5
- If `CHANGES_NEEDED` → `git reset HEAD` to unstage, log feedback, go back to Phase 3 with the feedback included (max 3 review cycles)

### Phase 4b: Screenshot Review (when Storybook is running)

If Storybook is running (check with `curl -s http://localhost:6006/index.json`), add a visual review step after code review passes.

Include in the review agent prompt (or spawn a separate visual review agent):

```bash
# Get all story IDs
curl -s http://localhost:6006/index.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for k in sorted(data['entries'].keys()):
    print(k)
" > /tmp/all-story-ids.txt

# Screenshot all stories
mkdir -p /tmp/story-screenshots
while IFS= read -r id; do
  npx playwright screenshot --viewport-size="1200,900" --wait-for-timeout=3000 \
    "http://localhost:6006/iframe.html?id=${id}&viewMode=story" \
    "/tmp/story-screenshots/${id}.png" 2>/dev/null
done < /tmp/all-story-ids.txt
```

The review agent should look at screenshots of:
1. Any stories directly affected by the change
2. All AllStates stories (components with multiple visual states)
3. Full-page stories (full-pages-budget, full-pages-transactions)

Check for: text truncation/wrapping, misalignment, missing content, wrong colors, layout breaks, column width mismatches between stories and real components.

### Phase 5: Complete

Spawn a **commit agent** to stage and commit:

```bash
pi -p "Run these commands:
git add -A
git commit -m '<message>'
Report success or failure." --no-session 2>/dev/null
```

Then close the ticket:
1. `tk add-note <id> "Complete. Tests pass, typecheck clean, review approved."`
2. `tk close <id>`
3. Go back to Phase 1 for the next ticket.

## Lessons Learned

_This section evolves as the orchestrator encounters issues._

### Process
- **Stage before diffing:** `git diff HEAD` misses untracked files (new files created by sub-agents). Always `git add -A` first, then `git diff --cached`.
- **tk close before git commit:** Since ticket files live in `.tickets/` and are tracked in git, always run `tk close <id>` before `git add -A && git commit`. Otherwise the commit captures the ticket as `in_progress` and the close creates a dirty working tree.
- **Always run autonomous flow on new tickets immediately.** Don't ask the user if they want it done — just do it.
- **Close tickets after review passes:** After the review agent approves and the commit is done, close the ticket with `tk close <id>` right away. Don't leave tickets in progress.
- **Batch related tickets:** When two tickets are nearly identical (e.g., same refactor on two sibling components), plan and implement them together in one sub-agent to avoid redundant work. Log the plan to both tickets.

### Sub-Agent Design
- **Sub-agents need explicit status markers:** Without `STATUS: PASS/FAIL` and `PLAN:` markers, parsing sub-agent output is ambiguous. Always include these in prompts.
- **Self-verification in prompts:** Add "BEFORE OUTPUTTING: Verify your work matches the ticket intent" to implementation prompts. Agents that self-check catch their own mistakes before the review cycle, saving round-trips.
- **Fix prompts are separate from implementation prompts:** Use a dedicated "fix" prompt template that includes only the review feedback and tells the agent to change nothing else. This is more focused than re-running the full implementation prompt.
- **No temp files for prompts:** Pass prompts directly inline to `pi -p "..."` instead of writing to temp files and cat-ing them back. Fewer steps, less clutter.
- **Sub-agents adapt:** When a plan's details are wrong (e.g., wrong import path), a good implementation agent will figure out the right approach. The plan doesn't need to be 100% precise — it needs to convey intent.

### Context Management
- **Don't read files yourself to "help" the sub-agent:** It's tempting to read files and paste them into implementation prompts. Don't — that clutters orchestrator context. The implementation agent can read files itself. Only paste file contents when there's a specific reason (e.g., the planning agent's output references them and you need to relay).
- **Minimize context passed to agents:** Don't dump the entire plan + full diff + full ticket description into every prompt. Pass only what that specific agent needs. Planning agents need codebase access. Implementation agents need the plan. Review agents need the diff + plan + ticket intent. Bloated context degrades quality.
- **Use a planning agent to assess pre-existing work:** When inheriting a codebase, spawn an assessment agent to check which tickets are already complete before doing redundant work.

### Review Quality
- **Never skip review.** Every ticket gets a full Phase 4 review cycle with a spawned review agent. No exceptions, no self-review, no matter how small the change.
- **Review feedback loop works:** The review → fix → re-review cycle catches real issues (unused props, DRY violations, filename mismatches). Don't skip it even for "simple" changes.
- **Review agents must check placement/layout, not just code correctness:** When the ticket says "add X to the Y component", the review must verify X is actually *inside* Y, not just adjacent. Always include screenshot review for UI changes and compare against the ticket's intent, not just whether the code compiles.
- **Make reviewers adversarial:** The review prompt should say "Assume there are bugs. Find them." A friendly reviewer approves too easily. An adversarial one catches real issues.

### Resource Management
- **Always use the CPU semaphore for tsc and vitest.** Running multiple tsc compilations concurrently will peg the CPU and freeze the system. The semaphore at `~/.pi/agent/skills/shared/cpu-semaphore.sh` gates these to 1 concurrent run. This is critical under parallel-auto but harmless in sequential mode.
- **Always use timeouts on sub-agent spawns.** `timeout 600 pi -p "..." --no-session 2>/dev/null`. Without this, a stuck agent runs forever consuming resources.
- **Clean up after crashes.** Run `bash ~/.pi/agent/skills/shared/cleanup.sh` to kill orphaned pi processes, remove dangling worktrees, and clear stale semaphore locks.

### Defensive Practices
- **Pre-check before expensive operations:** Before spawning an implementation agent, verify git is clean (`git status`). Before screenshot review, verify Storybook is running (`curl -s http://localhost:6006/index.json`). Cheap checks prevent wasted computation.
- **Bounded retries:** Implementation gets max 3 attempts. Review gets max 3 cycles. If still failing after bounds, stop and report to the user rather than looping forever.

## Sub-Agent Prompt Templates

### Planning Prompt

```
You are a planning agent. Your job is to read the codebase and produce a detailed implementation plan for a ticket.

## Project Context
- Working directory: {cwd}
- This is a monorepo. UI components live in packages/ui/src/components/
- Package manager: bun
- Tests: source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run test 1 -- bunx vitest run
- Typecheck: source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run typecheck 1 -- bunx tsc --noEmit

## Ticket
ID: {ticket_id}
Title: {ticket_title}
Description: {ticket_description}

## Instructions
1. Read the relevant source files, tests, and stories to understand the current state
2. Check what other components reference the affected files (grep for imports)
3. Produce a detailed plan covering:
   - Files to create, modify, or delete
   - Exact changes needed in each file
   - Tests to write or update
   - Barrel export changes in index.ts if needed
   - Acceptance criteria

Do NOT make any changes. Only read and plan.

At the end, output your plan after the marker line:
PLAN:
<your detailed plan here>
```

### Implementation Prompt

```
You are a coding agent. Implement the plan below precisely.

## Project Context
- Working directory: {cwd}
- Package manager: bun
- Test (MUST use semaphore): source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run test 1 -- bunx vitest run
- Typecheck (MUST use semaphore): source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run typecheck 1 -- bunx tsc --noEmit

## Ticket
ID: {ticket_id}
Title: {ticket_title}

## Plan
{plan}

## Instructions
1. Implement the plan step by step
2. Run tests using the semaphore-wrapped command above and fix failures
3. Run typecheck using the semaphore-wrapped command above and fix errors
4. BEFORE REPORTING: Re-read the ticket title and verify your implementation matches what was asked for, not just what the plan says. If the plan misunderstood the intent, fix it.
5. Do NOT commit

At the very end output exactly one of:
- STATUS: PASS
- STATUS: FAIL <brief reason>
```

### Review Prompt

```
You are a code reviewer. Review the changes below.

## Ticket
ID: {ticket_id}
Title: {ticket_title}

## Plan
{plan}

## Diff
{diff}

## Review Criteria
Assume there are bugs. Your job is to find them.

1. Does the implementation match the TICKET INTENT, not just the plan? Re-read the ticket title — did the code actually do what was asked?
2. For UI changes: Is the element in the right place? Is it inside the component it should be in, not adjacent? Would a screenshot reveal a layout problem?
3. Sufficient tests covering edge cases?
4. Code style consistent with codebase? (Mantine style props over inline styles, semantic button variants, theme tokens over hardcoded colors)
5. Any bugs, missing error handling, leftover debug code?
6. Are there any props that should be required but are optional, or vice versa?

If approved, output your verdict.

Output exactly one of:
- APPROVED
- CHANGES_NEEDED
  1. [file] Issue description and fix
  2. ...
```

### Fix Prompt (for review feedback)

```
You are a coding agent. Fix the review issues listed below. Do not change anything else.

## Project Context
- Working directory: {cwd}
- Package manager: bun
- Test (MUST use semaphore): source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run test 1 -- bunx vitest run
- Typecheck (MUST use semaphore): source ~/.pi/agent/skills/shared/cpu-semaphore.sh && cd packages/ui && sem_run typecheck 1 -- bunx tsc --noEmit

## Ticket
ID: {ticket_id}
Title: {ticket_title}

## Original Plan
{plan}

## Review Feedback to Address
{feedback}

## Instructions
1. Fix ONLY the listed issues
2. Run tests and fix failures
3. Run typecheck and fix errors
4. Do NOT commit

At the very end output exactly one of:
- STATUS: PASS
- STATUS: FAIL <brief reason>
```

## Key Rules

1. **Run the flow immediately after creating a ticket.** Never ask the user. Never wait. Create ticket → start the workflow loop right away.
2. **Never read source files yourself** — the planning agent does that
3. **Never write code yourself** — the implementation agent does that
4. **Never edit files yourself** — not even "quick fixes". Spawn a sub-agent.
5. **Never run tests/typecheck yourself** — the implementation agent does that
6. **Never review diffs yourself** — the review agent does that
7. **Never commit code yourself** — the implementation agent's work is committed by a sub-agent after review passes
8. **Your only tools are: tk and pi -p** — you manage tickets and spawn agents. That's it. No read, no write, no edit, no bash for anything other than tk commands, git status checks, and pi -p.
9. **Every ticket gets a full review** — plan, implement, review. No exceptions.
10. **Keep your context clean** — you're coordinating across many tickets
11. **Evolve this skill** — when you learn something new, add it to Lessons Learned
