---
name: commit-plan-only
description: >-
  Recommends how to split current working-tree / Changes into reviewable git
  commits (grouping, order, messages, git add paths). Use when the user asks
  how to split commits, commit 쪼개기, commit plan, commit 추천, how to commit
  Changes, or similar — recommendation only. Never stages or commits.
---

# Commit plan only (recommend only)

**Never create commits, stage files, or run `git commit` / `git add` while using this skill.**  
Output a plan the user can apply themselves (or paste into a later explicit “커밋해줘” request outside this skill).

Unlike skills that auto-stage and commit, this one is **plan-only** so the user keeps control of attribution, staging order, and review.

## When to use

User wants advice on splitting or ordering commits for current Changes — not to have the agent commit.

## Workflow

1. Prefer the workspace git root. Mention other dirty sibling repos only if already in context.
2. Gather (read-only), in parallel:
   - `git status -sb`
   - `git diff --stat HEAD`
   - `git diff --name-only HEAD`
   - `git log -8 --oneline` (match message style)
3. Cluster by concern from paths + light diff skims. Do not over-read hunks.
4. Emit the plan. Stop. Do not offer to execute commits as part of this skill’s default ending.

## Split rules

- One concern per commit; message should explain **why**.
- Prefer vertical slices (one feature’s API+UI together) when tightly coupled.
- Keep migration + schema + OpenAPI + the feature that needs them together or in clear adjacent order.
- Separate: features / fixes / pure refactors / docs-only / chore (Docker, tooling).
- Order for bisect/revert: foundations → feature → polish/docs.
- Multi-repo: plan **per repo**; note dependency order (e.g. module → host → BFF).
- Flag secrets (`.env`, credentials) — never recommend committing them.
- Match existing `git log` style (prefix, language, tone).

## Shell-safe `git add` (required)

Plans are copy-pasted into **zsh**. Always quote paths so globs do not expand.

- **Always prefix each Stage block with `cd` to that commit’s repo root** (absolute or `~/…`). Users often stay in another workspace and paste paths from a sibling repo → `pathspec did not match` / `could not open directory`.
- Quote **every** path in `git add` with single quotes: `'path/with/[brackets]/file.ts'`
- Especially required for `[`, `]`, `?`, `*`, spaces, or `!`
- Prefer one quoted path per line under a `git add \` continuation
- Do **not** emit bare `git add apps/.../[memberId]/...` — zsh fails with `no matches found`
- Paths are **relative to that repo root after `cd`**, never assume the Cursor workspace root

**Good:**
```bash
cd ~/Desktop/example-module
git add \
  'packages/example/lib/src/domain/relative_time.dart'

cd ~/Desktop/example-app
git add \
  'apps/web/src/app/api/v1/members/[memberId]/children/route.ts' \
  'apps/web/src/hooks/useActiveBaby.tsx'
```

**Bad:**
```bash
# still in wrong repo — sibling paths fail
git add 'packages/example/lib/src/domain/relative_time.dart'

git add apps/web/src/app/api/v1/members/[memberId]/children/route.ts
```

In the **Files:** bullet, backticks are fine for reading. In **Stage (you run):**, always use quoted paths for anything with special chars (or quote all paths for consistency).

## Output template

```markdown
## Commit plan — `<repo>`

### N. `<type>: <short why>`
- **Files:** `path1`, `path2`, …
- **Why:** one line
- **Stage (you run):**
\`\`\`bash
cd /absolute/or/~/path/to/<repo>
git add \\
  'path/with/[brackets]/a.ts' \\
  'path/without/special.tsx'
\`\`\`

### Order
1 → 2 → …

### Notes
- leftovers, ambiguous files, risks
```

## Hard rules

- No `git add`, `git commit`, `git push`, amend, or staging.
- No “이대로 커밋해 줄까요?” as the primary CTA — end with the plan only.
- If the user separately asks to commit later, that is outside this skill; follow normal commit rules then.
