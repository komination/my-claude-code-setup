---
name: permissions-optimizer
description: Optimize Claude Code permissions in settings files. Deduplicate and normalize allow/ask/deny rules, consolidate via safe wildcards, resolve scope conflicts, and produce a reviewable diff plan with minimal-risk defaults. Triggers: "optimize permissions", "permission optimizer", "clean permissions", "tidy settings.json", "permissions policy".
---

# Permissions Optimizer

A skill to optimize Claude Code permission rules with **least-privilege defaults**, clear scoping (user vs project vs local), and a review-friendly change plan.

This version explicitly supports **Ask rules** (`permissions.ask`) and follows the official evaluation order: **deny → ask → allow** (first match wins). :contentReference[oaicite:0]{index=0}

---

## Goals

- Reduce permission sprawl (duplicates, one-offs, noisy entries)
- Normalize patterns and consolidate where **provably safe** (see §Consolidation Safety Criteria)
- Use **Ask** for “always confirm” actions instead of blanket allow
- Resolve cross-scope conflicts and clarify exceptions
- Keep risky operations tight: **deny by default**, allow only with explicit approval
- Output a clear plan: **what changes**, **where**, and **why**

---

## Policy Baseline (recommended)

### 1) Keep *user/global permissions* empty by default
User scope (`~/.claude/settings.json`) should usually contain **no `permissions.allow|ask|deny` rules**. Put permissions in the project instead to avoid “permission drift” across unrelated repositories.

If you must have global rules, restrict them to **universal safety rails** only:
- Global `deny`: “never silently allow anywhere” primitives
- Global `ask`: “always confirm everywhere” operations

(Allow rules globally are strongly discouraged.)

### 2) Project shared is the main place for policy
`.claude/settings.json` is the canonical, reviewable, team-shared policy surface. :contentReference[oaicite:1]{index=1}

### 3) Project local stays near-empty
`.claude/settings.local.json` is for personal overrides and experimentation; it is not checked in, and Claude Code configures git to ignore it when created. :contentReference[oaicite:2]{index=2}

---

## Settings File Schema

Claude Code uses a hierarchical settings system with three main scopes and precedence:

1. Local (highest of these)  
2. Project  
3. User (lowest)  

More specific scopes override broader ones; e.g., a project deny can block a user allow. :contentReference[oaicite:3]{index=3}

Example structure:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run test *)",
      "Bash(git diff *)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Bash(sudo *)",
      "Read(./.env)",
      "Read(./secrets/**)"
    ]
  }
}
````

---

## Inputs (files to read)

Read these three files using absolute paths (expand `~` via `$HOME`):

```bash
cat "$HOME/.claude/settings.json"          # User (global)
cat .claude/settings.json                  # Project shared (repo root)
cat .claude/settings.local.json            # Project local (repo root)
```

### Notes

* Global and project files may be symlinked to the same target; detect with `readlink -f` and avoid double-editing.
* Handle missing files gracefully — treat as `{ "permissions": { "allow": [], "ask": [], "deny": [] } }`.
* Changes require a **new session** to take effect.

---

## Permission Rule Syntax (official)

Permission rules follow `Tool` or `Tool(specifier)`. ([Claude Code][1])

### Match all uses of a tool

Using just the tool name matches all uses (e.g., `Bash` matches all bash commands). `Bash(*)` is equivalent to `Bash`. ([Claude Code][1])

### Specifiers (fine-grained control)

Examples:

* `Bash(npm run build)` matches the exact command
* `Read(./.env)` matches reading `.env` in the current directory
* `WebFetch(domain:example.com)` matches fetches to that domain ([Claude Code][1])

### Wildcards for Bash

Bash rules support glob `*` anywhere in the command string. ([Claude Code][1])

> The space before `*` matters: `Bash(ls *)` matches `ls -la` but not `lsof`, while `Bash(ls*)` matches both.
> Legacy `:*` suffix syntax is equivalent to ` *` but is deprecated. ([Claude Code][1])

---

## Guardrails (non-negotiable)

* Do **not** auto-allow high-risk actions (destructive ops, privilege escalation, external exfiltration)
* Prefer **deny broader / allow narrower**
* Prefer **ask** over allow for anything you want to confirm every time (network, remotes, publishing)
* Keep `.claude/settings.local.json` **empty by default** (temporary exceptions only)
* Always propose changes as a **plan/diff** first; apply only after explicit user approval
* Write changes using Claude Code edit actions or output final JSON for copy/paste — never silently overwrite

---

## Workflow

### 1) Read & Parse

* Load and parse the three settings files (missing files become empty).
* Resolve `$HOME` to an absolute path; detect symlinks with `readlink -f`.
* Extract `allow` / `ask` / `deny` entries and record scope:

  * `user (global)`, `project(shared)`, `project(local)`

### 2) Intent / Threat Model (batch questions)

Before building the optimization plan, ask these in one batch:

1. **Network-capable bash commands** (`curl`, `wget`, etc.)

   * Default: **Ask** (or Deny) and prefer `WebFetch` policy instead
   * If needed: allow only very narrow patterns
2. **Git remote operations** (`git fetch/pull/push`)

   * Default: **Ask** each time
   * Options: allow in project only (rare), or keep ask
3. **Destructive operations** (`rm -rf`, `docker system prune`, `mkfs`, etc.)

   * Default: **Deny**
   * If any exceptions: user must specify exact safe commands
4. **Package managers** (`npm`, `pip`, `poetry`, etc.)

   * Default: project shared only
   * Typical split: `npm run *` allow; installs/upgrades ask

Carry the answers into Steps 4–6.

### 3) Inventory & Normalize

Create a normalized inventory per rule:

| Field   | Values                                               |
| ------- | ---------------------------------------------------- |
| action  | allow / ask / deny                                   |
| tool    | Bash / Read / Edit / WebFetch / MCP / etc.           |
| pattern | command/path/wildcard                                |
| scope   | user(global) / project(shared) / project(local)      |
| flags   | one-off, wildcardable, conflict, env-specific, risky |

#### Normalization rules (safe transforms only)

| Transform                      | Example                                            | Safe?                                               |
| ------------------------------ | -------------------------------------------------- | --------------------------------------------------- |
| Trim trailing whitespace       | `Bash(npm test )` → `Bash(npm test)`               | ✅ Always                                            |
| Remove exact duplicates        | Two identical rules in same file                   | ✅ Always                                            |
| Collapse internal multi-spaces | `Bash(npm  run  build)` → `Bash(npm run build)`    | ⚠️ Only if clearly accidental                       |
| Replace legacy `:*`            | `Bash(git diff:*)` → `Bash(git diff *)`            | ⚠️ Only with explicit plan note (deprecated syntax) |
| Merge different commands       | `git push` + `git push origin main` → `git push *` | ❌ Not without explicit approval                     |

> Principle: When in doubt, **do not normalize**. Redundant rules are cheaper than breaking a permission boundary.

### 4) Diagnose Issues

Detect and tag:

* **Exact duplicates** — same rule in the same file
* **Cross-scope duplicates** — same rule in user and project (project wins; user copy may be removable)
* **One-time permissions** — over-specific, unlikely to repeat
* **Wildcardable** — narrow rules fully covered by a safe wildcard (see §Consolidation Safety Criteria)
* **Scope conflicts** — project deny vs user allow, or contradictory rules within a file
* **Environment-specific paths** — absolute machine paths in shared project settings
* **Destructive operations** — delete/prune/format/recursive permission changes
* **Over-broad allow** — `Bash` / `Read` / `Edit` with no scoping
* **Ask/Allow mismatch** — rules that should be ask but are allow (per threat model), or ask that should be deny

### 5) Build an Optimization Plan (3 tiers)

#### Tier 1 — Safe automatic candidates (recommend by default)

* Remove exact duplicates (same file)
* Remove rules strictly subsumed by an existing wildcard **that is itself Tier 1**
* Remove cross-scope duplicates where the surviving rule is equal or tighter scope
* Trim clearly accidental trailing whitespace
* Flag deprecated `:*` and propose equivalent ` *` replacement (no behavior change intended)

#### Tier 2 — Needs batch user confirmation (Ask-first decisions)

* Network commands (prefer **Ask**; allow only narrow)
* Git remotes (`fetch/pull/push`) (default **Ask**)
* Package installs/upgrades (default **Ask**)
* Scope relocation (user ↔ project)
* Wildcard consolidation for safe families (e.g., `npm run *`)

#### Tier 3 — High-risk (default deny; confirm individually)

* Privilege escalation: `sudo`, `su`
* Remote access and file transfer: `ssh`, `scp`, `rsync`
* Destructive: `rm -rf`, `mkfs`, `dd`, broad prune operations
* Broad permission changes: `chmod -R`, `chown -R`
* Over-broad patterns: `Bash` (match all), `Read` (match all), etc.

### 6) Placement Rules (where each rule should live)

Decision flow:

1. High-risk primitives → **project deny** (or global deny only if you insist on universal rails)
2. Universal & low-risk defaults → **project shared allow**
3. “Confirm every time” operations → **project shared ask**
4. Temporary / machine-specific exceptions → **project local** (prefer ask over allow)

---

## Consolidation Safety Criteria

A set of narrow rules may be consolidated into a wildcard only when all of the following hold:

1. **Complete coverage**: Every command matched by the wildcard is already allowed/asked individually (no net expansion).
2. **Low/Medium risk only**: Never auto-consolidate into Tier 3 patterns.
3. **Same scope**: All source rules share the same scope, or the target scope is tighter.
4. **No deny weakening**: No deny rule is contradicted or effectively bypassed.

If any condition fails, keep the narrow rules and flag for manual review.

---

## Output Format (what to present)

### A) Summary

* Rule counts before/after (per file and total)
* How many removed / merged / moved / added
* Number of conflicts found
* High-risk items requiring explicit confirmation

### B) Proposed Changes (reviewable plan)

For each change:

| Field  | Content                                                |
| ------ | ------------------------------------------------------ |
| Action | remove / merge / move / add deny / add ask / add allow |
| Rule   | The exact permission string                            |
| From   | Source file (or "new")                                 |
| To     | Target file (or "deleted")                             |
| Reason | One-line explanation                                   |
| Risk   | low / medium / high                                    |

Group changes by risk level (low first, high last).

### C) Verification Steps (after applying + new session)

In a **new Claude Code session**:

```bash
# 1) Positive tests — should succeed
git status
git diff
npm test            # if project allows

# 2) Negative tests — should be DENIED or ASKed (as configured)
sudo ls                 # should be denied
rm -rf /tmp/test-dir    # should be denied
git push                # should ask (if configured as ask)
curl https://example.com # should ask or be denied (per policy)
```

> Both positive and negative verification are required.

---

## Definition of Done

* [ ] Permissions reduced to a small, meaningful set
* [ ] High-risk operations denied or narrowly exceptioned with approval
* [ ] Project shared contains the stable policy; local remains minimal
* [ ] No environment-specific absolute paths in shared settings
* [ ] Proposed changes are easy to review (plan + reasons + risks)
* [ ] Both positive and negative verification pass in a fresh session

```
::contentReference[oaicite:9]{index=9}
```

[1]: https://code.claude.com/docs/en/permissions "Configure permissions - Claude Code Docs"
