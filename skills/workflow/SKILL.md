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
   - PR: `{pr_url}` or "not submitted"

### `/workflow list`
Show ALL active features with their phases.

**Steps:**
1. Read `.workflow-state.json`
2. For each feature in `features{}`:
   - Show slug, phase, phase_entered, worker
   - Mark active feature with `→`

### `/workflow start <slug>`
Start a new feature workflow.

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
4. Create feature entry with phase: "prd_creation"
5. Set `active_feature` to `<slug>`
6. Set `phase_entered` to current timestamp
7. Ask user for worker name if not set
8. Save state
9. Inform user: "Feature `<slug>` started. Phase: prd_creation. Delegating to frndos-prd."

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

### `/workflow resume <slug>`
Resume a feature started by another team member.

**Steps:**
1. Scan committed artifacts to reconstruct phase:
   - PRD exists at `docs/prd/<slug>.md`? → past prd_creation
   - Wireframe directory exists with approved metadata.json? → past wireframe_review
   - Feature branch `feature/<slug>` exists? → past branch_creation
   - Service PRDs exist? → past prd_splitting
   - Track files show progress? → in implementation
   - PR URL in track file? → in pr_review
2. Create/update feature entry in `.workflow-state.json` with reconstructed state
3. Set `active_feature` to `<slug>`
4. Inform user of reconstructed state

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
