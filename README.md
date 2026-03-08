- Project: https://github.com/linuxcaffe/tw-deps-mgmt-hook
- Issues:  https://github.com/linuxcaffe/tw-deps-mgmt-hook/issues

# tw-deps-mgmt-hook

Keeps your Taskwarrior dependency chains intact when tasks are completed or deleted.

## TL;DR

- When you complete a task, see immediately which tasks it unblocks
- When you delete a task that others depend on, review the full orphan tree before anything is deleted
- Choose per-orphan: delete it too, or keep it and clear the stale reference
- `a` accepts all remaining orphans at once; `q` / Esc aborts the entire deletion
- All decisions are collected first — nothing executes until you confirm the primary task
- Companion `tw deps` command draws the full dependency tree for any filter
- Requires Taskwarrior 2.6.2

## Why this exists

Taskwarrior tracks dependencies between tasks, but it doesn't protect you when you delete a task that others depend on. The dependent tasks silently become orphans — their `depends` field points to a UUID that no longer exists. You won't notice until you run `task diag` or wonder why a task is still blocked by nothing.

Completing a blocker is the happy path: Taskwarrior unblocks dependents automatically. But you still don't get any feedback. If you're mid-project with several blocked tasks, knowing which ones just became actionable is useful — you shouldn't have to `task next` and guess.

The built-in `task delete` prompts for confirmation but says nothing about downstream tasks. A three-level chain — prerequisite → task → subtask — can be silently half-destroyed in two keystrokes.

This hook intercepts both events. Completion gets a one-line summary. Deletion gets a full interactive review before a single dependent is touched.

## What this means for you

You can see which dependencies are active and what they connect, and you won't end up with broken chains. Deleting a blocker becomes a deliberate act — you review what depends on it, decide the fate of each, and nothing changes until you say so.

## Core concepts

**Blocker** — A task that must be completed before another task can start. In Taskwarrior, if task B has `depends:A`, then A is the blocker and B is blocked by A.

**Orphan** — A task whose blocker has been deleted. Its `depends` field contains a UUID that no longer exists. The hook prevents orphans from forming silently.

**Cascade** — Deleting a blocker may reveal that its dependents are themselves blockers. The hook follows the chain to full depth and asks about each level before executing anything.

## Installation

### Option 1 — Install script

```bash
chmod +x deps-mgmt.install
./deps-mgmt.install
```

Installs `on-modify_deps-mgmt.py` to `~/.task/hooks/` and the `deps` companion script to `~/.task/scripts/`.

### Option 2 — Via [awesome-taskwarrior](https://github.com/linuxcaffe/awesome-taskwarrior)

```bash
tw -I deps-mgmt
```

### Option 3 — Manual

```bash
# Copy the hook
cp on-modify_deps-mgmt.py ~/.task/hooks/
chmod +x ~/.task/hooks/on-modify_deps-mgmt.py

# Copy the companion script
cp deps ~/.task/scripts/
chmod +x ~/.task/scripts/deps

# Register the deps wrapper (append to your .tw_wrappers file)
echo 'deps|deps|Draw dependency tree for matching tasks|command' >> ~/.task/config/.tw_wrappers

# Verify
task 1 delete   # if task 1 has dependents, you should see the [deps] prompt
```

## Usage

### Completing a blocker

When you complete a task that was blocking others, the hook prints which tasks are now unblocked:

```
[deps] 'Write API spec' unblocks: 'Implement endpoints', 'Write client tests'
```

No action required. Taskwarrior handles the unblocking; the hook adds visibility.

### Deleting a task with dependents

```bash
task 14 delete
```

If task 14 has dependents, the hook intercepts and shows the full orphan tree:

```
[deps] Deleting 'Write API spec' will orphan 2 task(s):
  [18] Implement endpoints
    [22] Write client tests

  Delete [18] 'Implement endpoints'? [Y/n/a/q]
  Delete [22] 'Write client tests'? [Y/n/a/q]
  Delete [14] 'Write API spec' (primary)? [Y/n/q]

[deps] Deleted [22] 'Write client tests'
[deps] Deleted [18] 'Implement endpoints'
[deps] 3 tasks deleted total (including primary)
```

**Interactive keys:**

| Key | Action |
|-----|--------|
| `Y` / Enter | Delete this task |
| `n` | Keep task, strip the stale dependency reference |
| `a` | Yes to all remaining (primary still asked last) |
| `q` / Esc | Abort — nothing is deleted, original task is unchanged |

Decisions are collected first. Execution happens bottom-up (deepest tasks first) only after you confirm the primary.

### Viewing dependency trees

```bash
tw deps                  # all dep trees (roots = unblocked blockers)
tw 14 deps               # tree rooted at task 14
tw +project deps         # trees for tasks matching +project
tw +work +active deps    # combined filter
```

Example output:

```
[14] Write API spec
├── [18] Implement endpoints
│   └── [22] Write client tests
└── [19] Update docs
```

Tasks with a `wait` date show `(waiting)`. Tasks that are both blocked and blocking show `(also blocked)`.

## Example workflow

1. You have a three-task chain: `[5] Research → [8] Draft → [12] Review`
2. Scope changes. You want to drop the whole chain.
3. `task 5 delete` — the hook shows the full tree and asks about 8, then 12, then 5
4. You press `a` to accept all, then `Y` for the primary
5. All three are deleted bottom-up. No orphans. No broken references.

Or: you want to keep the Draft and Review but drop the Research prerequisite:

1. `task 5 delete`
2. At `Delete [8] 'Draft'?` press `n` — keeps the task, clears the stale dep on 5
3. At `Delete [12] 'Review'?` press `n` — same
4. At `Delete [5] 'Research' (primary)?` press `Y`
5. Task 5 is deleted. Tasks 8 and 12 remain, now unblocked.

## Project status

⚠️ Early release. Core behaviour is working and tested. Edge cases (diamond dependencies, very deep chains) may surface with real-world data — please report them.

## Metadata

- License: MIT
- Language: Python 3.6+
- Requires: Taskwarrior 2.6.2
- Platforms: Linux
- Version: 0.1.0
