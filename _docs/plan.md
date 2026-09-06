# Shared Household Chores Tool — MVP Scope

## Problem

Households with multiple people struggle with chores in two ways at once:
- Tasks get neglected (no reminders, no ownership).
- The split feels unfair — one person ends up doing more, and effort isn't naturally equal because people have different capacity/free time.

## Users

- 1–10 people living in the same household.
- Chore split is expected to be **naturally uneven** (different schedules/capacities), not equal.

## Core Mechanic

1. Every task has a **point value** representing effort/unpleasantness (not duration). Points come with sensible defaults ("common knowledge") and are editable.
2. Each person sets a **daily free-time capacity** (a single number, e.g. minutes/day) — self-reported, editable at will initially, but see fairness mitigation below.
3. **Claim-first**: tasks sit in an open pool; people claim what they're willing to do, first-come-first-served.
4. At a daily cutoff, **unclaimed tasks auto-assign** to whoever has the lowest fairness ratio:
   - `ratio = points assigned ÷ stated capacity`
   - Ratio is **rolling/cumulative** — it never resets to zero (no weekly reset).
   - Whoever has the most "slack" relative to their own capacity gets the next unclaimed task.
5. **Missed/incomplete tasks roll over** to the next day and count as "debt" — they stay owed until completed (in addition to normal daily assignment).

## Points Governance

- Consensus on point values happens socially, outside the tool.
- Only **admins** can change a task's point value.
- Every point-value change is **logged and visible** (who changed what, when).

## Household Setup

- People **join a household via an invite code**.
- No multi-tenant complexity beyond that — a simple household membership model.

## Platform

- Simple **web app**. No native mobile app in MVP.

## MVP Feature List (In Scope)

- Household creation + join-by-invite-code
- Task list with default point values (admin-editable, changes logged with who/when)
- Recurring tasks (daily/weekly)
- Per-person daily free-time capacity, self-reported
- **Capacity visible to all household members** (transparency, not private)
- **Capacity changes logged** (who changed it, when) — same audit pattern as point values
- Claim-first task pool
- Automatic daily assignment of unclaimed tasks based on lowest points-used ÷ capacity ratio (rolling, no reset)
- Rollover + "debt" tracking for missed/incomplete tasks
- Mark-as-done tracking
- Simple dashboard showing each person's points, capacity, and ratio
- Basic reminder for unclaimed/overdue tasks

## Out of Scope (v2+)

- Trading/swapping claimed tasks between people
- Gamification (streaks, badges, leaderboards)
- Vacation/exception handling (pausing capacity temporarily)
- Multi-week capacity patterns or historical trend analysis
- Native mobile app
- Rich notifications beyond a basic reminder

## Riskiest Assumption

A single self-reported number ("daily free-time capacity") may not be a good enough proxy for real-world availability/fairness. People might find it awkward to self-report accurately, or might lowball it to dodge chores.

**Mitigation folded into MVP**: rather than policing this algorithmically, apply the same transparency philosophy already used for point-value governance:
1. Capacity is **visible to the whole household**, not private — social visibility discourages obvious lowballing.
2. Capacity changes are **logged with a timestamp** (same audit-trail pattern as point-value edits), so a pattern like "always lowers it right before assignment" becomes visible over time.

Disputes are expected to be resolved socially by the household, same as point-value disputes — the tool's job is transparency, not enforcement.

## Suggested Next Step

Turn this into a written spec/PRD: goals, user stories, data model, and acceptance criteria — ready to build from.
