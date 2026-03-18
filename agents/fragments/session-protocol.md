## SESSION START PROTOCOL (MANDATORY)

**This protocol MUST be executed before ANY other work. No exceptions.**

### Step 0: Detect workspace state

Check the workspace to determine what's needed:

1. **No service directories** (none of `api/`, `web/`, `ai-service/`, `data-service/` exist):
   → Fresh workspace. Tell user: "This workspace hasn't been set up yet. Run `/onboard` to configure your development environment."
   → Do NOT proceed with workflow commands until onboarding is complete.

2. **`.onboard-state.json` exists and `status` is `"in_progress"`**:
   → Onboarding was started but not fully completed. Check what's missing:
   - Read `env_status` — are all selected services' .env files present?
   - Read `steps.db_setup` — is the database restored?
   - If critical items are missing, tell user:
     "Onboarding is incomplete. Missing: [list items]. Run `/onboard resume` to continue, or `/onboard verify` to re-check."
   - If user wants to proceed anyway with `/workflow start`, **BLOCK** and explain what will fail without the missing items.

3. **`.onboard-state.json` exists and `status` is `"completed"`** (or no `.onboard-state.json` but service directories exist):
   → Workspace is set up. Check for `.workflow-state.json`:
   - If it exists → proceed normally with active features
   - If it doesn't → welcome user with `/workflow start`, `/workflow list`

4. **`.workflow-state.json` exists**:
   → Proceed normally through all steps.

### Step 1: Check for instruction updates

```bash
bash .agentic-workflows/scripts/update-check.sh
```

If the script doesn't exist, bootstrap it:

```bash
curl -sL "https://raw.githubusercontent.com/alva-intelligence/agentic-workflows/main/scripts/update-check.sh" \
  -o /tmp/aw-update-check.sh && bash /tmp/aw-update-check.sh --bootstrap
```

### Step 2: Load workflow state

Read `.workflow-state.json` to determine:
- Which feature is currently active (`active_feature`)
- What phase it's in (`features[active_feature].phase`)
- Who the current worker is (`worker`)

If `.workflow-state.json` doesn't exist, no features are active yet.

### Step 3: Sync latest code

Only if inside a git repository with a remote:

```bash
git fetch origin && git pull --rebase origin $(git branch --show-current)
```

If lockfiles changed, update dependencies:
- `bun install` (web — only if `web/` exists)
- `composer install` (api — only if `api/` exists)
- `uv sync` (python services — only if they exist)

If conflicts arise:
1. List ALL conflicted files with a summary
2. Ask user: "There are merge conflicts. Would you like to: (A) resolve them yourself, or (B) let me resolve them?"
3. If user picks B, resolve and show resolution for approval before committing
4. NEVER auto-resolve conflicts silently

### Step 4: Service health checks

Only check services that **actually exist** in the workspace:

| Service | Check If Exists | Health Check |
|---------|----------------|-------------|
| API | `api/` directory | `curl -sf http://localhost:9191/health` |
| Frontend | `web/` directory | `curl -sf http://localhost:3000` |
| AI Service | `ai-service/` directory | `curl -sf http://localhost:8000/health` |
| Data Service | `data-service/` directory | `curl -sf http://localhost:9999/health` |
| PostgreSQL | `api/` exists | `pg_isready -h localhost -p 5432` |
| ClickHouse | `data-service/` exists | `curl -sf http://localhost:8123/ping` |
| Redis | `ai-service/` exists | `redis-cli ping` |

If any required service is DOWN:
- Tell user which services are not running
- Offer: "Should I run `./run-all.sh` to start all services?"
- Or: "Should I start just the specific services needed?"
- Wait for services to be healthy before proceeding

### Step 5: Route to correct agent

Based on `.workflow-state.json`, automatically delegate to the appropriate `frndos-*` agent for the current phase. Do NOT tell the user to manually invoke an agent — delegate directly.
