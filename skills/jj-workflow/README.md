# /jj-workflow — JJ Workspace Management

Parallel feature development using [JJ (Jujutsu)](https://martinvonz.github.io/jj/) colocated workspaces.

## What it does

Creates isolated working directories so separate Claude Code sessions can work on different features simultaneously. Each workspace gets its own `.workflow-state.json` and feature state machine, while sharing the same git commit graph — committed changes propagate instantly between workspaces.

## Key constraints

- **JJ colocated mode only** — all git commands stay unchanged. JJ is used exclusively for workspace management (`workspace add`, `workspace list`, `workspace forget`).
- **Multi-repo orchestration** — frndOS is 4 service repos (`api/`, `web/`, `ai-service/`, `data-service/`). JJ operations are applied per-service-repo, orchestrated at the top level.
- **Claude Code only** — Cursor/OpenCode don't benefit from separate working directories.
- **Separate from `/workflow`** — `/workflow` handles the feature state machine; `/jj-workflow` handles workspace isolation.

## Commands

| Command | Description |
|---------|-------------|
| `/jj-workflow init` | Initialize JJ colocated mode in service repos (runs `jj git init --colocate` per repo) |
| `/jj-workflow new <slug>` | Create a sibling workspace directory with JJ working copies + symlinked orchestration files |
| `/jj-workflow list` | List all workspaces with their current feature, phase, and worker |
| `/jj-workflow status` | Show current workspace type (primary/secondary) and JJ state per service repo |
| `/jj-workflow cleanup <slug>` | Remove a workspace — forget JJ workspaces, delete directory, update registry |

## Typical workflow

```
# 1. Initialize JJ (one-time, or during onboarding)
/jj-workflow init

# 2. Create a workspace for a new feature
/jj-workflow new feature-b
# → Creates ../frndos-feature-b/ with JJ working copies

# 3. Open a new terminal and start Claude Code there
cd ../frndos-feature-b/
claude
/workflow start feature-b

# 4. Work on feature-b independently while feature-a continues in the original workspace

# 5. When done, clean up from the primary workspace
/jj-workflow cleanup feature-b
```

## State schema

**Primary workspace** — `.workflow-state.json` gains an optional `workspaces` map:
```json
"workspaces": {
  "feature-b": {
    "path": "../frndos-feature-b",
    "created": "2026-03-31T08:00:00Z",
    "worker": "arhen"
  }
}
```

**Secondary workspace** — `.workflow-state.json` has a `workspace_meta` pointer:
```json
"workspace_meta": {
  "primary_workspace": "../frndos",
  "feature_slug": "feature-b",
  "is_jj_workspace": true
}
```

## Important notes

- **Port conflicts**: Can't run the same services in two workspaces simultaneously (same ports). Stop services in one before starting the other.
- **No nesting**: Only the primary workspace can create new workspaces.
- **Uncommitted changes are isolated**: Only committed changes propagate between workspaces.
- **Nix users**: JJ (`jujutsu`) is included in `flake.nix` automatically.
- **Non-Nix users**: Install with `brew install jj`.

## Files touched by this feature

| File | Change |
|------|--------|
| `skills/jj-workflow/SKILL.md` | New skill definition |
| `workflow/state-schema.json` | Added `workspaces`, `workspace_meta`, `workspace_entry` definitions |
| `flake.nix` | Added `jujutsu` to buildInputs |
| `agents/fragments/workflow-rules.md` | Added JJ workspace rules section |
| `agents/fragments/session-protocol.md` | Added workspace type detection (Step 0) |
| `skills/workflow/SKILL.md` | `/workflow start` suggests parallel workspace; `/workflow list-all` scans JJ workspaces |
| `agents/tools/claude-code/frndos-orchestra.md` | Workspace detection in Step 0; JJ commands in idle state |
| `skills/onboard/SKILL.md` | Step 2.5: optional JJ setup for Claude Code users |
| `manifest.json` | Registered new skill + updated hashes |
