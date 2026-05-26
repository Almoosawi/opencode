# Enterprise Managed Config Schema Reference

**Version:** opencode v1.15.10
**Date:** 2026-05-25
**Purpose:** Technical reference for IT/Cyber Security to deploy centrally-managed opencode configuration via Group Policy.

---

## 1. Deployment Path

| OS | Config directory | Filenames read (in order) |
|---|---|---|
| Windows | `%ProgramData%\opencode\` (typically `C:\ProgramData\opencode\`) | `opencode.json`, `opencode.jsonc` |
| macOS | `/Library/Application Support/opencode/` | `opencode.json`, `opencode.jsonc` |
| Linux | `/etc/opencode/` | `opencode.json`, `opencode.jsonc` |

**Important:** Only `opencode.json` and `opencode.jsonc` are loaded. A file named `managed.json` will be silently ignored.

JSONC (JSON with Comments) is supported. Use `.jsonc` if you want inline documentation in the deployed file.

### Override behavior

Managed config is loaded **last** in the config merge chain (after global user config, project config, and console/org config). Because the merge uses deep-merge semantics, managed config fields override all user-level settings. This is the correct behavior for enterprise enforcement.

**Full merge order (config.ts `loadInstanceState`):**
1. Well-known remote config (from authenticated providers)
2. Global user config (`~/.config/opencode/opencode.json`)
3. `OPENCODE_CONFIG` env var path (if set)
4. Project-local config (`.opencode/opencode.json` in working directory)
5. Console/org config (if enterprise account is active)
6. **Managed config** (`ProgramData/opencode/`) -- **this is where IT deploys**
7. macOS MDM managed preferences (Darwin only, highest priority)

---

## 2. Full Schema Reference

The managed config file accepts every field in the `Config.Info` schema (defined in `packages/opencode/src/config/config.ts`). Below are all fields, grouped by security relevance.

### 2.1 Security-Critical Fields

#### `permission` -- Tool Permission Rules

Controls what the AI agent is allowed to do. This is the primary security lever.

**Type:** `PermissionConfig` -- either a single action string (`"allow"`, `"deny"`, `"ask"`) applied to all tools, or an object with per-tool rules.

**Per-tool rule structure:**
```jsonc
{
  "permission": {
    // Per-tool rules. Each value is either:
    //   - A string: "allow" | "deny" | "ask"  (applies to all patterns)
    //   - An object: { "<glob-pattern>": "allow" | "deny" | "ask" }
    "read": "allow",               // Allow reading all files
    "edit": { "*": "ask" },         // Require confirmation for all edits
    "bash": { "*": "ask" },         // Require confirmation for shell commands
    "glob": "allow",
    "grep": "allow",
    "list": "allow",
    "task": "ask",
    "external_directory": "deny",   // Block access outside project
    "todowrite": "deny",
    "question": "deny",
    "webfetch": "deny",             // Block web fetching
    "websearch": "deny",            // Block web searching
    "repo_clone": "deny",
    "repo_overview": "deny",
    "lsp": "allow",
    "doom_loop": "ask",
    "skill": "deny"
  }
}
```

**Known permission keys** (from `ConfigPermission.InputObject`):
| Key | Controls | Actions |
|---|---|---|
| `read` | File reading | `allow`, `deny`, `ask` (supports glob patterns) |
| `edit` | File writing/editing/patching | `allow`, `deny`, `ask` (supports glob patterns) |
| `glob` | File globbing/listing | `allow`, `deny`, `ask` |
| `grep` | Code searching | `allow`, `deny`, `ask` |
| `list` | Directory listing | `allow`, `deny`, `ask` |
| `bash` | Shell command execution | `allow`, `deny`, `ask` (supports glob patterns) |
| `task` | Task/subagent spawning | `allow`, `deny`, `ask` |
| `external_directory` | Access outside project dir | `allow`, `deny`, `ask` (supports glob patterns) |
| `todowrite` | Writing TODO items | `allow`, `deny`, `ask` |
| `question` | Asking user questions | `allow`, `deny`, `ask` |
| `webfetch` | Fetching web content | `allow`, `deny`, `ask` |
| `websearch` | Web searches | `allow`, `deny`, `ask` |
| `repo_clone` | Cloning repositories | `allow`, `deny`, `ask` |
| `repo_overview` | Repository overview | `allow`, `deny`, `ask` |
| `lsp` | Language server protocol | `allow`, `deny`, `ask` |
| `doom_loop` | Doom loop detection | `allow`, `deny`, `ask` |
| `skill` | Skill execution | `allow`, `deny`, `ask` |

Additional arbitrary tool names are accepted via the rest schema (`Record<string, Rule>`).

**Evaluation logic:** Rules are evaluated via `findLast()` -- the last matching rule wins. Rulesets are merged in order: `defaults` -> `agent-specific` -> `user config`. Since managed config overrides user config, managed permission rules will be the final authority.

**`"deny"` is hard enforcement.** When a tool is denied, the agent receives a `DeniedError` with the ruleset displayed. There is no user override for `"deny"` rules -- they cannot be bypassed at runtime.

#### `tools` -- Legacy Tool Enable/Disable

**Type:** `Record<string, boolean>`

Deprecated shorthand. Converted internally to `permission` rules: `true` becomes `"allow"`, `false` becomes `"deny"`. Keys `write`, `edit`, `patch` all map to `permission.edit`.

Prefer using `permission` directly for clarity.

#### `enabled_providers` -- Provider Allowlist (Fail-Closed)

**Type:** `string[]`

When set, **ONLY** these provider IDs are enabled. All other providers are completely ignored. This is the fail-closed provider restriction mechanism.

```jsonc
{
  "enabled_providers": ["azure-openai", "bedrock"]
}
```

#### `disabled_providers` -- Provider Blocklist

**Type:** `string[]`

Disables specific providers that would otherwise be auto-loaded. Use `enabled_providers` instead for a fail-closed posture.

#### `share` -- Session Sharing Control

**Type:** `"manual" | "auto" | "disabled"`

Controls whether sessions can be shared externally. Set to `"disabled"` to prevent any external data transmission via sharing.

#### `autoupdate` -- Auto-Update Control

**Type:** `boolean | "notify"`

Set to `false` to disable automatic updates. Corporate deployments should disable this since updates are controlled via MSI distribution.

#### `enterprise` -- Enterprise URL

**Type:** `{ url?: string }`

Enterprise console URL. Set if using opencode's enterprise features.

### 2.2 Operational Configuration Fields

#### `server` -- HTTP Server Configuration

**Type:** `ServerConfig`

```jsonc
{
  "server": {
    "port": 3000,           // Port to listen on (positive integer)
    "hostname": "127.0.0.1", // Hostname to bind
    "mdns": false,           // Disable mDNS service discovery
    "mdnsDomain": "opencode.local", // mDNS domain name
    "cors": []               // Additional CORS origins
  }
}
```

**NOTE:** Server authentication (username/password) is NOT configurable via managed JSON. See Section 4 for the server auth gap analysis.

#### `model` -- Default Model

**Type:** `string` (format: `provider/model-name`)

```jsonc
{
  "model": "azure-openai/gpt-4o"
}
```

#### `small_model` -- Small Model for Background Tasks

**Type:** `string` (format: `provider/model-name`)

Used for title generation, summaries, etc.

#### `provider` -- Provider Configuration

**Type:** `Record<string, ProviderConfig>`

Configure custom provider endpoints, API keys (via env var references), model lists, timeouts.

```jsonc
{
  "provider": {
    "azure-openai": {
      "api": "azure-openai",
      "options": {
        "baseURL": "https://your-instance.openai.azure.com",
        "apiKey": "$env:AZURE_OPENAI_API_KEY"
      }
    }
  }
}
```

#### `mcp` -- MCP Server Configuration

**Type:** `Record<string, McpConfig>`

Configure Model Context Protocol servers (local or remote).

```jsonc
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["node", "path/to/server.js"],
      "enabled": true,
      "timeout": 5000
    }
  }
}
```

#### `shell` -- Default Shell

**Type:** `string`

Default shell for terminal and bash tool.

#### `logLevel` -- Log Level

**Type:** `"DEBUG" | "INFO" | "WARN" | "ERROR"`

### 2.3 Agent Configuration

#### `agent` -- Agent Definitions

**Type:** Object with named agents, each having:

| Field | Type | Description |
|---|---|---|
| `model` | `string` | Model ID (provider/model format) |
| `variant` | `string` | Model variant |
| `temperature` | `number` | Temperature setting |
| `top_p` | `number` | Top-p setting |
| `prompt` | `string` | System prompt |
| `permission` | `PermissionConfig` | Per-agent permission overrides |
| `disable` | `boolean` | Disable the agent |
| `mode` | `"subagent" \| "primary" \| "all"` | Agent mode |
| `hidden` | `boolean` | Hide from autocomplete |
| `steps` | `positive integer` | Max agentic iterations |

Built-in agent names: `build`, `plan`, `general`, `explore`, `scout`, `title`, `summary`, `compaction`.

#### `default_agent` -- Default Agent Selection

**Type:** `string`

Must be a primary agent name. Falls back to `"build"` if invalid.

### 2.4 UI & Behavior Fields

| Field | Type | Description |
|---|---|---|
| `snapshot` | `boolean` | Enable/disable filesystem snapshots (default: `true`) |
| `username` | `string` | Display name (overrides system username) |
| `instructions` | `string[]` | Additional instruction file paths/patterns |
| `formatter` | `FormatterConfig` | Code formatter settings |
| `lsp` | `LSPConfig` | Language server settings |
| `attachment` | `AttachmentConfig` | Image size limits and resizing |
| `watcher.ignore` | `string[]` | File watcher ignore patterns |
| `layout` | `LayoutConfig` | Deprecated -- always stretch |
| `skills` | `SkillsConfig` | Additional skill folder paths |
| `reference` | `ReferenceConfig` | Named git/directory references |
| `command` | `Record<string, CommandConfig>` | Custom commands |
| `plugin` | `PluginSpec[]` | Plugin specifications |

### 2.5 Advanced / Experimental

#### `compaction` -- Context Compaction

```jsonc
{
  "compaction": {
    "auto": true,           // Auto-compact when context is full
    "prune": true,           // Prune old tool outputs
    "tail_turns": 2,         // Recent turns to keep verbatim
    "preserve_recent_tokens": 0, // Tokens from recent turns to preserve
    "reserved": 0            // Token buffer for compaction
  }
}
```

#### `tool_output` -- Tool Output Truncation

```jsonc
{
  "tool_output": {
    "max_lines": 2000,      // Max lines before truncation
    "max_bytes": 51200       // Max bytes before truncation
  }
}
```

#### `experimental` -- Experimental Flags

```jsonc
{
  "experimental": {
    "disable_paste_summary": false,
    "batch_tool": false,
    "openTelemetry": false,
    "primary_tools": [],
    "continue_loop_on_deny": false,
    "mcp_timeout": 5000
  }
}
```

---

## 3. Proposed Corporate Deployment JSON

Deploy this file to `C:\ProgramData\opencode\opencode.jsonc` via Group Policy.

```jsonc
// OpenCode Corporate Managed Configuration
// Deployed via Group Policy to %ProgramData%\opencode\opencode.jsonc
// This file overrides all user-level configuration.
// Last updated: 2026-05-25
{
  "$schema": "https://opencode.ai/config.json",

  // --- AUTO-UPDATE: Disabled ---
  // Updates are controlled via MSI distribution through SCCM/Intune.
  "autoupdate": false,

  // --- SHARING: Disabled ---
  // No session data transmitted externally.
  "share": "disabled",

  // --- PROVIDER ALLOWLIST (Fail-Closed) ---
  // ONLY these providers are active. All others are blocked.
  // Adjust this list to match your approved LLM endpoints.
  "enabled_providers": [
    "azure-openai"
    // Add other approved providers here, e.g.:
    // "bedrock",
    // "copilot"
  ],

  // --- PROVIDER CONFIGURATION ---
  // Configure your approved provider endpoints.
  "provider": {
    "azure-openai": {
      "api": "azure-openai",
      "options": {
        "baseURL": "https://YOUR-INSTANCE.openai.azure.com"
        // API key should be set via environment variable, not here.
        // Set AZURE_OPENAI_API_KEY via Group Policy environment variables.
      }
    }
  },

  // --- DEFAULT MODEL ---
  "model": "azure-openai/gpt-4o",

  // --- TOOL PERMISSIONS (Defense in Depth) ---
  // Managed permissions override user config. "deny" is hard-enforced.
  "permission": {
    "read": "allow",
    "edit": "ask",
    "glob": "allow",
    "grep": "allow",
    "list": "allow",
    "bash": "ask",
    "task": "ask",
    "lsp": "allow",
    "external_directory": "deny",
    "todowrite": "allow",
    "question": "deny",
    "webfetch": "deny",
    "websearch": "deny",
    "repo_clone": "deny",
    "repo_overview": "deny",
    "doom_loop": "ask",
    "skill": "deny"
  },

  // --- SERVER: Restrict binding ---
  // Server auth requires env vars (see deployment instructions).
  "server": {
    "hostname": "127.0.0.1",
    "mdns": false
  },

  // --- SNAPSHOTS: Enabled ---
  "snapshot": true,

  // --- MCP SERVERS: None by default ---
  // Add approved MCP servers here if needed.
  // "mcp": {}

  // --- EXPERIMENTAL: Conservative defaults ---
  "experimental": {
    "openTelemetry": false,
    "batch_tool": false
  }
}
```

---

## 4. Gaps and Server Auth Analysis

### 4.1 Server Authentication Gap

**Problem:** Server authentication (HTTP Basic Auth) is controlled exclusively via environment variables `OPENCODE_SERVER_PASSWORD` and `OPENCODE_SERVER_USERNAME` (defined in `packages/opencode/src/server/auth.ts`). These fields are **not** part of the managed config JSON schema. The `server` config object only has `port`, `hostname`, `mdns`, `mdnsDomain`, and `cors`.

**Impact:** If a user runs `opencode serve`, the HTTP server starts without authentication unless the env vars are set. Managed config alone cannot enforce server auth.

#### Option A: Deploy env vars via Group Policy (No Code Change)

Set environment variables via Group Policy Preferences (GPP):

| Variable | Value | Scope |
|---|---|---|
| `OPENCODE_SERVER_PASSWORD` | Strong random password | Machine-level (HKLM) |
| `OPENCODE_SERVER_USERNAME` | `opencode` (default) | Machine-level (HKLM) |

**Group Policy path:** Computer Configuration -> Preferences -> Windows Settings -> Environment

**Pros:**
- Zero code changes required
- Works with current v1.15.10 release
- Standard enterprise deployment pattern

**Cons:**
- Env vars are visible to any process running as the same user (`set` command, `/proc/*/environ`)
- Not part of the managed config story -- separate GPO needed
- Cannot be audited by reading the managed config file alone

#### Option B: Add Auth Fields to Managed Config Schema (Code Change Required)

Add `username` and `password` fields to `ConfigServer.Server` schema, and modify `auth.ts` to read from config as a fallback when env vars are not set.

**Estimated code change:**
1. `packages/opencode/src/config/server.ts` -- Add `username: Schema.optional(Schema.String)` and `password: Schema.optional(Schema.String)` to `Server` schema
2. `packages/opencode/src/server/auth.ts` -- Fall back to config values when env vars are absent
3. Net change: ~15 lines across 2 files

**Pros:**
- Single deployment artifact (managed JSON) controls everything
- Config-as-code auditing
- Consistent with managed config philosophy

**Cons:**
- Password in plaintext JSON on disk (though `ProgramData` is admin-writable only)
- Requires maintaining a code fork delta
- Upstream may add their own auth config in a future version

#### Recommendation

**Use Option A for initial deployment** (zero code change, standard GPO pattern). If the plaintext env var visibility is unacceptable to the security team, implement Option B as a hardening branch (`hardening/server-auth-config`).

### 4.2 No Server Disable Switch

There is no managed config field to completely prevent the server from starting. The `server` config only controls how it behaves when started, not whether it can be started.

**Mitigation:** The server only starts when explicitly invoked via `opencode serve` or `opencode web`. It does not auto-start. Binding to `127.0.0.1` (via managed config) plus requiring authentication (via env vars) provides adequate protection.

### 4.3 MCP Server Restrictions

MCP servers defined in managed config are additive -- they merge with user-defined MCP servers. There is no managed config mechanism to **block** user-defined MCP servers or restrict MCP to an allowlist.

**Mitigation:** Tool permissions (`skill: "deny"`) can prevent the agent from using MCP-provided tools, but the MCP connection itself still occurs.

### 4.4 Plugin Restrictions

Similar to MCP, there is no managed config mechanism to block user plugins. The `plugin` field is additive via merge.

**Mitigation:** Monitor via endpoint management. If critical, a code change could add a `plugin_allowlist` field.

### 4.5 Instructions Override Limitation

The `instructions` field uses array concatenation (not replacement) during merge. Managed instructions are appended to user instructions, not replacing them. A user cannot override managed instructions, but managed config also cannot remove user-provided instructions.

---

## 5. Deployment Instructions for IT

### 5.1 File Deployment

1. Create the directory `C:\ProgramData\opencode\` (if it does not exist)
2. Deploy the config file as `C:\ProgramData\opencode\opencode.jsonc`
3. Set file permissions: `SYSTEM` and `Administrators` have Full Control; `Users` have Read
4. Deploy via Group Policy Files Preferences or SCCM/Intune package

**Group Policy Preferences path for file deployment:**
Computer Configuration -> Preferences -> Windows Settings -> Files

| Setting | Value |
|---|---|
| Source file | `\\your-share\opencode\opencode.jsonc` |
| Destination | `%ProgramData%\opencode\opencode.jsonc` |
| Action | Replace |

### 5.2 Environment Variables (for server auth)

Deploy via Group Policy Preferences -> Environment:

| Variable | Value | Scope |
|---|---|---|
| `OPENCODE_SERVER_PASSWORD` | (generate strong random) | System |
| `OPENCODE_SERVER_USERNAME` | `opencode` | System |

### 5.3 Verification

After deployment, verify on a target machine:

```powershell
# Check managed config exists
Test-Path "$env:ProgramData\opencode\opencode.jsonc"

# Check content is valid JSON
Get-Content "$env:ProgramData\opencode\opencode.jsonc" | ConvertFrom-Json

# Check env vars (if deploying server auth)
[System.Environment]::GetEnvironmentVariable("OPENCODE_SERVER_PASSWORD", "Machine")
```

### 5.4 JSONC Format Notes

- JSONC supports `//` single-line comments and `/* */` block comments
- The opencode parser (`jsonc-parser` library) handles both formats
- Comments are preserved on file read but may be stripped if the application rewrites the file (managed config is read-only from the application's perspective, so this is not a concern)

---

## 6. Summary

| Metric | Value |
|---|---|
| Total schema fields | 30 top-level fields in `Config.Info` |
| Security-critical fields | 6 (`permission`, `tools`, `enabled_providers`, `disabled_providers`, `share`, `autoupdate`) |
| Operational fields | 8 (`server`, `model`, `small_model`, `provider`, `mcp`, `shell`, `logLevel`, `enterprise`) |
| Gaps identified | 4 (server auth, no server disable, no MCP allowlist, no plugin allowlist) |
| Code changes recommended | Optional -- server auth in managed config (Option B, ~15 lines, 2 files) |
| Deployment method | Group Policy file copy to `%ProgramData%\opencode\` |
