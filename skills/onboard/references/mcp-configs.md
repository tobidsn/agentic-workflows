# MCP Configuration Templates

All MCP configs follow tool-specific formats. Merge these into the appropriate config file — don't overwrite existing entries.

**Format differences by tool:**

| Tool | Config File | MCP Root Key | Command Format | Env Key |
|------|-------------|-------------|----------------|---------|
| Claude Code | `.mcp.json` | `mcpServers` | `"command": "npx", "args": [...]` | `env` |
| Cursor | `.cursor/mcp.json` | `mcpServers` | `"command": "npx", "args": [...]` | `env` |
| OpenCode | `opencode.json` | `mcp` | `"type": "local", "command": ["npx", ...]` | `environment` |

---

## Context7 (Required — all services)

### Claude Code (`.mcp.json`)

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@context7/mcp"]
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
      "args": ["-y", "@context7/mcp"]
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
      "command": ["npx", "-y", "@context7/mcp"],
      "enabled": true
    }
  }
}
```

---

## GitHub MCP (Required — all services)

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

### Claude Code (`.claude/settings.local.json`)

```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": ["-y", "@anthropic/lark-mcp"],
      "env": {
        "LARK_APP_ID": "<from-team-lead>",
        "LARK_APP_SECRET": "<from-team-lead>"
      }
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
      "args": ["-y", "@anthropic/lark-mcp"],
      "env": {
        "LARK_APP_ID": "<from-team-lead>",
        "LARK_APP_SECRET": "<from-team-lead>"
      }
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
      "command": ["npx", "-y", "@anthropic/lark-mcp"],
      "enabled": true,
      "environment": {
        "LARK_APP_ID": "<from-team-lead>",
        "LARK_APP_SECRET": "<from-team-lead>"
      }
    }
  }
}
```

**Credentials:** Contact arhen for the Lark App ID and Secret.

---

## Figma MCP (Optional — design-to-code)

### Claude Code (`.claude/settings.local.json`)

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic/figma-mcp"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "<your-personal-access-token>"
      }
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic/figma-mcp"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "<your-personal-access-token>"
      }
    }
  }
}
```

### OpenCode (`opencode.json`)

```json
{
  "mcp": {
    "figma": {
      "type": "local",
      "command": ["npx", "-y", "@anthropic/figma-mcp"],
      "enabled": true,
      "environment": {
        "FIGMA_ACCESS_TOKEN": "<your-personal-access-token>"
      }
    }
  }
}
```

**Credentials:** Generate a personal access token at https://www.figma.com/developers/api#access-tokens
