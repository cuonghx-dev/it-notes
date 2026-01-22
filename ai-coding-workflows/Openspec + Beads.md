- Source: https://github.com/steveyegge/beads/discussions/240#discussioncomment-14916427

# Beads vs OpenSpec: Complementary Tools for AI-Assisted Development

After analyzing both projects, adding Beads to your existing OpenSpec workflow would provide significant value—but in a way that complements rather than replaces OpenSpec. These tools solve different problems in the AI coding workflow.

## What Each Tool Does

**OpenSpec** is a specification-driven development framework that helps you and your AI agent agree on *what* to build before writing code. It uses a proposal → apply → archive workflow where you:

- Create change proposals with specification deltas (ADDED/MODIFIED/REMOVED requirements)
- Generate `tasks.md` that breaks down implementation steps
- Archive completed changes by merging spec deltas into the source of truth

OpenSpec excels at brownfield development (modifying existing code) because it separates current specs (`openspec/specs/`) from proposed changes (`openspec/changes/`), making it clear what's changing and what's staying the same.

**Beads** is a graph-based issue tracker designed as a memory system for coding agents. It addresses the "amnesia problem" where AI agents lose context across sessions or during compaction. Key features:

- Distributed database backed by git (JSONL files + SQLite cache)
- Four dependency types: blocks, related, parent-child, discovered-from
- `bd ready` command shows unblocked work automatically
- Hash-based IDs (v0.20.1+) prevent merge conflicts in multi-worker scenarios
- Persistent tracking of discovered issues during implementation

### The Key Difference: Planning vs Execution Tracking

> **OpenSpec** answers "WHAT should we build?" while **Beads** answers "WHERE are we in building it?"

## When Adding Beads Makes Sense

Based on your current use of OpenSpec, adding Beads would be valuable if you experience these scenarios:

- **Complex features with dependencies** - Beads' dependency graph (blocks, parent-child) can enforce the ordering suggested in OpenSpec's `tasks.md`
- **Multi-week projects** - Beads provides persistent memory so agents can pick up exactly where they left off
- **Agent context loss** - When your agent forgets what it was working on between sessions, Beads solves this with `bd ready` showing the next unblocked task
- **Discovering work during implementation** - Beads' `discovered-from` dependency automatically files issues for bugs/tasks found while coding
- **Multi-agent or team workflows** - Hash-based IDs (`bd-a1b2`) prevent collisions when multiple agents create issues concurrently

## Integration Workflow

Here's how they work together:

### Phase 1: Planning with OpenSpec

```bash
# Create OpenSpec proposal
openspec init  # (you've already done this)
# AI creates: proposal.md, tasks.md, spec deltas
```

### Phase 2: Seed Beads from OpenSpec

```bash
bd init
# Convert OpenSpec tasks.md into Beads issues
# Example: each task in tasks.md → bd create issue
bd create "Task 1.1: Create database schema" -p 1 -t task
bd create "Task 1.2: Build API endpoints" -p 1 -t task
bd dep add bd-f14c bd-a1b2 --type blocks  # 1.2 blocks on 1.1
```

### Phase 3: Implementation Tracking

```bash
# Agent checks ready work
bd ready --json

# Agent implements from OpenSpec specs
# Updates Beads as work progresses
bd update bd-a1b2 --status in_progress
bd close bd-a1b2 --reason "Completed per spec"

# Discovers new issue during work
bd create "Found validation bug" -t bug -p 0
bd dep add bd-3e7a bd-a1b2 --type discovered-from
```

### Phase 4: Archive

```bash
# Close all Beads issues for this OpenSpec change
bd close bd-f14c bd-3e7a bd-x1y2 --force

# Archive OpenSpec change
openspec archive add-feature-name --yes
```

## Practical Benefits of Combining Both

**Context Continuity:** OpenSpec provides the "source of truth" specs, while Beads maintains the work queue across sessions. An agent can always:
- Read OpenSpec specs to understand requirements
- Query `bd ready` to find the next unblocked task
- Update Beads status as work progresses

**Discovery Tracking:** During implementation, agents naturally discover edge cases, bugs, or refactoring needs. With OpenSpec alone, these get lost unless you manually update `tasks.md`. With Beads, the agent can immediately `bd create` discovered work with `--type discovered-from` to link it back to the parent task.

**Multi-Worker Safety:** If you're running multiple AI agents or working across branches, OpenSpec's `tasks.md` can have merge conflicts. Beads' hash-based IDs (since v0.20.1) are collision-resistant, making multi-worker scenarios much smoother.

**Audit Trail:** OpenSpec's spec deltas show *what* changed in requirements. Beads' audit trail shows *how* those changes were implemented (which issues were filed, dependencies, status transitions).

## When You Might NOT Need Beads

Based on your OpenSpec usage, you can skip Beads if:

- You're doing simple 1-2 day features where OpenSpec's `tasks.md` is sufficient
- You're working solo with a single agent and short sessions
- Your projects don't have complex dependency chains
- You rarely discover new work during implementation

## Community Interest

There's already community interest in this integration. Issue #245 in the OpenSpec GitHub repo is a feature request for Beads integration, and Reddit discussions mention users trying to "integrate Spec Kit with beads to maintain a clear focus on the relevant context". This suggests others are seeing the same complementary value.

## Recommendation

**Add Beads if you experience any of these:**
- Long-running projects (2+ weeks)
- Complex dependency chains
- Multi-agent coordination needs
- Frequent context loss between sessions
- Discovering work during implementation that gets lost

**Stick with OpenSpec only if:**
- Your features are straightforward (1-2 days)
- Single agent, short sessions
- Simple linear task flows

## Getting Started

If you decide to add Beads:

```bash
# Install Beads
curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash

# Initialize in your project
cd your-project
bd init --quiet  # Non-interactive setup

# Tell your AI agent
echo "Use 'bd' to track implementation of OpenSpec tasks" >> AGENTS.md
bd onboard  # Shows agent integration instructions
```

The setup is minimal (under 1 minute), and both tools coexist peacefully in your git repo - OpenSpec in `openspec/`, Beads in `.beads/`.

---

*Given your technical background and experience with multi-LLM evaluation, self-hosted infrastructure, and agentic systems, you'd likely appreciate Beads' distributed-database-via-git architecture and its ability to provide persistent agent memory across the complex workflows you're building.*
