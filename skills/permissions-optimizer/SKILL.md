```md
---
name: permissions-optimizer
description: Optimize Claude Code permissions in settings files. Deduplicate and normalize allow/deny rules, consolidate via safe wildcards, resolve global/project conflicts, and produce a reviewable diff plan with minimal-risk defaults. Triggers: "optimize permissions", "permission optimizer", "clean permissions", "tidy settings.json", "permissions policy".
---

# Permissions Optimizer

A skill to optimize Claude Code permission allow/deny rules with **least-privilege defaults**, clear scoping (global vs project), and a review-friendly change plan.

## Goals

- Reduce permission sprawl (duplicates, one-offs, noisy entries)
- Normalize patterns and consolidate where **provably safe** (see §Consolidation Safety Criteria)
- Resolve global/project conflicts and clarify exceptions
- Keep risky operations tight: **deny by default**, allow only with explicit user approval
- Output a clear plan: **what changes**, **where**, and **why**

---

## Settings File Schema

Claude Code settings files use the following JSON structure:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run build)",
      "Bash(npm test)",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Read(*)",
      "Edit(*)"
    ],
    "deny": [
      "Bash(sudo *)",
      "Bash(rm -rf *)",
      "Bash(curl *)"
    ]
  }
}
```

### Pattern syntax

| Format | Meaning | Example |
|---|---|---|
| `Tool(exact command)` | Matches that exact command only | `Bash(npm test)` |
| `Tool(prefix:*)` | Matches any command starting with `prefix:` | `Bash(git diff:*)` |
| `Tool(*)` | Matches all invocations of that tool | `Read(*)` |

Tools include: `Bash`, `Read`, `Edit`, `WebFetch`, `MCP` (and others as Claude Code evolves).

> **Important**: Whitespace inside parentheses is significant. `Bash(npm run build)` and `Bash(npm  run  build)` are different patterns. Normalize only when the extra whitespace is clearly accidental (e.g., trailing spaces).

---

## Inputs (files to read)

Read these three files using absolute paths (expand `~` via `$HOME`):

```bash
cat "$HOME/.claude/settings.json"          # Global
cat .claude/settings.json                  # Project shared (repo root)
cat .claude/settings.local.json            # Project local (repo root)
```

### Notes

- `$HOME/.claude/settings.local.json` is **not loaded** by Claude Code (only project `.claude/settings.local.json` matters).
- Global and project files may be symlinked to the same target; detect with `readlink -f` and avoid double-editing.
- Changes require a **new session** to take effect.
- Handle missing files gracefully — treat as `{ "permissions": { "allow": [], "deny": [] } }`.

---

## Guardrails (non-negotiable)

- Do **not** auto-allow high-risk actions (destructive ops, privilege escalation, external access).
- Prefer **deny broader / allow narrower**.
- Keep `.claude/settings.local.json` **empty by default** (temporary exceptions only).
- Always propose changes as a **plan/diff** first; apply only after explicit user approval.
- Write changes using `Edit` tool or output the final JSON for user to copy — never silently overwrite.

---

## Workflow

### 1) Read & Parse

- Load and parse the three settings files (handle missing files gracefully).
- Resolve `$HOME` to an absolute path; detect symlinks with `readlink -f`.
- Extract allow/deny entries and record scope:
  - `global`, `project(shared)`, `project(local)`

### 2) User Intent Confirmation

**Before** building the optimization plan, ask the user batch questions to understand their threat model. This prevents wasted work and ensures the plan reflects actual needs.

Use the following compact checklist (present via AskUserQuestion):

1. **Network commands** (`curl`, `wget`, etc.)
   - No (prefer WebFetch) — **default**
   - Yes (API testing) → allow with narrow patterns
2. **Git remote operations** (`git fetch/pull/push`)
   - Ask each time — **default**
   - Allow in project only
   - Allow globally
3. **Destructive operations** (`docker prune`, `rm -rf`, `mkfs`, etc.)
   - Keep denied — **default**
   - Allow a specific safe subset (user specifies exact commands)
4. **Package managers** (`npm`, `pip`, `poetry`, etc.)
   - Project shared only — **default**
   - Global allow
   - Local only (temporary)

Carry the answers forward into Steps 4 and 5.

### 3) Inventory & Normalize

Create a normalized inventory per rule:

| Field | Values |
|---|---|
| action | allow / deny |
| tool | Bash / Edit / Read / WebFetch / MCP / etc. |
| pattern | command/path/wildcard |
| scope | global / project(shared) / project(local) |
| flags | one-off, wildcardable, conflict, env-specific, risky |

#### Normalization rules (safe transforms only)

| Transform | Example | Safe? |
|---|---|---|
| Trim trailing whitespace | `Bash(npm test )` → `Bash(npm test)` | ✅ Always |
| Remove duplicate entries | Two identical `Bash(npm test)` in same file | ✅ Always |
| Collapse internal multi-spaces | `Bash(npm  run  build)` → `Bash(npm run build)` | ⚠️ Only if clearly accidental |
| Merge subcommand into wildcard | `Bash(git diff)` + `Bash(git diff --stat)` → `Bash(git diff:*)` | ⚠️ Only if **all** `git diff` variants are safe |
| Merge different commands | `Bash(git push)` + `Bash(git push origin main)` → `Bash(git push:*)` | ❌ Semantically different — do not merge without explicit approval |
| Normalize quoting style | `Bash("npm test")` → `Bash(npm test)` | ❌ Claude Code may treat these differently; preserve as-is |

> **Principle**: When in doubt, do not normalize. Preserving a redundant rule is cheaper than breaking a permission boundary.

### 4) Diagnose Issues

Detect and tag rules under these categories:

- **Exact duplicates** — identical rule in the same file
- **Cross-scope duplicates** — same rule in global and project (project wins; global copy may be removable)
- **One-time permissions** — over-specific, unlikely to repeat (e.g., `Bash(cat /tmp/debug-2024-01-15.log)`)
- **Wildcardable** — narrow rules fully covered by a safe wildcard (see §Consolidation Safety Criteria)
- **Scope conflicts** — global deny vs project allow, or contradictory entries within the same file
- **Environment-specific paths** — absolute machine paths (`/Users/alice/...`) that break portability
- **Destructive operations** — delete/prune/format/recursive permission changes
- **Over-broad allow** — patterns that permit too much (e.g., `Bash(*)`)

### 5) Build an Optimization Plan (3 tiers)

**Tier 1 — Safe automatic candidates (recommend by default)**

- Remove exact duplicates (same file)
- Remove rules strictly subsumed by an existing wildcard **that is itself Tier 1**
- Remove cross-scope duplicates where the surviving rule has equal or tighter scope
- Trim clearly accidental whitespace

**Tier 2 — Needs batch user confirmation** (informed by Step 2 answers)

- Network-capable commands (`curl`, `wget`, `git fetch/pull/push`)
- Package installs (`npm`, `pip`, etc.)
- Scope relocation (global ↔ project)
- Wildcard consolidation of related commands (e.g., `npm run *`)

**Tier 3 — High-risk (default deny; confirm individually)**

- Privilege escalation: `sudo`, `su`
- Remote access: `ssh`, `scp`, `rsync` to remote targets
- Destructive: `rm -rf`, `mkfs`, `dd`, `docker system prune -a`, broad `chmod -R`, `chown -R`
- Broad environment leaks: `env`, `printenv` (if secrets are possible)
- Over-broad patterns: `Bash(*)`

### 6) Placement Rules (where each rule should live)

| Location | Put here |
|---|---|
| Global allow | Universal, low-risk, commonly used (`Read(*)`, `Edit(*)`, inspection tools) |
| Global deny | High-risk primitives and broad patterns you never want silently allowed |
| Project shared | Project-specific commands (build/test scripts, local DB CLIs, etc.) |
| Project local | Temporary exceptions only; keep near-empty |

Decision flow:

1. High-risk → **global deny** (exceptions must be narrow + approved)
2. Universal & low-risk → **global allow**
3. Project-only → **project shared**
4. Temporary / machine-specific → **project local**

### 7) Apply Changes

After user approval:

- Use the `Edit` tool to modify each settings file in place.
- Alternatively, output the complete final JSON for each modified file so the user can copy-paste.
- Never silently overwrite — always confirm before writing.
- After writing, print a short summary of files changed.

---

## Consolidation Safety Criteria

A set of narrow rules may be consolidated into a wildcard **only when all of the following hold**:

1. **Complete coverage**: Every command matched by the proposed wildcard is already individually allowed (no new commands become permitted).
2. **Low risk**: The wildcard falls within Tier 1 or Tier 2 (never auto-consolidate into a Tier 3 pattern).
3. **Same scope**: All source rules share the same scope, or the target scope is equal or tighter.
4. **No deny conflict**: No existing deny rule would be weakened or contradicted.

If any condition fails, keep the narrow rules as-is and flag for manual review.

---

## Output Format (what to present)

### A) Summary

- Rule counts before/after (per file and total)
- How many removed / merged / moved / added
- Number of conflicts found
- High-risk items requiring explicit confirmation

### B) Proposed Changes (reviewable plan)

For each change:

| Field | Content |
|---|---|
| Action | remove / merge / move / add deny / add allow |
| Rule | The exact permission string |
| From | Source file (or "new") |
| To | Target file (or "deleted") |
| Reason | One-line explanation |
| Risk | low / medium / high |

Group changes by risk level (low first, high last) for easy scanning.

### C) Verification Steps (after applying + new session)

Verify in a **new Claude Code session**:

```bash
# 1. Positive tests — these should succeed
git status              # if git allowed
jq --version            # representative low-risk tool
npm test                # if project allows

# 2. Negative tests — these should be DENIED
sudo ls                 # must be denied
rm -rf /tmp/test-dir    # must be denied
curl http://example.com # denied unless explicitly allowed
env                     # denied if env leak restriction chosen
```

> **Both positive and negative verification are required.** A permission system is only correct if allowed commands succeed AND denied commands are rejected.

---

## Definition of Done

- [ ] Permissions reduced from one-off noise to a small, meaningful set
- [ ] High-risk operations are denied or narrowly exceptioned with approval
- [ ] Project-specific rules live in project shared; local remains minimal
- [ ] No environment-specific absolute paths in shared settings
- [ ] Proposed changes are easy to review (clear plan + reasons + risks)
- [ ] Both positive and negative verification pass in a fresh session
```