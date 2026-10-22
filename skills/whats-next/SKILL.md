---
name: whats-next
description: Context-aware workflow compass for compound engineering. Use when the user asks "what should I be doing next", "what's next", "where was I", "what now", "remind me of the workflow", or wants to know which ce-* command fits the current situation. Inspects git, PRs, plans, brainstorms, STRATEGY.md milestones, GitHub issues by priority, hand-off notes, and session context, then recommends the next workflow step with reasoning.
---

# What's Next — Workflow Compass

Answer one question: **given the actual state of this project right now, what is the single most valuable next step, and which command starts it?** This skill is read-only — gather evidence, diagnose, recommend. Never modify anything.

## Where "what's next" lives

State is never stored in one "next tasks" file; it is recomputed from these systems of record every time. The skill must look at all of them:

| Source | Answers |
|---|---|
| Git working tree, branches, log | What is uncommitted, what shipped |
| Open PRs, checks, review threads | What is mid-review |
| `STRATEGY.md` (whole file) | Direction, tracks, **dated milestones**, not-working-on triggers |
| `docs/brainstorms/*-requirements.md` | What a feature must do |
| `docs/plans/*-plan.md` | How it will be built (units U1..Un) |
| `docs/residual-review-findings/` | Deferred review findings |
| GitHub issues, by priority label | Bugs, debt, product backlog, user-run checklists |
| `docs/solutions/`, `CONCEPTS.md` | Whether recent fixes were compounded |
| Hand-off notes (`~/.claude/projects/<slug>/memory/`) | Authoritative "where was I" written at session end |
| `.context/session-*.md` | Session transcript summary (secondary, large) |

Not inputs: decision records, archived backlogs.

The hand-off notes and session summaries are my own convention: a short note written deliberately at the end of each session. Point those two rows, and the matching lines in Step 1, at wherever you keep yours, or delete them.

## Step 1: Gather evidence (fast, parallel, tolerate failures)

Run these checks from the **project root**; skip silently anything that fails or doesn't apply (no gh, no STRATEGY.md, etc.):

```bash
git branch --show-current && git status --porcelain | head -15 && git log --oneline -7
git branch --no-merged 2>/dev/null | head          # local branches never merged (stranded work?)
gh pr status 2>/dev/null                          # open PR for this branch? checks? review comments?
ls -t docs/plans/*-plan.md 2>/dev/null | head -3   # newest feature plans
ls -t docs/brainstorms/*.md 2>/dev/null | head -3  # newest brainstorms
ls docs/residual-review-findings/ 2>/dev/null      # unaddressed review residuals
[ -f STRATEGY.md ] && cat STRATEGY.md              # WHOLE file: tracks, milestones, not-working-on
grep -n -A6 '^## Milestones' STRATEGY.md 2>/dev/null   # dated milestones — never skip this line
gh issue list --state open --label p1 2>/dev/null  # p1 issues, any age
gh issue list --state open --limit 30 --json number,title,labels \
  --jq '.[] | "\(.number)\t\(.labels|map(.name)|join(","))\t\(.title)"' 2>/dev/null
ls -t ~/.claude/projects/$(pwd | sed 's|/|-|g')/memory/project_session_handoff_*.md 2>/dev/null | head -1
ls -t .context/session-*.md 2>/dev/null | head -1  # most recent session summary
find docs/solutions -name '*.md' -exec ls -t {} + 2>/dev/null | head -1   # newest captured learning
```

Then read, in this order, stopping when "where was I" is answered:

1. **The newest hand-off note** if present. It is written deliberately at session end and is the authoritative resume point. (If it is already loaded in context, do not re-read it.)
2. **The newest `.context/session-*.md`** only if no hand-off exists or it leaves a gap. These files are large; read the Key Decisions block and the tail, not the whole narrative.
3. **Every p1 issue body.** p1 issues often carry a checklist of user-run steps an agent cannot do; those are recommendations in their own right.
4. **The newest plan's unit headings** (`grep -n '^### U[0-9]' <plan>`) compared against `git log --oneline <plan-commit>..HEAD` to estimate which units shipped.

From STRATEGY.md, extract every dated milestone and compute days remaining from today's date.

## Step 2: Diagnose

Work down this priority order. Collect every rule that fires, then pick the top 1 recommendation plus up to 2 alternatives. Work already in flight always outranks starting something new.

If a dated milestone has already passed, lead with it whatever else fires, and say which rule below is the quickest way to close it.

| Priority | Signal | Recommendation |
|---|---|---|
| 1 | Uncommitted changes on **any** branch, including main | Finish the current unit; then `/ce-commit` (or continue `/ce-work`). On main, commit or branch before anything else |
| 2 | Local branch not merged and no PR | Decide: open a PR, merge, or delete — stranded work goes stale |
| 3 | Open PR with failing checks | Fix CI — read the failure logs, repair the real issue |
| 4 | Open PR with unresolved review comments | `/ce-resolve-pr-feedback` |
| 5 | File in `docs/residual-review-findings/` for a current/recent branch | Address the findings, or convert them into issues |
| 6 | Newest plan has units not yet reflected in git history | `/ce-work <plan path>` to resume, naming the first unshipped unit |
| 7 | Brainstorm/requirements doc newer than the newest plan | `/ce-plan <brainstorm path>` |
| 8 | Open **p1 issue** not covered by the newest plan, or a p1 with unchecked user-run steps | Surface it by number; user-run steps become an "also worth considering" item the user must do themselves |
| 9 | STRATEGY.md milestone within 30 days, or a fallback/checkpoint date already passed | State the milestone, days remaining, and whether the current plan reaches it |
| 10 | Hand-off note or session summary mentions an unsolved problem | Resume it — offer `/ce-debug` if it's a bug |
| 11 | Recent commits show a hard-won fix, and `docs/solutions/` has nothing newer | `/ce-compound` — capture it before it evaporates |
| 12 | No `STRATEGY.md` | `/ce-strategy` to anchor direction |
| 13 | Nothing in flight | Read `STRATEGY.md` tracks → `/ce-brainstorm` or `/ce-ideate` the next bet; prefer the track that serves the nearest milestone |

Periodic nudges — append at most ONE, and only when triggered: plugin possibly stale (`/ce-update`), `docs/solutions/` hasn't grown in weeks despite gnarly commits (`/ce-compound`), solutions pile looks stale or contradictory (`/ce-compound-refresh`), a STRATEGY.md not-working-on revisit trigger now appears to be met.

## Step 3: Recommend

Keep the output under ~15 lines:

```
## What's next

**→ /ce-work docs/plans/2026-06-10-001-feat-export-plan.md**
Units U3–U5 aren't in git history yet; U1–U2 shipped in a4f21c9 and 8be0d11.

Also worth considering:
- Issue #42 has a checklist of 4 steps only you can do (account, API key, DNS)
- /ce-compound — last week's export fix isn't captured in docs/solutions/

Milestone: v1 launch due 2026-07-01, 21 days out; checkpoint 2026-06-20.

Want me to start the first one?
```

Every recommendation must cite the observed evidence that triggered it (file, count, commit, PR number, issue number, date) — no generic advice. If the user says yes, invoke the recommended skill/command directly.

## Workflow reference (for context)

Strategy → requirements → plan → work → review → ship → compound:
`/ce-strategy` → `/ce-brainstorm` → `/ce-plan` → `/ce-work` → `/ce-code-review` → `/ce-commit-push-pr` → `/ce-compound`. Bug intake: `/ce-debug`.
