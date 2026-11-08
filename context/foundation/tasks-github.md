---
project: fin-assist
system: github
created: 2026-11-08
updated: 2026-11-08
source: context/foundation/roadmap.md
---

# Task management: GitHub

The roadmap in `context/foundation/roadmap.md` is mirrored into GitHub Issues + a GitHub Project board.
`roadmap.md` stays the source of truth for scope and sequencing; GitHub tracks execution.

## Where

| What        | Location                                                                    |
| ----------- | --------------------------------------------------------------------------- |
| Repository  | `minikowsky/fin-assist` (issues enabled)                                    |
| Issues      | https://github.com/minikowsky/fin-assist/issues                             |
| Milestone   | `M-1: First categorized month` (milestone #1)                               |
| Board       | [fin-assist roadmap](https://github.com/users/minikowsky/projects/1) (user project #1, linked to the repo, private) |
| CLI         | `gh` — needs scopes `repo` + `project` (`gh auth refresh -h github.com -s project`) |

## Mapping: roadmap → GitHub

| Roadmap concept           | GitHub                                                                 |
| ------------------------- | ---------------------------------------------------------------------- |
| Milestone (M-N)           | GitHub milestone `M-N: <name>`; description = Intent + "Done when"     |
| Foundation (F-NN)         | Issue, label `foundation`                                              |
| Slice (S-NN)              | Issue, label `slice`                                                   |
| Open Roadmap Question     | Issue `[Q-N] …`, label `question`, same milestone                      |
| Status `ready`/`proposed` | Label `status: ready` / `status: proposed`                             |
| Stream (A–D)              | Label `stream: A-account`, `stream: B-import`, `stream: C-sets`, `stream: D-categories` |
| North star                | Label `north-star`                                                     |
| Prerequisites             | Native "blocked by" issue dependency + `#n` refs in the body           |
| Status `done`             | Issue closed (board Status = Done)                                     |
| Parked                    | Not migrated; stays in roadmap.md only                                 |

## Issue format

**Title:** `[<Roadmap ID>] <Suggested issue title from Backlog Handoff>`, e.g. `[S-02] Import bank CSV and review/edit every record`.

**Body:**

```markdown
**Outcome:** user can …

> ⭐ **North star** — … (north-star slice only)

| Field | Value |
|---|---|
| Roadmap ID | S-02 |
| Change ID | `csv-import-review` |
| PRD refs | US-01, FR-007, FR-008, FR-009 |
| Prerequisites | #1 (F-01), #3 (S-02) — or — |
| Parallel with | #1 (F-01), #2 (S-01) |
| Unlocks | … (foundations only) |
| Ready for `/10x-plan` | yes / no |

### Blockers
- [ ] …   (or —)

### Unknowns
- [ ] … — Owner: user · Blocks: no   (or —)

### Risk
…

### Next step
`/10x-plan <change-id>`  (or "After F-01, S-02 — then `/10x-plan <change-id>`")

---
Source: [roadmap.md › S-02](https://github.com/minikowsky/fin-assist/blob/master/context/foundation/roadmap.md#<heading-anchor>)
```

**Question body:** the question, `- **Owner:** …`, `- **Blocks:** …`, source link to `#open-roadmap-questions`.

## Board

- Built-in **Status** field: Todo / In Progress / Done.
- Items with `status: ready` start in **Todo**; `status: proposed` items have no status until their blockers close.
- IDs for automation (`gh project item-edit`):
  - Project node ID: `gh project view 1 --owner minikowsky --format json --jq .id`
  - Status field: `PVTSSF_lAHOApHW_c4BquzVzhpElEM`
  - Options: Todo `f75ad846`, In Progress `47fc9ee4`, Done `98236657`

## Current index (M-1)

| Roadmap ID | Issue | Blocked by        |
| ---------- | ----- | ----------------- |
| F-01       | #1    | —                 |
| S-01       | #2    | —                 |
| S-02       | #3    | —                 |
| S-03       | #4    | #1, #3            |
| S-04       | #5    | #1, #3            |
| S-05       | #6    | #5, #12           |
| S-06       | #7    | #1, #3            |
| S-07       | #8    | #4                |
| S-08       | #9    | #4                |
| S-09       | #10   | #4                |
| Q-1        | #11   | —                 |
| Q-2        | #12   | — (blocks #6)     |
| Q-3        | #13   | —                 |

The `Issue` column in roadmap.md's "At a glance" and "Backlog Handoff" tables mirrors this index.

## Workflow

1. **Pick work:** an open issue with `status: ready` and no open "blocked by" links.
2. **Start:** board Status → In Progress; run `/10x-new` / `/10x-plan <change-id>`.
3. **Ship:** reference the issue in the PR/commit (`Closes #n`); closing it moves the board item to Done.
4. **Unblock:** when an item's blockers are all closed, swap `status: proposed` → `status: ready` and set board Status to Todo.
5. **Keep in sync:** update the roadmap.md Status (`ready` → `in-progress` → `done`) when the issue changes state. Scope changes go into roadmap.md first, then into the issue.

## Adding items (new slices or the next milestone)

1. Add the item to roadmap.md (via `/10x-roadmap`).
2. Create the milestone if new: `gh api repos/minikowsky/fin-assist/milestones -f title="M-2: …" -f description="…"`.
3. Create the issue in the format above with `gh issue create -R minikowsky/fin-assist --title … --body-file … --milestone … --label …`.
4. Add dependencies: `gh api -X POST repos/minikowsky/fin-assist/issues/<n>/dependencies/blocked_by -F issue_id=<blocker's database id>` (database id: `gh api repos/minikowsky/fin-assist/issues/<blocker> --jq .id`).
5. Add to board: `gh project item-add 1 --owner minikowsky --url <issue url>`.
6. Write the issue number back into roadmap.md and the index above.
