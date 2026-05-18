---
name: workflow
description: Manage feature workflow state — status, transitions, context switching
---

# Workflow Manager

Manages the feature development workflow state machine.

## Commands

### `/workflow status`
Show the current workflow state for the active feature.

**Steps:**
1. Read `.workflow-state.json`
2. Display:
   - Active feature: `{active_feature}`
   - Current phase: `{phase}` ({phase_name})
   - Phase entered: `{phase_entered}`
   - Worker: `{worker}`
   - Wireframes: list with approval status
   - Branch: `{branch}` or "not created"
   - Service PRDs: list with status
   - PRs: list `{pr_urls}` entries (service → URL) or "not submitted"

### `/workflow list`
Show ALL active features with their phases.

**Steps:**
1. Read `.workflow-state.json`
2. For each feature in `features{}`:
   - Show slug, phase, phase_entered, worker
   - Mark active feature with `→`

### `/workflow start <slug>`
Start a new workflow. **Auto-routes by Task Type** between two tracks:

- **Feature** / **User Story** → full track (PRD → wireframe → split → impl → PR)
- **Bug** / **Improvement** / **Task** / **Golang Refactor** → small track
  (impl → PR only, no PRD/wireframe/split)

**Flags:**
- `--type=feature` — force full track (default if no Lark task is linked)
- `--type=small` — force small track
- `--task-type "<Lark task type>"` — pass the Lark Task Type label (e.g.
  "Bug", "Improvement") and let the router decide. Used by
  `/nexus-task start` to pipe Lark metadata in.
- `--lark-task <guid>` — link the feature to a Lark task. Stored in
  `.workflow-state.json.features.<slug>.lark_task_id`.

**Steps:**
1. **Check onboarding state first.** Read `.onboard-state.json`:
   - If file doesn't exist and no service directories → BLOCK: "Run `/onboard` first."
   - If `status` is `"in_progress"`, check critical steps:
     - `steps.env_files` must be `"completed"` (all services have real .env)
     - `steps.db_setup` must be `"completed"` (if API service is selected)
     - `steps.clone_repos` must be `"completed"`
     - `steps.install_deps` must be `"completed"`
   - If ANY critical step is missing → BLOCK and list what's needed:
     ```
     Cannot start workflow. Onboarding is incomplete:
       - api/.env is missing — contact arhen
       - Database not restored — contact arhen for dev dump
     Run `/onboard resume` to complete setup.
     ```
   - If `status` is `"completed"` → proceed
2. Read `.workflow-state.json` (create if doesn't exist)
3. Check that `<slug>` doesn't already exist in features
4. **Resolve track** (full vs small):
   - If `--type` is passed: use it directly.
   - Else if `--task-type` is passed: route by label —
     `{Bug, Improvement, Task, Golang Refactor}` → `small`,
     `{Feature, User Story}` → `full`, anything else → `full` (safe default).
   - Else (no hints): default to `full`.
5. **If an active feature already exists AND JJ is available** (`command -v jj`):
   - Suggest parallel workspace as an option:
     > "You have an active feature (`<active-slug>` in `<phase>`). You can:"
     > - **Continue here** with context-switching (`/workflow switch` between features)
     > - **Create a parallel workspace** with `/jj-workflow new <slug>` to work on `<slug>` in a separate directory
   - If user chooses parallel workspace → tell them to run `/jj-workflow new <slug>` and stop
   - If user chooses to continue here → proceed with step 6
6. Create feature entry:
   - **Full track:** `phase = "prd_creation"`, `track = "full"`.
   - **Small track:** `phase = "small_implementation"`, `track = "small"`,
     `branch = "fix/<worker>/vc-<slug>"` (or `task/<worker>/vc-<slug>` for
     Task type, `golang/<worker>/vc-<slug>` for Golang Refactor).
   - In both cases: store `lark_task_id` and `lark_task_type` if provided.
7. Set `active_feature` to `<slug>`
8. Set `phase_entered` to current timestamp
9. Ask user for worker name if not set
10. Save state
11. Inform user:
    - **Full track:** "Feature `<slug>` started. Track: full. Phase:
      prd_creation. Delegating to frndos-prd."
    - **Small track:** "Feature `<slug>` started. Track: small. Phase:
      small_implementation. Branch: `<branch>`. Delegating to a sub-agent."

## Small track

A single-phase fast lane for Bug / Improvement / Task / Golang Refactor
tickets. Skips PRD, wireframe, prd_splitting, and multi-service orchestration.

### Phase: `small_implementation`

Driven by a background sub-agent (Agent tool, `subagent_type:
"general-purpose"`). The agent operates in the **current working directory**
(no auto-cd into another repo). Caller is expected to be in the target
service repo before invoking.

**Sub-agent prompt** (constructed by `/workflow start ... --type=small`):
- Full Lark task summary + description (if linked).
- Slug, branch name, target repo path (cwd at invocation).
- Instructions:
  1. `git checkout -b <branch>` from current HEAD (assumed to be the default
     branch — verify with `git rev-parse --abbrev-ref HEAD` and warn if not).
  2. Read the Lark task content; identify the change scope from the
     description.
  3. Make code changes. Keep commits scoped — one logical unit per commit,
     using the repo's commit-message style (`git log -10 --oneline`).
  4. Push branch: `git push -u origin <branch>`.
  5. Construct PR URL via `gh pr create --web --draft --fill` (don't submit,
     just print the URL the command opens), or fall back to the GitHub
     compare URL `https://github.com/<org>/<repo>/pull/new/<branch>` parsed
     from `git remote get-url origin`.
  6. For each commit on the new branch (`git log <default>..<branch>
     --reverse --format="%s"`), call:
     ```bash
     lark-cli task subtasks create \
       --params '{"task_guid":"<parent_lark_guid>"}' \
       --data '{"summary":"<commit-subject>"}'
     ```
     so the Lark task gets one subtask per commit. Skip if no
     `lark_task_id`.
  7. Update `.workflow-state.json.features.<slug>`:
     ```json
     { "phase": "small_pr_review",
       "branch": "<branch>",
       "pr_url": "<url>",
       "pr_urls": { "<service>": "<url>" },
       "commit_subjects": [ "<subject1>", "<subject2>" ] }
     ```
  8. Report back: branch name, PR URL, list of created subtasks.

### Phase: `small_pr_review`

Terminal phase for the small track. Same semantics as `pr_review` on the
full track for the purposes of `/nexus-task sync` — when this phase is
reached, the linked Lark task is eligible for auto-close.

### Gates

The small track has no manual gate transitions — `small_implementation` →
`small_pr_review` is automatic on agent completion. The only manual gate is
human PR review/merge (outside the skill).

### `/workflow next`
Transition to the next phase (if gate conditions are met).

**Steps:**
1. Read `.workflow-state.json`
2. Get current phase for active feature
3. Look up gate conditions from `.agentic-workflows/workflow/gates.json`
4. Check EACH condition:
   - If all pass → transition to next phase, update `phase` and `phase_entered`
   - If any fail → report which conditions failed and what's needed
5. Save state
6. Inform user of new phase and which agent will handle it

### `/workflow switch <slug>`
Switch active feature context.

**Steps:**
1. Read `.workflow-state.json`
2. Verify `<slug>` exists in features
3. Save any pending state for current feature
4. Set `active_feature` to `<slug>`
5. Load target feature's phase context
6. If on a different git branch, inform user: "Feature `<slug>` is on branch `{branch}`. Switch with: `git checkout {branch}`"
7. Save state

### `/workflow list-all`
Discover ALL features across the team — not just local ones. Scans committed artifacts, git branches, and JJ workspaces.

**Steps:**
1. Fetch latest from all remotes: `git fetch --all` in each service dir
2. Scan for features from multiple sources:
   - **PRDs:** `ls docs/prd/*.md` → extract slugs
   - **Feature branches:** `git branch -r | grep 'feature/.*/vc-'` in each service → extract slugs
   - **Track files:** `find . -name '*.track.md'` across all services → extract slugs
   - **Wireframes:** `ls web/src/app/(dashboard)/wireframes/` → extract slugs
   - **JJ workspaces:** if `workspaces` map exists in `.workflow-state.json`, read each workspace's `.workflow-state.json` to discover features tracked in secondary workspaces
3. For each unique slug found, reconstruct the phase (same logic as resume):
   - PRD exists? Wireframe approved? Branch exists? Service PRDs? Track progress? PR?
4. Also check local `.workflow-state.json` for any features only tracked locally
5. Display a table:

```
All features (committed artifacts + local state):

  Slug                        Phase              Last Activity         Worker
  ─────────────────────────── ────────────────── ───────────────────── ──────────
→ brand-health-dashboard      implementation     2026-03-20 (track)    fahrizky
  user-analytics              pr_review          2026-03-18 (PR #42)   daffa
  kv-generator                prd_creation       2026-03-15 (PRD)      arhen

→ = active in your local .workflow-state.json
```

6. Show: "To pick up a feature: `/workflow resume <slug>`"

### `/workflow resume <slug>`
Resume a feature started by another team member.

**Steps:**
1. Fetch latest: `git fetch --all` in each service dir
2. Scan committed artifacts to reconstruct phase:
   - PRD exists at `docs/prd/<slug>.md`? → past prd_creation
   - Wireframe directory exists with approved metadata.json? → past wireframe_review
   - Feature branch `feature/<worker>/vc-<slug>` exists? → past branch_creation
   - Service PRDs exist? → past prd_splitting
   - Track files show progress? → in implementation
   - PR URL in track file? → in pr_review
3. Read the latest session log from track files to show who last worked on it and what they did
4. Create/update feature entry in `.workflow-state.json` with reconstructed state
5. Set `active_feature` to `<slug>`
6. Ask user for their worker name if not already set
7. Inform user:
   - Reconstructed phase
   - Last session log entry (who did what)
   - What's left to do (remaining tasks from track file)
   - Which branch to checkout: `git checkout feature/<worker>/vc-<slug>`

### `/workflow add-wireframe <wireframe-slug>`
Add a new wireframe to the current feature.

**Steps:**
1. Read `.workflow-state.json`
2. Verify active feature is in `wireframe` or `wireframe_review` phase
3. Add new wireframe entry with slug, empty path, current worker as owner, null approval
4. Save state
5. Inform user and delegate to frndos-wireframe

### `/workflow progress`
Show detailed progress for the active feature.

**Steps:**
1. Read `.workflow-state.json` and all related files
2. Show:
   - Phase progression timeline
   - Wireframe status for each wireframe
   - Service PRD status
   - Track file task completion percentage
   - Session log summary

### `/workflow mode`
Switch Claude Code between Agent Session and Team Session mode.

**Steps:**
1. Check if Claude Code is the active tool (`.claude/settings.json` exists or `.claude/` dir exists)
   - If not Claude Code: "This command is only available for Claude Code."
2. Read `.claude/settings.json` to determine current mode:
   - If `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` is `"1"`: current mode is **Team Session**
   - Otherwise: current mode is **Agent Session**
3. Show current mode and ask to switch:

   > "Current mode: **Agent Session**"
   > - Switch to Team Session (EXPERIMENTAL — parallel engineers + architect, more tokens)
   > - Keep current mode

   Or:

   > "Current mode: **Team Session** (EXPERIMENTAL)"
   > - Switch to Agent Session (sequential, single agent, recommended)
   > - Keep current mode

4. If user chooses to switch:
   - Read current `.claude/settings.json`
   - **To Team Session:** set `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` to `"1"`
   - **To Agent Session:** remove `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` from `env` (or set to `"0"`)
   - Write updated `.claude/settings.json` (preserve other settings)
   - Update `.onboard-state.json`: set `claude_session_mode` to `"team"` or `"agent"`
   - Inform user: "Switched to [mode]. Restart your agent session for changes to take effect."
5. If user keeps current mode: "Keeping [current mode]. No changes made."
