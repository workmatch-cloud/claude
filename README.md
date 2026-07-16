# Cost-Controlled OpenCode Configuration for Anthropic

This repository contains a conservative OpenCode configuration designed to:

- use only Anthropic models;
- start every session in a read-only planning agent;
- use Claude Haiku 4.5 for routine analysis and exploration;
- reserve Claude Sonnet 5 for controlled implementation;
- require approval before edits, most shell commands, web searches, and URL fetches;
- block access to secrets and paths outside the active project;
- limit agent iterations to reduce accidental cost;
- disable public session sharing, MCP servers, plugins, and expensive general-purpose subagents by default.

## Files

### `opencode.jsonc`

The main OpenCode configuration. It uses JSONC, so comments are allowed.

### `AGENTS.md`

The configuration references `AGENTS.md` as the repository instruction file:

```jsonc
"instructions": ["AGENTS.md"]
```

Create this file in the project root. Keep it short and include only durable repository rules, such as:

```md
# Project Instructions

- Follow the existing architecture and naming conventions.
- Do not modify generated files.
- Prefer small, reviewable changes.
- Run the narrowest relevant tests before broader test suites.
- Never expose credentials, tokens, or environment-variable values.
```

Remove `AGENTS.md` from the `instructions` array if the project does not need a repository-specific instruction file.

## Requirements

- OpenCode installed and available as the `opencode` command.
- An Anthropic API key with access to:
  - `claude-haiku-4-5`
  - `claude-sonnet-5`

Confirm the models available to the current account with:

```text
/models
```

## Installation

Copy `opencode.jsonc` to the root of the project:

```text
your-project/
├── opencode.jsonc
├── AGENTS.md
└── ...
```

A project-level configuration overrides standard global settings for that project.

For a global configuration, place the file at:

```text
~/.config/opencode/opencode.jsonc
```

A custom path can also be supplied through `OPENCODE_CONFIG`:

```bash
export OPENCODE_CONFIG=/absolute/path/to/opencode.jsonc
opencode
```

## Configure the Anthropic API Key

The key is intentionally not stored in the configuration. OpenCode reads it from:

```text
ANTHROPIC_API_KEY
```

### Linux or macOS

For the current terminal session:

```bash
export ANTHROPIC_API_KEY="your-api-key"
opencode
```

To persist it, add the export command to the appropriate shell profile, such as `~/.bashrc`, `~/.zshrc`, or an equivalent secure environment configuration.

### Windows PowerShell

For the current PowerShell session:

```powershell
$env:ANTHROPIC_API_KEY = "your-api-key"
opencode
```

To store it for the current Windows user:

```powershell
[Environment]::SetEnvironmentVariable(
  "ANTHROPIC_API_KEY",
  "your-api-key",
  "User"
)
```

Open a new terminal after setting a persistent user environment variable.

Do not place the real key in `opencode.jsonc`, `AGENTS.md`, `.env.example`, source control, screenshots, or logs.

## Validate the Resolved Configuration

Run:

```bash
opencode debug config
```

Check that:

- `enabled_providers` contains only `anthropic`;
- the default agent is `plan`;
- the resolved models are the intended Anthropic model IDs;
- the API key is supplied by the environment;
- project-level or managed configuration has not unexpectedly overridden the settings.

The `$schema` entry enables validation and autocomplete in editors that support JSON Schema.

## Daily Workflow

Start OpenCode in the project directory:

```bash
opencode
```

The session starts with the `plan` primary agent.

### 1. Plan First

Use `plan` to inspect the repository, identify affected files, review risks, and propose a change. This agent:

- uses Claude Haiku 4.5;
- cannot edit files;
- cannot execute shell commands;
- can invoke only the read-only `explore` subagent;
- stops after at most six agentic iterations.

Example request:

```text
Analyze how authentication is implemented and propose the smallest safe change.
Do not modify files.
```

### 2. Review the Plan

Check the proposed files, assumptions, tests, and migration impact before allowing implementation.

### 3. Switch to Build

Use `Tab` to cycle between primary agents and select `build`.

The `build` agent:

- uses Claude Sonnet 5;
- asks before modifying files;
- asks before most shell commands;
- automatically permits a small set of read-only commands;
- can invoke only the `explore` subagent;
- stops after at most twelve agentic iterations.

Example request:

```text
Implement the approved plan. Keep the patch minimal and ask before editing.
```

### 4. Review Every Approval

Approve only operations that match the task. Reject unexpected file writes, broad shell commands, unrelated searches, or access outside the project.

## Agent Overview

| Agent | Type | Model | Steps | Main purpose |
|---|---|---:|---:|---|
| `plan` | Primary | Claude Haiku 4.5 | 6 | Read-only analysis and planning |
| `build` | Primary | Claude Sonnet 5 | 12 | Controlled implementation |
| `explore` | Subagent | Claude Haiku 4.5 | 5 | Read-only file and symbol discovery |
| `general` | Disabled | — | — | Prevent broad or parallel execution |
| `scout` | Disabled | — | — | Prevent automatic external research |

## Permission Behavior

OpenCode permission outcomes are:

- `allow`: run without approval;
- `ask`: request approval;
- `deny`: block the action.

When multiple permission patterns match, the last matching rule wins. This is why `*.env.example` appears after the broader `*.env.*` denial: example files remain readable while real environment files remain blocked.

### Global Restrictions

The configuration:

- allows ordinary project-file reads;
- denies reads of `.env`, `.env.*`, `secrets/**`, and credential-like files;
- explicitly allows `.env.example`;
- denies access outside the active project;
- asks before web searches and URL fetches;
- blocks an identical tool call after repeated attempts;
- disables automatic session sharing.

### Build Shell Rules

These commands can run without approval:

```text
git status...
git diff...
git log...
git show...
rg ...
grep ...
```

All other shell commands require approval, except:

```text
git push...
rm ...
```

Those patterns are denied.

The configuration does not claim that every destructive command is listed. Commands not explicitly allowed or denied still require approval through the catch-all `"*": "ask"` rule.

### File Editing

The `build` agent uses:

```jsonc
"edit": "ask"
```

This covers file writes, edits, and patches. The `plan` and `explore` agents deny editing completely.

## Claude Sonnet 5 Thinking Setting

Claude Sonnet 5 enables adaptive thinking by default. This configuration explicitly sends:

```jsonc
"thinking": {
  "type": "disabled"
}
```

The setting appears under the model's `options` and under the `build` agent's `options`. The agent-level value makes the build behavior explicit and overrides the global model configuration for that agent when applicable.

Disabling adaptive thinking can reduce reasoning-token usage for routine implementation. It can also reduce quality on difficult architecture, debugging, or multi-file reasoning tasks.

For a difficult task, temporarily change the build agent to adaptive thinking:

```jsonc
"options": {
  "thinking": {
    "type": "adaptive"
  }
}
```

Do not use the older manual extended-thinking form with `budget_tokens` for Claude Sonnet 5.

## Cost-Control Features

### Lower-Cost Default Model

Both `model` and `small_model` use Claude Haiku 4.5. This keeps ordinary sessions and lightweight internal tasks away from Sonnet unless the `build` agent is selected.

### Step Limits

`steps` limits agentic iterations before OpenCode forces a text-only response:

- Plan: 6
- Build: 12
- Explore: 5

These are not token or currency limits. A single step can still be expensive if it reads large files or produces a large response.

### Context Compaction

```jsonc
"compaction": {
  "auto": true,
  "prune": true,
  "reserved": 12000
}
```

This enables automatic context compaction, prunes old tool output, and reserves context space for the compaction summary.

### Watcher Exclusions

The watcher ignores dependency folders, build output, logs, source maps, minified JavaScript, lock files, environment files, and secret directories. This reduces irrelevant file-system events.

Watcher exclusions do not replace permission rules. Sensitive files are also denied through `permission.read`.

### Prompt Cache Key

```jsonc
"setCacheKey": true
```

This asks the Anthropic provider integration to use a stable prompt cache key where supported. Actual cache savings depend on request structure, cache eligibility, and provider behavior.

## MCP Servers and Plugins

The configuration starts with no MCP servers and no plugins:

```jsonc
"mcp": {},
"plugin": []
```

Add them only when a project has a defined need. Review their permissions, commands, network access, and data exposure before enabling them.

## Language Server Support

```jsonc
"lsp": true
```

This enables built-in language-server integration where supported. It helps OpenCode locate definitions, references, diagnostics, and symbols without reading entire files unnecessarily.

Actual behavior depends on the language and available language server.

## Important Limitations

This configuration reduces accidental actions; it is not a complete security sandbox.

- A user can still approve a dangerous command.
- Model and tool behavior can change between OpenCode versions.
- Project configuration can override global configuration.
- Managed organizational settings can override other configuration layers.
- Shell commands not listed as denied may still be approved manually.
- Step limits do not impose a hard financial budget.
- Ignoring a file in the watcher does not automatically deny reading it.
- A secret already included in a prompt can still be sent to the model.

Use operating-system permissions, isolated development environments, protected branches, backups, and provider spending limits as additional controls.

## Troubleshooting

### OpenCode cannot find the API key

Confirm the variable is available in the same terminal:

Linux or macOS:

```bash
test -n "$ANTHROPIC_API_KEY" && echo "ANTHROPIC_API_KEY is set"
```

PowerShell:

```powershell
if ($env:ANTHROPIC_API_KEY) {
  "ANTHROPIC_API_KEY is set"
}
```

Do not print the key itself.

### A model is unavailable

Run `/models` and confirm that the model ID appears for the Anthropic provider. Also verify account access and API billing.

### OpenCode does not start in Plan

Run:

```bash
opencode debug config
```

Confirm that the resolved `default_agent` is `plan` and check for a higher-priority managed configuration.

### A safe read-only command asks for approval

The command must match a configured pattern. For example, `rg pattern` matches `rg *`, while an alias, wrapper, pipeline, or differently parsed command may not.

### `AGENTS.md` is missing

Create it in the project root or remove it from:

```jsonc
"instructions": ["AGENTS.md"]
```

## Recommended Repository Layout

```text
your-project/
├── opencode.jsonc
├── AGENTS.md
├── README.md
├── .env.example
└── ...
```

Keep actual secret files untracked and protected by both `.gitignore` and OpenCode permissions.

## References

- OpenCode configuration: https://opencode.ai/docs/config/
- OpenCode agents: https://opencode.ai/docs/agents/
- OpenCode permissions: https://opencode.ai/docs/permissions/
- OpenCode providers: https://opencode.ai/docs/providers/
- OpenCode models: https://opencode.ai/docs/models/
- OpenCode configuration schema: https://opencode.ai/config.json
- Claude Sonnet 5 model notes: https://platform.claude.com/docs/en/about-claude/models/whats-new-sonnet-5
