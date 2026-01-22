Source: https://github.com/steveyegge/beads/issues/976#issuecomment-3734123713

# Recommended Approach: Planning Docs + Lightweight Beads

## Pattern: Planning docs + lightweight beads

### 1. Keep detailed plans in separate docs

Store planning docs in `docs/plans/` or similar. Beads are intentionally lightweight for tracking execution, not storing full specifications.

### 2. Reference plans from beads

Use the description field to link back:

```bash
bd create "Implement auth flow phase 1" --desc "See docs/plans/auth-design.md#phase-1"
```

### 3. Use epics for grouping

Create an epic for the overall plan, then child issues for steps:

```bash
bd create "Auth system redesign" --type epic
bd create "Add OAuth provider" --parent bd-xxx
bd create "Implement token refresh" --parent bd-xxx
```

### 4. Automated plan-to-beads conversion

You can script this! Parse your planning doc and generate beads:

```bash
# Example: create beads from a markdown checklist
grep '^- \[ \]' plan.md | while read line; do
  title=$(echo "$line" | sed 's/^- \[ \] //')
  bd create "$title" --parent bd-epic-id
done
```

### 5. Templates

Use `bd create --template` for recurring patterns with pre-filled descriptions.

---

## The Philosophy

> Beads track **what** to do and dependencies, while docs capture **how** to do it and **why**.

This separation keeps the issue tracker fast and scannable while preserving rich context elsewhere.
