# MCP Configuration Templates

All MCP configs follow tool-specific formats. Merge these into the appropriate config file — don't overwrite existing entries.

**Format differences by tool:**

| Tool | Config File | MCP Root Key | Command Format | Env Key |
|------|-------------|-------------|----------------|---------|
| Claude Code | `.mcp.json` | `mcpServers` | `"command": "npx", "args": [...]` | `env` |
| Cursor | `.cursor/mcp.json` | `mcpServers` | `"command": "npx", "args": [...]` | `env` |
| OpenCode | `opencode.json` | `mcp` | `"type": "local", "command": ["npx", ...], "enabled": true` | `environment` |

---

## Context7 (Required — all services)

Package: `@upstash/context7-mcp`

### Claude Code (`.mcp.json`)

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

### OpenCode (`opencode.json`)

```json
{
  "mcp": {
    "context7": {
      "type": "local",
      "command": ["npx", "-y", "@upstash/context7-mcp"],
      "enabled": true
    }
  }
}
```

---

## GitHub MCP (Required — all services)

Package: `@modelcontextprotocol/server-github`

### Claude Code (`.mcp.json`)

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-github-pat>"
      }
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-github-pat>"
      }
    }
  }
}
```

### OpenCode (`opencode.json`)

```json
{
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "enabled": true,
      "environment": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-github-pat>"
      }
    }
  }
}
```

**Credentials:** Use the same PAT from `gh auth` or generate one at https://github.com/settings/tokens

---

## Laravel Boost MCP (Required — API service only)

Package: `@nicholasgriffintn/laravel-boost-mcp`

Configure inside `api/` directory only.

### Claude Code (`api/.mcp.json`)

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "npx",
      "args": ["-y", "@nicholasgriffintn/laravel-boost-mcp"]
    }
  }
}
```

### Cursor (`api/.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "npx",
      "args": ["-y", "@nicholasgriffintn/laravel-boost-mcp"]
    }
  }
}
```

### OpenCode (`api/opencode.json`)

```json
{
  "mcp": {
    "laravel-boost": {
      "type": "local",
      "command": ["npx", "-y", "@nicholasgriffintn/laravel-boost-mcp"],
      "enabled": true
    }
  }
}
```

---

## Sentry MCP (Optional — production debugging)

Package: `@sentry/mcp-server`

### Claude Code (`.mcp.json`)

```json
{
  "mcpServers": {
    "sentry": {
      "command": "npx",
      "args": ["-y", "@sentry/mcp-server"],
      "env": {
        "SENTRY_AUTH_TOKEN": "<your-sentry-token>"
      }
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "sentry": {
      "command": "npx",
      "args": ["-y", "@sentry/mcp-server"],
      "env": {
        "SENTRY_AUTH_TOKEN": "<your-sentry-token>"
      }
    }
  }
}
```

### OpenCode (`opencode.json`)

```json
{
  "mcp": {
    "sentry": {
      "type": "local",
      "command": ["npx", "-y", "@sentry/mcp-server"],
      "enabled": true,
      "environment": {
        "SENTRY_AUTH_TOKEN": "<your-sentry-token>"
      }
    }
  }
}
```

**Credentials:** Generate at https://sentry.io/settings/account/api/auth-tokens/

---

## Lark MCP (Optional — PRD from Lark docs)

Package: `@larksuiteoapi/lark-mcp`

**Note:** Lark MCP uses command-line args for credentials, not env vars.

### Claude Code (`.mcp.json`)

```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": ["-y", "@larksuiteoapi/lark-mcp", "mcp", "-a", "<LARK_APP_ID>", "-s", "<LARK_APP_SECRET>"]
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": ["-y", "@larksuiteoapi/lark-mcp", "mcp", "-a", "<LARK_APP_ID>", "-s", "<LARK_APP_SECRET>"]
    }
  }
}
```

### OpenCode (`opencode.json`)

```json
{
  "mcp": {
    "lark": {
      "type": "local",
      "command": ["npx", "-y", "@larksuiteoapi/lark-mcp", "mcp", "-a", "<LARK_APP_ID>", "-s", "<LARK_APP_SECRET>"],
      "enabled": true
    }
  }
}
```

**Credentials:** Contact arhen for the Lark App ID and Secret.

---

## Figma MCP (Optional — design-to-code)

**Remote server** — no npm package needed. Connects to `https://mcp.figma.com/mcp`.

### Claude Code

Run this command (or add manually to `.mcp.json`):
```bash
claude mcp add --transport http figma https://mcp.figma.com/mcp
```

Or manually in `.mcp.json`:
```json
{
  "mcpServers": {
    "figma": {
      "url": "https://mcp.figma.com/mcp",
      "type": "http"
    }
  }
}
```

### Cursor

Use the Cursor plugin system:
```
/add-plugin figma
```

Or manually in `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "figma": {
      "url": "https://mcp.figma.com/mcp",
      "type": "http"
    }
  }
}
```

### OpenCode (`opencode.json`)

```json
{
  "mcp": {
    "figma": {
      "type": "remote",
      "url": "https://mcp.figma.com/mcp",
      "enabled": true
    }
  }
}
```

**No API token needed** — authenticates via browser OAuth when first used.
See: https://help.figma.com/hc/en-us/articles/32132100833559
