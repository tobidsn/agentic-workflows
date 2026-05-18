---
name: nexus-task
description: Bridge the fixed Nexus 2026 Lark task list to the feature workflow — list my open Core "To Do" tasks (sorted by Priority), pick one, spawn a sub-agent that runs `/workflow start <slug>`, and auto-close the Lark task when the feature reaches PR review.
---

# Nexus Task

Connects the Lark task list **Nexus 2026**
(`https://applink.larksuite.com/client/todo/task_list?guid=d9fb25b0-7023-4f42-8da3-06232b531f4d`)
to the `/workflow` feature pipeline in this repo.

**Fixed tasklist GUID:** `d9fb25b0-7023-4f42-8da3-06232b531f4d`

Each Lark todo becomes a candidate feature. The user picks one, a sub-agent
spawns and runs `/workflow start <slug>`, and the Lark todo is closed once the
feature enters the `pr_review` phase.

## Default view

Running `/nexus-task` shows tasks matching **all** of:

- `Project` includes `Core`
- `Owner` = current authenticated Lark user (resolved at runtime via
  `lark-cli auth status` → `.userOpenId`)
- `Status` ∈ {`To Do`, `Reopened`}

Results are **sorted by Priority** (High → Medium → Low → unset), then by Lark
`task_id` for stability.

## Flags (override defaults)

| Flag | Effect |
|---|---|
| `--all` | Bypass the Owner filter only. Keeps Project + Status filters. Use to browse Core/To Do tasks across the whole team. |
| `--status <label[,label…]>` | Override Status filter. Pass `any` to disable. |
| `--project <label[,label…]>` | Override Project filter. Pass `any` to disable. |
| `--owner <open_id\|"name">` | Filter by a specific user instead of me. |

## Prerequisites

- `lark-cli` installed and logged in (`lark-cli auth login`). `lark-cli auth
  status` must return a `userOpenId` — that is the value used for `Owner = me`.
- Scopes required on the lark-cli app: `task:task:read`, `task:task:write`,
  `task:tasklist:read`, `contact` (for resolving assignee names).
- `jq` on PATH.
- `options.json` in this skill's directory — holds the option-ID → label maps
  for Status, Project, Priority, plus the `user_map` cache. Labels are
  maintained manually because the lark-cli app lacks
  `task:custom_field:read`.
- `.workflow-state.json` in the repo root (created by `/workflow start` on
  first use).

## Commands

### `/nexus-task` (alias: `/nexus-task list`)
List filtered todos sorted by Priority.

**Steps:**

1. Load `options.json`. Capture: `default_filter`, `status.labels`,
   `project.labels`, `priority.labels`, `priority.sort_order`, `user_map`.
2. Parse CLI flags. Build the effective filter:
   - `status_labels` = `--status` value (split by comma), or
     `default_filter.status`.
   - `project_labels` = `--project` value, or `default_filter.project`.
   - `owner_id`:
     - If `--all`: `null` (no owner filter).
     - Else if `--owner <value>`: resolve `value` — UUID-like → use directly;
       human name → search `user_map` for matching name and use its open_id;
       on miss, run `lark-cli contact +search-user --query "<name>"` and pick
       the top hit.
     - Else (default): run `lark-cli auth status` and read `.userOpenId`. If
       empty, abort: "Not authenticated. Run `lark-cli auth login`."
3. Resolve label sets to ID sets:
   - `keep_status_ids` = `{ id | status.labels[id] ∈ status_labels }`.
     If a requested label is missing from `labels`, surface a warning and
     skip it. If all requested labels miss, abort with the list of known
     labels.
   - `keep_project_ids` = same logic for `project.labels`.
4. Fetch the tasklist (basic fields only — `guid`, `summary`, `members`):
   ```bash
   lark-cli task tasklists tasks \
     --params '{"tasklist_guid":"d9fb25b0-7023-4f42-8da3-06232b531f4d","completed":false,"page_size":50}' \
     --page-all --format json
   ```
5. Apply the cheap owner pre-filter before fetching task details: if
   `owner_id` is set, drop tasks whose `members` array does not contain a
   member with `id == owner_id` (any role — assignee or follower). This
   avoids ~90 detail fetches when only a few tasks involve me. A strict
   assignee-only re-check runs in step 8.
6. For the remaining tasks, fetch full detail in batches of ≤8 in parallel
   (higher concurrency tends to return empty payloads under rate limit):
   ```bash
   lark-cli task tasks get --params '{"task_guid":"<guid>"}' --format json
   ```
   Retry empty responses sequentially with ~300 ms delay.
7. From each detail, extract `status_id`, `project_ids[]`, `priority_id`,
   `task_type_id`, `description`, `task_id`, and `assignee_ids` (members
   with `role == "assignee"`). Use `first(.custom_fields[]? |
   select(.name=="X") | ...) // null` when extracting single-select fields
   — a task can legitimately have no Status/Priority/Task Type assigned,
   and a non-guarded jq expression will drop the whole row.
8. Apply remaining filters:
   - `status_id ∈ keep_status_ids`
   - `project_ids ∩ keep_project_ids ≠ ∅`
   - If `owner_id` is set: `owner_id ∈ assignee_ids` (strict — a task where
     I'm only a follower does **not** match "owner=me").
9. Sort the result by `priority.sort_order.indexOf(priority_id)` ascending
   (unknown priority sorts last); break ties by `task_id` ascending.
10. Resolve assignee open_ids to names via `user_map`. For misses, call
    `lark-cli contact +get-user --user-id <id>` and write the result back to
    `options.json.user_map` so the next run is cache-only.
11. Cross-reference `.workflow-state.json`. If a feature has
    `lark_task_id == <task.guid>`, annotate "linked → <slug> (<phase>)".
12. Render the table — columns **#, ID, Title, Type, Status, Owner,
    Priority, Description**. The Type column is resolved via
    `options.json.task_type.labels`; unknown IDs show as `?`. Truncate
    description to ~40 chars; use `/nexus-task show <#>` for the full body.

    ```
    Nexus 2026 · Project: Core · Status: To Do, Reopened · Owner: Mohamad Tobiin
    Matched: <N> tasks (sorted by Priority)

    │ # │ ID      │ Title                              │ Type        │ Status   │ Owner         │ Pri  │ Description
    │ ─ │ ─────── │ ────────────────────────────────── │ ─────────── │ ──────── │ ───────────── │ ──── │ ─────────────────
    │ 1 │ t129941 │ External Voucher                   │ Improvement │ Reopened │ Ilham Shiddiq │ High │ FSD: https://…
    │ 2 │ t146478 │ Antiloyalty Template - Milestone   │ Feature     │ To Do    │ Fatan Aminul… │ High │ …

    Pick a todo: /nexus-task start <#>
    ```

13. On zero matches: print the empty header, then "No tasks match. Try
    `/nexus-task list --all` to see everyone's Core/To-Do tasks."

### `/nexus-task show <#>`
Print full detail of one todo from the most recent list — full description,
all assignees + followers, due date, source URL.

### `/nexus-task start <#>`
Spawn a sub-agent that runs `/workflow start` for the selected todo.
Auto-routes between full and small track based on the Lark **Task Type**.

**Steps:**

1. Re-run the list query to get fresh task data. `<#>` is 1-indexed against
   the last printed table.
2. Resolve `<#>` → `{task_guid, summary, description, task_type_id,
   task_type_label}` (resolve label via `options.json.task_type.labels`).
   - If already linked in `.workflow-state.json`: ask "Already linked to
     `<slug>` (`<phase>`). Switch with `/workflow switch <slug>`?" and stop.
3. Derive a feature slug from `summary`: lowercase, non-alphanumeric → `-`,
   collapse repeats, trim, truncate to 40 chars on a word boundary. If the
   slug collides with another unrelated feature, append `-2`, `-3`, … until
   unique.
4. Re-use the full task detail already fetched in step 1 of this command (no
   second API call needed — same command run, no persistent task cache).
5. **Route by Task Type:**
   - `{Feature, User Story}` → **full track** (existing /workflow flow).
   - `{Bug, Improvement, Task, Golang Refactor}` → **small track**
     (single-pass implementation, see `/workflow` skill → "Small track").
   - Unknown / unmapped Task Type → default to **full track**; print a
     warning naming the unrecognised label so the user can adjust
     `options.json` or override with `--type=small`.
6. Spawn a sub-agent via the Agent tool, `subagent_type: "general-purpose"`,
   `run_in_background: true`:
   - **description:** `"Start workflow for Lark todo: <slug>"`
   - **prompt:** include
     - full Lark todo summary + description
     - the slug, Lark `task_guid`, tasklist guid, and resolved
       `task_type_label`
     - instruction: ``"Run `/workflow start <slug> --type=<full|small>
       --task-type \"<label>\" --lark-task <guid>`. Follow that skill's
       phase machine; do not skip phases. When the feature reaches
       `pr_review` (full) or `small_pr_review` (small), stop and report
       back."``
7. After the Agent dispatches (background — returns immediately), reconcile
   `.workflow-state.json`:
   ```json
   "features": {
     "<slug>": {
       "lark_task_id": "<task_guid>",
       "lark_tasklist_guid": "d9fb25b0-7023-4f42-8da3-06232b531f4d",
       "lark_task_type": "<label>",
       "track": "<full|small>"
     }
   }
   ```
8. Report: "Spawned background agent for `<slug>` (Lark todo: <summary>,
   Type: <label>, Track: <full|small>). Track with `/workflow status`. You
   will be notified when the agent finishes."

### `/nexus-task sync`
Reconcile workflow state with Lark. Close any Lark todos whose features have
reached a "PR review" phase — `pr_review` (full track), `small_pr_review`
(small track), or `completion`.

**Steps:**

1. Read `.workflow-state.json`. For each feature: skip if no `lark_task_id`,
   skip if `phase` not in `{pr_review, small_pr_review, completion}`, skip
   if `lark_completed_at` already set.
2. For each remaining: `lark-cli task +complete --task-id <task_id>`.
3. On success, write `lark_completed_at` (ISO 8601). On failure, surface the
   error verbatim and leave the timestamp empty for retry.
4. Report a table of closed todos and a count of no-ops.

### `/nexus-task done <slug>`
Manually close the Lark todo linked to `<slug>`. Skips phase check.

### `/nexus-task unlink <slug>`
Clear `lark_task_id`, `lark_tasklist_guid`, `lark_completed_at` from
`features.<slug>` without closing the Lark todo. Use for mis-linked features.

## Refreshing option labels

Lark Task v2 does not expose custom-field option labels to read-only callers
without `task:custom_field:read`. The lark-cli app does not have that scope
registered. Labels live in `options.json` and are updated manually.

**Discover option IDs in a task:**
```bash
lark-cli task tasks get --params '{"task_guid":"<guid>"}' --format json \
  | jq '.data.task.custom_fields'
```

**Bulk-enumerate option IDs across the open list:**
```bash
lark-cli task tasklists tasks \
  --params '{"tasklist_guid":"d9fb25b0-7023-4f42-8da3-06232b531f4d","completed":false}' \
  --page-all --format json \
  | jq -r '.data.items[].guid' \
  | xargs -n1 -P8 -I{} lark-cli task tasks get --params '{"task_guid":"{}"}' --format json \
  | jq -s '[.[] | .data.task.custom_fields[]?]
           | group_by(.guid) | map({field: .[0].name, field_guid: .[0].guid,
             options: ([.[].single_select_value, .[].multi_select_value[]?]
                       | unique | map(select(. != null and . != "")))})'
```

Open one task per unknown option ID in the Lark UI to see the label, then
edit `options.json`.

## Notes

- **No task cache:** the skill always fetches tasklist + per-task detail
  fresh on every `/nexus-task` run. If a task's assignee, status, or
  priority changes in Lark, the next `/nexus-task` reflects it immediately.
  The only persistent cache is `options.json.user_map` (assignee open_id →
  display name), which is safe because names rarely change.
- **Empty default:** if you are not an assignee on any task matching
  Project=Core + Status∈{To Do, Reopened}, the default view returns zero.
  Use `--all` to browse, or `--owner "<name>"` to view someone else's.
- **Auto-close on phase transition:** this skill does not modify `/workflow
  next`. To get hands-free closing, run `/nexus-task sync` after transitions,
  or wrap in `/loop 10m /nexus-task sync`.
- **Slug collisions:** derived slugs are deterministic from the summary;
  re-running `start` on the same todo lands on the same slug. The
  already-linked check prevents duplicate features.
- **Tasklist scope:** the tasklist guid is hardcoded. To support multiple
  tasklists later, accept `--tasklist <guid|url>` and parse the applink URL.
- **Priority sort order:** defined in `options.json.priority.sort_order`.
  Labels for the priority option IDs may still be `TODO-confirm` until you
  open a task in Lark — sorting will still work as long as `sort_order` is
  correct.
