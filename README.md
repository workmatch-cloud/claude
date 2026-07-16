# Token-Conscious OpenCode Configuration for Anthropic

This project provides a conservative OpenCode configuration focused on controlling API usage, limiting unnecessary agent loops, and requiring approval before potentially expensive or destructive actions.

The configuration is intended for software-development projects where Claude Haiku 4.5 handles planning and file discovery, while Claude Sonnet 5 is reserved for controlled implementation.

## Main Goals

- Use only the Anthropic provider.
- Start every session with a read-only planning agent.
- Use Claude Haiku 4.5 for lower-cost planning and exploration.
- Use Claude Sonnet 5 only for implementation.
- Disable adaptive thinking for routine Sonnet tasks.
- Limit agentic iterations to reduce repeated tool calls.
- Prevent the build agent from creating additional subagent sessions.
- Keep agent responses focused through low temperature and concise prompts.
- Require approval before edits, most shell commands, web searches, and URL fetches.
- Block access to secrets and files outside the active project.
- Disable public session sharing, MCP servers, plugins, and broad-purpose subagents by default.

## Included Files

### `opencode.jsonc`

The main OpenCode configuration. It uses JSONC, so comments are supported.

### `README.md`

This document.

### `AGENTS.md`

The configuration references a project instruction file:

```jsonc
"instructions": ["AGENTS.md"]
```

Create `AGENTS.md` in the project root or remove it from the `instructions` array.

Because instruction files are included in model context, keep `AGENTS.md` concise. Store only stable, project-specific information that the agent would otherwise need to rediscover.

A suitable starting point is:

```md
# Project Instructions

- Follow the existing architecture and naming conventions.
- Make small, focused changes.
- Read only files relevant to the current request.
- Do not modify generated files.
- Run the narrowest relevant verification first.
- Avoid repeated summaries and unrelated improvements.
- Never expose credentials, tokens, or environment-variable values.
```

OpenCode can also create or update this file with:

```text
/init
```

Review the generated content afterward and remove unnecessary details.

## Requirements

- OpenCode installed and available through the `opencode` command.
- An Anthropic API key.
- Anthropic account access to:
  - `claude-haiku-4-5`
  - `claude-sonnet-5`

Inside the OpenCode TUI, use the following command to review available models:

```text
/models
```

Model availability depends on the Anthropic account and the model identifiers supported by the installed OpenCode version.

## Install OpenCode

Use one of the official installation methods below.

### Universal Shell Installer

On Linux, macOS, or WSL:

```bash
curl -fsSL https://opencode.ai/install | bash
```

Restart the terminal if the installer updates `PATH`, then verify the installation:

```bash
opencode --version
```

### npm

With a current Node.js installation:

```bash
npm install -g opencode-ai
```

### Bun

```bash
bun add -g opencode-ai
```

### Homebrew

```bash
brew install anomalyco/tap/opencode
```

### Arch Linux

Using an AUR helper:

```bash
paru -S opencode
```

### Windows

OpenCode can run directly on Windows, but the official documentation recommends WSL for the most consistent terminal, filesystem, and development-tool experience.

Install WSL from an elevated PowerShell terminal:

```powershell
wsl --install
```

Restart Windows if requested, open the installed Linux distribution, and install OpenCode inside WSL:

```bash
curl -fsSL https://opencode.ai/install | bash
opencode --version
```

For the best filesystem performance, keep repositories inside the WSL filesystem:

```bash
mkdir -p ~/code
cd ~/code
git clone <repository-url>
cd <repository-directory>
opencode
```

Projects stored on Windows drives are available through `/mnt`:

```bash
cd /mnt/c/Users/YourName/Documents/project
opencode
```

### Desktop Application

OpenCode Desktop is available from the official download page for Windows, macOS, and Linux.

The terminal configuration in this repository remains useful when running OpenCode from a terminal or when connecting the Desktop application to an OpenCode server.

When exposing a server beyond localhost, set a server password:

```bash
OPENCODE_SERVER_PASSWORD="replace-with-a-strong-password" \
  opencode serve --hostname 0.0.0.0 --port 4096
```

Do not commit the server password or place it in this repository.

### VS Code, Cursor, Windsurf, and VSCodium

Open the IDE's integrated terminal and run:

```bash
opencode
```

OpenCode can install its IDE extension automatically. It can then use the current editor selection and file references as context.

Useful shortcuts on Windows and Linux include:

```text
Ctrl+Esc        Open or focus OpenCode
Ctrl+Shift+Esc  Start a new OpenCode terminal session
Alt+Ctrl+K      Insert a file reference
```

If automatic installation does not work, search for the OpenCode extension in the IDE extension marketplace.

## Install This Configuration

### Project-Level Configuration

Place the files in the project root:

```text
your-project/
├── opencode.jsonc
├── AGENTS.md
└── ...
```

Then start OpenCode from that directory:

```bash
opencode
```

### Global Configuration

To use the configuration globally, place it at:

```text
~/.config/opencode/opencode.jsonc
```

A project-level configuration may override global settings.

### Custom Configuration Path

Set `OPENCODE_CONFIG` to load the file from another location:

```bash
export OPENCODE_CONFIG=/absolute/path/to/opencode.jsonc
opencode
```

PowerShell:

```powershell
$env:OPENCODE_CONFIG = "C:\absolute\path\to\opencode.jsonc"
opencode
```

## Configure the Anthropic API Key

The API key is not stored in `opencode.jsonc`. The provider reads it from:

```text
ANTHROPIC_API_KEY
```

### Linux or macOS

For the current terminal session:

```bash
export ANTHROPIC_API_KEY="your-api-key"
opencode
```

To persist the variable, add it through the appropriate secure shell or operating-system environment configuration.

### Windows PowerShell

For the current PowerShell session:

```powershell
$env:ANTHROPIC_API_KEY = "your-api-key"
opencode
```

To save it for the current Windows user:

```powershell
[Environment]::SetEnvironmentVariable(
  "ANTHROPIC_API_KEY",
  "your-api-key",
  "User"
)
```

Open a new terminal after creating a persistent environment variable.

Do not place the real key in:

- `opencode.jsonc`
- `AGENTS.md`
- `.env.example`
- source control
- screenshots
- logs
- prompts sent to the model

## Validate the Configuration

Run:

```bash
opencode debug config
```

Check the resolved configuration for the following values:

```text
enabled_providers: anthropic
default_agent: plan
share: disabled
```

Also confirm that:

- `plan` uses Claude Haiku 4.5;
- `build` uses Claude Sonnet 5;
- `explore` uses Claude Haiku 4.5;
- the step limits are 4, 8, and 3;
- `general` and `scout` are disabled;
- MCP and plugin lists are empty;
- no higher-priority configuration unexpectedly overrides these settings.

The `$schema` field enables validation and autocomplete in editors that support JSON Schema.

## Recommended Workflow

### 1. Start in Plan

Run:

```bash
opencode
```

The configured default agent is `plan`.

Use it to identify relevant files, understand the current implementation, evaluate risks, and produce a small implementation plan.

Example:

```text
Analyze how authentication is implemented. Inspect only the relevant files and
propose the smallest safe change. Do not modify anything.
```

The `plan` agent:

- uses Claude Haiku 4.5;
- has a temperature of `0.1`;
- cannot edit files;
- cannot run shell commands;
- cannot write task lists;
- cannot search or fetch web content;
- may invoke only the `explore` subagent;
- stops after at most four agentic iterations.

### 2. Review the Plan

Before switching agents, check:

- which files will be changed;
- whether the assumptions are correct;
- which tests or checks are required;
- whether the task can be made smaller;
- whether external research is actually necessary.

### 3. Switch to Build

Use the OpenCode agent selector or the configured TUI shortcut to select `build`.

The `build` agent:

- uses Claude Sonnet 5;
- has a temperature of `0.1`;
- has adaptive thinking disabled;
- asks before editing files;
- asks before most shell commands;
- cannot create task lists;
- cannot invoke any subagent;
- asks before web searches and URL fetches;
- stops after at most eight agentic iterations.

Example:

```text
Implement the approved plan. Make the smallest correct patch, inspect only
relevant files, and briefly verify the result.
```

### 4. Review Approval Requests

Approve only actions required by the current task.

Reject requests that involve:

- unrelated files;
- broad repository scans;
- unnecessary web searches;
- unexpected package installation;
- destructive shell commands;
- access outside the project;
- changes not included in the approved plan.

### 5. Start a New Session for Unrelated Work

Use:

```text
/new
```

when beginning a separate task. Reusing a long session for an unrelated problem can cause old context to be sent again.

Use:

```text
/compact
```

during a long task when the existing conversation still matters but has become too large.

## Agent Overview

| Agent | Type | Model | Temperature | Steps | Subagents |
|---|---|---|---:|---:|---|
| `plan` | Primary | Claude Haiku 4.5 | 0.1 | 4 | `explore` only |
| `build` | Primary | Claude Sonnet 5 | 0.1 | 8 | None |
| `explore` | Subagent | Claude Haiku 4.5 | 0.1 | 3 | None |
| `general` | Disabled | — | — | — | — |
| `scout` | Disabled | — | — | — | — |


## Using This Configuration with Spring Boot

This configuration works best with Spring Boot when each OpenCode session handles one small, clearly defined task.

The recommended workflow is:

```text
Plan with Haiku
→ identify the relevant classes and configuration
→ define the minimum change
→ define the narrowest useful test
→ review the plan
→ switch to Build with Sonnet
→ implement the approved change
→ run focused verification
→ start a new session for the next unrelated task
```

### Spring Boot Project Requirements

Before starting OpenCode, confirm that the project has:

- the JDK version required by `pom.xml`, `build.gradle`, or `build.gradle.kts`;
- the Maven Wrapper (`mvnw` and `mvnw.cmd`) or Gradle Wrapper (`gradlew` and `gradlew.bat`);
- a clean or understood Git working tree;
- local development configuration that does not expose production credentials;
- any required local services, such as a database or message broker, documented separately.

Prefer the project wrapper instead of a globally installed Maven or Gradle version.

### Recommended Project Layout

```text
spring-boot-project/
├── opencode.jsonc
├── AGENTS.md
├── pom.xml
├── mvnw
├── mvnw.cmd
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
│       └── java/
└── target/
```

For Gradle projects, replace `pom.xml` and the Maven wrapper files with the corresponding Gradle files.

Build output such as `target/` or `build/` must not be edited.

### Recommended Spring Boot `AGENTS.md`

Keep this file aligned with the actual project. Do not claim a Java, Spring Boot, Maven, Gradle, database, or testing version that the project does not use.

```md
# Project Instructions

## Technology

- Read the Java and Spring Boot versions from the build file.
- Use the project Maven or Gradle Wrapper.
- Follow the existing package structure and coding style.

## Development Rules

- Make the smallest change required by the task.
- Do not modify unrelated classes or dependencies.
- Do not edit generated files or build output.
- Prefer constructor injection.
- Keep controllers focused on HTTP concerns.
- Keep business logic in services.
- Keep persistence logic in repositories.
- Preserve existing API contracts unless the task explicitly changes them.
- Do not add dependencies unless they are required and approved.
- Do not override Spring Boot managed dependency versions without explaining why.
- Never expose credentials, tokens, connection strings, or environment values.

## Verification

- Run the narrowest relevant test first.
- Use full test suites only after focused verification succeeds.
- Report changed files, commands executed, and checks not completed.
```

The `/init` command can generate an initial `AGENTS.md`, but review and shorten it before normal use.

### Start OpenCode

From the Spring Boot project root:

```bash
opencode
```

The configured `plan` agent starts automatically.

### Plan a Spring Boot Change

The planning prompt should name the exact feature, failure, or migration scope.

Example:

```text
Analyze why UserController returns HTTP 500 when the user is not found.

Inspect only:
- @pom.xml
- @src/main/java/com/example/user/UserController.java
- @src/main/java/com/example/user/UserService.java
- the related exception handler
- the related tests

Do not edit files.

Return:
1. the root cause;
2. the minimum files that must change;
3. the smallest safe implementation plan;
4. the focused test command.
```

For a compilation problem:

```text
Analyze the compilation error involving RestTemplateBuilder.

Inspect only pom.xml, the HTTP client configuration, direct usages, and related
tests. Do not edit files. Identify the exact API change and propose the minimum
migration.
```

The plan should establish the relevant files before switching to `build`, because this configuration prevents the build agent from invoking subagents.

### Implement the Approved Plan

Switch to the `build` agent and use a constrained prompt:

```text
Implement only the approved plan.

Do not modify unrelated classes, dependencies, configuration, or tests.
Do not perform opportunistic refactoring.
Run the focused verification first.
Show the final diff and briefly explain each changed file.
```

The build agent asks before edits and most shell commands. Review each approval request instead of approving broad command patterns automatically.

### Maven Verification

Linux, macOS, or WSL:

```bash
# Compile production code without running tests.
./mvnw -DskipTests compile

# Run one test class.
./mvnw -Dtest=UserServiceTest test

# Run one test method.
./mvnw -Dtest=UserServiceTest#shouldReturnUser test

# Run the normal test suite.
./mvnw test

# Run packaging and integration-test lifecycle checks when required.
./mvnw verify
```

Windows PowerShell:

```powershell
# Compile production code without running tests.
.\mvnw.cmd -DskipTests compile

# Run one test class.
.\mvnw.cmd -Dtest=UserServiceTest test

# Run one test method.
.\mvnw.cmd "-Dtest=UserServiceTest#shouldReturnUser" test

# Run the normal test suite.
.\mvnw.cmd test

# Run packaging and integration-test lifecycle checks when required.
.\mvnw.cmd verify
```

Start the application with a local Spring profile:

Linux, macOS, or WSL:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```

Windows PowerShell:

```powershell
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=local"
```

Do not start production profiles or connect to production services unless that action is explicitly required and approved.

### Gradle Verification

Linux, macOS, or WSL:

```bash
./gradlew test
./gradlew test --tests "com.example.user.UserServiceTest"
./gradlew test --tests "com.example.user.UserServiceTest.shouldReturnUser"
./gradlew build
./gradlew bootRun
```

Windows PowerShell:

```powershell
.\gradlew.bat test
.\gradlew.bat test --tests "com.example.user.UserServiceTest"
.\gradlew.bat build
.\gradlew.bat bootRun
```

### Recommended Verification Order

Use the narrowest check that can detect the expected failure:

```text
1. Compile the affected code.
2. Run the specific test method or class.
3. Run tests for the affected module.
4. Run the complete unit-test suite.
5. Run verify or the full build only when integration or packaging checks matter.
```

This order reduces unnecessary command output and model context while still providing useful confidence.

### Referencing Files Directly

When the relevant file is known, reference it directly instead of requesting a repository-wide search:

```text
Explain the validation flow in
@src/main/java/com/example/api/UserController.java.
```

For a multi-file change:

```text
Analyze only:
@pom.xml
@src/main/java/com/example/config/RestClientConfiguration.java
@src/test/java/com/example/config/RestClientConfigurationTest.java
```

Direct references reduce exploratory tool calls and make the requested scope explicit.

### Spring Boot Upgrade Workflow

Do not handle a major Spring Boot upgrade as one unrestricted task. Split it into separate sessions:

1. build file, parent, plugins, and dependency management;
2. compilation errors;
3. HTTP client configuration;
4. Spring Security;
5. persistence and database migrations;
6. messaging and external integrations;
7. test framework and test configuration;
8. configuration-property changes;
9. full verification and packaging.

Example planning request:

```text
Analyze the Spring Boot upgrade in this session only for RestTemplateBuilder
compilation failures.

Do not analyze Spring Security, persistence, Spring Cloud, or unrelated
dependencies. Inspect the build file, the affected configuration classes,
direct usages, and related tests. Do not edit files.
```

After the plan is approved:

```text
Implement only the approved RestTemplateBuilder migration.
Do not update unrelated dependencies.
Run the smallest relevant test and show the final diff.
```

Start a new session for the next migration category so old errors, logs, and diffs are not repeatedly included in context.

### Spring Boot Tasks That Fit This Workflow

Good single-session tasks include:

- fix one compilation error;
- add or repair one REST endpoint;
- adjust validation for one request model;
- add one service method and its focused tests;
- fix one repository query;
- migrate one deprecated Spring API;
- update one configuration-properties class;
- diagnose one failing test class;
- review one security rule;
- update one dependency only when required by the task.

Avoid prompts such as:

```text
Review the whole application and improve everything.
```

Prefer:

```text
Fix the null-handling behavior in UserService#getUser.
Inspect only the service, its direct collaborators, and its tests.
Do not refactor unrelated code.
```

### Protecting Spring Boot Secrets

Common sensitive values include:

- database passwords;
- OAuth client secrets;
- JWT signing keys;
- cloud credentials;
- SMTP passwords;
- private repository tokens;
- production URLs containing credentials.

Keep them outside source control and outside OpenCode prompts. Use environment variables, secret managers, or local files denied by the configuration.

A safe example file may document variable names without real values:

```properties
SPRING_DATASOURCE_URL=
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=
```

Do not ask OpenCode to print the resolved Spring environment or dump all environment variables.


## How Token Consumption Is Reduced

### Lower-Cost Default Model

The global `model` and `small_model` both use Claude Haiku 4.5:

```jsonc
"model": "anthropic/claude-haiku-4-5",
"small_model": "anthropic/claude-haiku-4-5"
```

Claude Sonnet 5 is selected only by the `build` agent.

### Reduced Agent Step Limits

The configuration sets:

```text
Plan:    4 steps
Build:   8 steps
Explore: 3 steps
```

The `steps` option limits the number of agentic iterations. When the limit is reached, OpenCode instructs the agent to stop using tools and return a textual result.

A step limit controls iterations, not tokens or currency directly. A single step can still consume substantial input if large files or long tool results are included.

### No Subagents During Build

The build agent contains:

```jsonc
"task": "deny"
```

This prevents Sonnet from opening additional subagent sessions while implementing a change.

The planning agent can still use the lower-cost `explore` subagent when file discovery is necessary.

### Focused Agent Prompts

Each active agent receives instructions to:

- inspect only relevant files;
- avoid broad repository scans;
- avoid repeated summaries;
- avoid speculative improvements;
- stop when the requested work is complete.

This does not create a hard token limit, but it reduces unnecessary context collection and verbose output.

### Low Temperature

All active agents use:

```jsonc
"temperature": 0.1
```

A low temperature makes responses more focused and deterministic. It is primarily a behavior setting rather than a direct token cap.

### Adaptive Thinking Disabled

Claude Sonnet 5 is configured with:

```jsonc
"thinking": {
  "type": "disabled"
}
```

This is applied in the provider model options and explicitly on the `build` agent.

Disabling adaptive thinking can reduce reasoning-token usage for routine changes. For difficult architecture, debugging, or multi-file work, it may reduce solution quality.

For a difficult task, temporarily enable adaptive thinking only when needed:

```jsonc
"thinking": {
  "type": "adaptive"
}
```

Revert it afterward if routine cost control is the priority.

### Automatic Context Compaction

The configuration uses:

```jsonc
"compaction": {
  "auto": true,
  "prune": true,
  "reserved": 12000
}
```

This:

- automatically compacts long sessions;
- removes old tool outputs from active context;
- reserves context space for the compaction process.

`reserved` is not a spending limit and does not reserve billable tokens in advance. It protects enough context capacity for OpenCode to compact the session safely.

### Task-List Tool Disabled

The active agents deny `todowrite`:

```jsonc
"todowrite": "deny"
```

This avoids extra tool calls for internal task-list maintenance. It is useful for small and medium changes, but complex work may become less structured.

### Web Access Restricted

The `plan` and `explore` agents deny external research:

```jsonc
"websearch": "deny",
"webfetch": "deny"
```

The `build` agent requires approval:

```jsonc
"websearch": "ask",
"webfetch": "ask"
```

This prevents automatic network calls and the large result payloads they may add to the context.

### MCP Servers and Plugins Disabled

The default configuration loads no MCP servers or plugins:

```jsonc
"mcp": {},
"plugin": []
```

External integrations can add tool descriptions and tool results to the context. Enable them only for projects that need them.

### Watcher Exclusions

The watcher ignores dependencies, generated output, logs, source maps, minified JavaScript, lock files, environment files, and secret directories.

This reduces irrelevant filesystem activity, but watcher exclusions are not a direct token budget and do not replace file-read permissions.

### Prompt Cache Key

The Anthropic provider uses:

```jsonc
"setCacheKey": true
```

This keeps a consistent cache key where supported. Actual cache savings depend on the provider, request structure, repeated prompt prefixes, and cache eligibility.

## Permission Behavior

OpenCode permissions resolve to:

- `allow`: run without approval;
- `ask`: request user approval;
- `deny`: block the action.

Agent-specific permissions override the global configuration.

For pattern-based rules, the last matching rule takes precedence. This is why broad rules appear before more specific rules.

## Global File Restrictions

Normal project files are readable, but these patterns are denied:

```text
*.env
*.env.*
**/secrets/**
**/credentials*
```

This pattern is explicitly allowed afterward:

```text
*.env.example
```

As the more specific matching rule appears later, example environment files remain readable while real environment files remain blocked.

Access outside the active project is denied:

```jsonc
"external_directory": "deny"
```

## Build Shell Rules

The build agent asks before shell commands by default:

```jsonc
"bash": {
  "*": "ask"
}
```

The following read-only patterns are automatically allowed:

```text
git status...
git diff...
git log...
git show...
rg ...
grep ...
```

The following patterns are denied:

```text
git push...
rm ...
```

A command not explicitly allowed or denied still requires approval through the catch-all `"*": "ask"` rule.

The denied list is intentionally not presented as a complete shell sandbox. A user can still approve another dangerous command. Review every approval request.

## File Editing

The build agent uses:

```jsonc
"edit": "ask"
```

This requires approval before file writes, edits, or patches.

The plan and explore agents use:

```jsonc
"edit": "deny"
```

They cannot modify files.

## Language Server Support

The configuration enables:

```jsonc
"lsp": true
```

Language-server tools can help locate symbols, definitions, references, and diagnostics without reading entire files manually.

Actual availability depends on the programming language and installed language-server support.

## Session Sharing

Public session sharing is disabled:

```jsonc
"share": "disabled"
```

The `/share` command cannot publish the session while this setting is active.

## Security and Cost Limitations

This configuration reduces accidental operations and unnecessary model usage. It is not a complete security sandbox or a hard spending cap.

Important limitations:

- A user can approve a costly or dangerous command.
- Step limits do not cap input tokens, output tokens, or currency directly.
- Large source files can consume substantial context in one operation.
- Long prompts and instruction files are sent as model context.
- Project, global, or managed configuration layers may override values.
- Model identifiers and provider behavior can change.
- Watcher exclusions do not automatically deny reading a file.
- A secret already pasted into a prompt can still be sent to the provider.
- Disabling subagents and task lists may reduce performance on complex work.
- Disabling adaptive thinking may reduce quality on difficult reasoning tasks.

Use additional controls where appropriate:

- Anthropic spending limits and usage alerts;
- protected branches;
- source-control review;
- operating-system permissions;
- isolated development environments;
- backups;
- secret scanning;
- short, task-specific OpenCode sessions.

## Practical Cost-Control Guidelines

Use prompts that identify the exact task and relevant area:

```text
Fix the null handling in src/services/user-service.ts.
Do not change unrelated code. Run only the relevant test.
```

Avoid vague prompts such as:

```text
Review the whole repository and improve everything.
```

Prefer direct file references when the relevant file is known:

```text
Explain the validation flow in @src/api/user-controller.ts.
```

Start a new session for an unrelated task instead of carrying the previous context forward.

Keep `AGENTS.md` and custom prompts short. Avoid placing large coding guides directly in always-loaded instruction files.

Do not enable MCP servers, plugins, web access, or additional agents unless the task requires them.

Use the `plan` agent for analysis. Switch to `build` only after the scope is clear.

## Troubleshooting

### OpenCode Cannot Find the API Key

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

### A Model Is Unavailable

Run:

```text
/models
```

Confirm that the configured model identifiers are available for the Anthropic provider and account.

### OpenCode Does Not Start in Plan

Run:

```bash
opencode debug config
```

Confirm that the resolved value is:

```jsonc
"default_agent": "plan"
```

The default agent must exist and use `mode: "primary"`.

### Build Cannot Use Explore

This is intentional. The build agent uses:

```jsonc
"task": "deny"
```

Return to `plan` for additional repository exploration, then switch back to `build` with a narrower implementation request.

### Plan Cannot Search the Web

This is intentional. Both external tools are denied in `plan`:

```jsonc
"websearch": "deny",
"webfetch": "deny"
```

Switch to `build` only when external research is genuinely needed, then review the approval request.

### An Allowed Shell Command Still Asks for Approval

The exact command must match a configured pattern. Aliases, pipelines, wrappers, shell operators, or a different parsed command may not match.

### The Agent Stops Before Finishing

The configured step limit may have been reached.

Possible responses:

- provide a smaller, more focused request;
- start a new session for a distinct phase;
- temporarily increase the relevant agent's `steps`;
- use `/compact` if the session context is large.

### `AGENTS.md` Is Missing

Create it manually, run `/init`, or remove it from:

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

Keep real secret files untracked and protected by both source-control ignore rules and OpenCode permissions.

## References

- OpenCode download: https://opencode.ai/download
- OpenCode Windows and WSL: https://opencode.ai/docs/windows-wsl/
- OpenCode IDE integration: https://opencode.ai/docs/ide/
- OpenCode configuration: https://opencode.ai/docs/config/
- OpenCode agents: https://opencode.ai/docs/agents/
- OpenCode permissions: https://opencode.ai/docs/permissions/
- OpenCode rules and `AGENTS.md`: https://opencode.ai/docs/rules/
- OpenCode TUI commands: https://opencode.ai/docs/tui/
- OpenCode providers: https://opencode.ai/docs/providers/
- OpenCode models: https://opencode.ai/docs/models/
- OpenCode configuration schema: https://opencode.ai/config.json
- Claude model documentation: https://platform.claude.com/docs/
