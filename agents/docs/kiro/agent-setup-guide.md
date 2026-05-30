# Kiro Agent & Skill Setup Guide

Reference: 
- [kiro.dev/docs/cli/custom-agents](https://kiro.dev/docs/cli/custom-agents.md)
- [kiro.dev/docs/cli/skills](https://kiro.dev/docs/cli/skills.md)
- [kiro.dev/llms.txt](https://kiro.dev/llms.txt)

---

## Creating an Agent

### Quick way (AI-assisted)

From inside a Kiro chat session:

```text
> /agent create my-agent
```

Or from the terminal:

```bash
kiro-cli agent create my-agent
```

Flags:

| Flag | Description |
|------|-------------|
| `--directory workspace` | Save to `.kiro/agents/` (project-local) |
| `--directory global` | Save to `~/.kiro/agents/` (default) |
| `--description "..."` | Agent description (AI-assisted mode only) |
| `--mcp-server name` | Include an MCP server (AI-assisted mode only) |
| `--manual` | Open editor instead of AI generation |
| `--from agent-name` | Base on an existing agent (implies `--manual`) |

### Manual: agent config file

Agents are JSON files in `~/.kiro/agents/` (global) or `.kiro/agents/` (workspace). The filename (without `.json`) becomes the agent name.

```json
{
  "name": "my-agent",
  "description": "What this agent does",
  "prompt": "You are a specialist in...",
  "model": "claude-sonnet-4",
  "tools": ["read", "write", "shell"],
  "allowedTools": ["read"],
  "toolsSettings": {
    "shell": {
      "autoAllowReadonly": true
    }
  },
  "resources": [
    "file://README.md",
    "skill://.kiro/skills/**/SKILL.md",
    "skill://~/.kiro/skills/**/SKILL.md"
  ],
  "welcomeMessage": "Ready. What's the task?"
}
```

### Key config fields

| Field | Description |
|-------|-------------|
| `name` | Agent identifier (lowercase, hyphens OK) |
| `description` | Human-readable purpose |
| `prompt` | System prompt — inline text or `file://./prompt.md` |
| `model` | Model ID (e.g. `claude-sonnet-4`) |
| `tools` | All tools the agent can potentially use. Use `"*"` for all. |
| `allowedTools` | Tools that run without asking permission. Supports glob patterns. |
| `toolsSettings` | Per-tool config (e.g. `allowedPaths`, `autoAllowReadonly`) |
| `resources` | Files and skills loaded into context |
| `mcpServers` | MCP servers the agent can access |
| `hooks` | Commands run at lifecycle events |
| `keyboardShortcut` | e.g. `"ctrl+a"` to switch to this agent |
| `welcomeMessage` | Message shown when switching to this agent |

### tools vs allowedTools

- `tools` — declares what the agent *can* use
- `allowedTools` — declares what runs *without prompting the user*

```json
"tools": ["read", "write", "shell", "@git"],
"allowedTools": ["read", "@git/git_status", "@git/git_diff"]
```

Wildcard patterns are supported in `allowedTools`:

```json
"allowedTools": ["read", "@git/git_*", "@fetch"]
```

### Prompt from a file

For long prompts, keep them in a separate file:

```json
"prompt": "file://./prompts/my-agent.md"
```

Path is relative to the agent config file.

### Resources

```json
"resources": [
  "file://README.md",
  "file://docs/**/*.md",
  "skill://.kiro/skills/**/SKILL.md",
  "skill://~/.kiro/skills/**/SKILL.md"
]
```

- `file://` — loaded into context at startup
- `skill://` — only metadata loaded at startup; full content loaded on demand

Custom agents do **not** load skills by default — you must add `skill://` URIs explicitly.

### Hooks

```json
"hooks": {
  "agentSpawn": [{ "command": "git status" }],
  "preToolUse": [{
    "matcher": "execute_bash",
    "command": "echo \"$(date) - bash:\" >> /tmp/audit.log"
  }],
  "postToolUse": [{
    "matcher": "fs_write",
    "command": "npm run lint"
  }]
}
```

Available triggers: `agentSpawn`, `userPromptSubmit`, `preToolUse`, `postToolUse`, `stop`.

### Using your agent

Switch inside a session:

```text
> /agent swap
```

Or start directly:

```bash
kiro-cli --agent my-agent
```

---

## Adding Skills to an Agent

Skills extend agent capabilities. They're loaded from:

- `.kiro/skills/` — workspace scope
- `~/.kiro/skills/` — global scope

### Enable skills in your agent config

Add skill resources to the agent's `resources` field:

```json
"resources": [
  "file://README.md",
  "skill://.kiro/skills/**/SKILL.md",
  "skill://~/.kiro/skills/**/SKILL.md"
]
```

Use `*/` glob patterns to load all skills, or reference specific skills:

```json
"resources": [
  "skill://.kiro/skills/pr-review/SKILL.md",
  "skill://~/.kiro/skills/cdk-deploy/SKILL.md"
]
```

**Important:** The default agent loads skills automatically. Custom agents must explicitly add `skill://` URIs to `resources`.

### How skills activate

- **Automatically** — Kiro matches your request against the skill's description
- **As slash command** — Type `/skill-name` to invoke directly

View loaded skills:

```text
> /context show
```

---

## File Locations Summary

| Type | Workspace (project) | Global (user) |
|------|---------------------|---------------|
| Agents | `.kiro/agents/*.json` | `~/.kiro/agents/*.json` |
| Skills | `.kiro/skills/<name>/SKILL.md` | `~/.kiro/skills/<name>/SKILL.md` |

Workspace takes precedence over global when names conflict.

---

## Example: Python Dev Agent

```json
{
  "name": "python-dev",
  "description": "Expert Python developer",
  "prompt": "You are an expert Python developer. Use `uv` for package management. Write clean, minimal code.",
  "model": "claude-sonnet-4",
  "tools": ["read", "write", "shell", "code"],
  "allowedTools": ["read", "code"],
  "toolsSettings": {
    "shell": { "autoAllowReadonly": true }
  },
  "resources": [
    "file://README.md",
    "skill://.kiro/skills/**/SKILL.md",
    "skill://~/.kiro/skills/**/SKILL.md"
  ],
  "welcomeMessage": "Ready for Python development. What's the task?"
}
```

## Example: CDK Deploy Skill

```
cdk-deploy/
├── SKILL.md
└── references/
    └── stack-patterns.md
```

**SKILL.md:**

```markdown
---
name: cdk-deploy
description: Deploy AWS CDK stacks. Use when deploying infrastructure, running cdk deploy, or troubleshooting CDK issues.
---

## Deployment workflow

1. Run `cdk synth` to validate templates
2. Run `cdk diff` to preview changes
3. Run `cdk deploy` and review IAM changes

For environment-specific patterns, see `references/stack-patterns.md`.
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Agent not found | Check JSON is in `~/.kiro/agents/` or `.kiro/agents/` with `.json` extension |
| Invalid JSON | Validate JSON syntax; check commas and quotes |
| Skills not loading in custom agent | Add `skill://` URIs to the agent's `resources` field |
| Skill not activating | Make the description more specific with keywords matching your request |
| Slash command not found | Verify folder name matches the skill `name` in frontmatter; check `/context show` |
| Tool requires permission every time | Add it to `allowedTools` |
