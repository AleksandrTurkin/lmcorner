+++
date = '2026-07-30T00:25:35+02:00'
draft = true
title = 'Copilot CLI vs Gemini CLI: Command Comparison'
tags = ["aiTools"]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

This reference compares the fixed, built-in interactive commands documented for Copilot CLI and Gemini CLI as of July 30, 2026. It covers slash commands plus documented `@` and `!` forms. Startup options, environment variables, keyboard-only shortcuts, and user-defined commands are outside the table.

`X` means that the other CLI has no direct documented counterpart. Near matches state the difference instead of treating similar names as identical commands.

Sources: [GitHub Copilot CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference) and [Gemini CLI commands](https://geminicli.com/docs/reference/commands/).

| Copilot CLI Command | Gemini CLI Command | Short description |
| --- | --- | --- |
| `/model [--repo\|--local\|--session] [MODEL]`<br>`/models` | `/model`<br>`[manage\|set <model> [--persist]]` | Select and configure the AI model. Copilot also changes reasoning effort and context settings. |
| `/plan [PROMPT]` | `/plan [copy]` | Use read-only planning before implementation; Gemini can copy the approved plan. |
| `/compact [FOCUS-INSTRUCTIONS]` | `/compress` | Summarize conversation context to free context-window capacity. |
| `/context` | `X` | Show Copilot's context-window token usage and visualization. |
| `/ask QUESTION` | `X` | Ask Copilot a side question without adding it to conversation history. |
| `/refine TEXT` | `X` | Rewrite a rough prompt into a clearer prompt for review. |
| `/resume [SESSION-ID]`<br>`/continue [SESSION-ID]` | `/resume`<br>`/chat` | Browse or reopen a saved session; Gemini's `/chat` is an alias and also manages checkpoints. |
| `/session`, `/sessions` | `/resume`<br>`[list\|save\|resume\|delete\|share]` | Inspect, rename, clean up, or delete sessions and checkpoints. |
| `/clear [PROMPT]`<br>`/new [PROMPT]`, `/reset [PROMPT]` | `X` | Start a new Copilot conversation; this is not Gemini's screen-clearing command. |
| `X` | `/clear` | Clear Gemini's visible terminal history and scrollback without necessarily resetting its saved session. |
| `/share`, `/export` | `/resume share [FILENAME]` | Export or share the current conversation. Copilot supports links, Markdown, HTML, and gists; Gemini exports Markdown or JSON. |
| `/copy` | `/copy` | Copy the last model response to the clipboard. |
| `/undo` | `/restore [TOOL_CALL_ID]` | Revert changes: Copilot rewinds the last turn, while Gemini restores a checkpoint before a tool call. |
| `/rewind` | `/rewind` | Navigate back through conversation history and optionally revert associated file changes. |
| `/review [PROMPT]` | `X` | Run Copilot's code-review agent against local changes. |
| `/security-review [PROMPT]` | `X` | Run Copilot's focused security-review agent against active local changes. |
| `/diff` | `X` | Review the current working-tree or branch diff in Copilot's diff interface. |
| `/sandbox [enable\|disable]` | `X` | Configure Copilot's OS-level sandbox for shell, MCP, LSP, and built-in tools. |
| `/permissions [show\|reset]`<br>`/reset-allowed-tools` | `/permissions`<br>`trust [DIRECTORY]` | Inspect or reset Copilot's in-session approvals; Gemini manages folder trust. |
| `/allow-all [on\|off\|show]`, `/yolo [on\|off\|show]` | `X` | Enable or inspect Copilot's all-permissions mode. |
| `/mcp`<br>`[list\|show\|add\|edit\|delete]`<br>`[disable\|enable\|auth\|reload\|search]` | `/mcp`<br>`[auth\|desc\|disable\|enable]`<br>`[list\|reload\|schema]` | Manage MCP servers. Gemini additionally exposes descriptions and schemas; Copilot can add and edit servers interactively. |
| `/add-dir PATH`, `/list-dirs` | `/directory [add\|show]`, `/dir` | Allow or inspect additional workspace directories. |
| `/cwd`, `/cd [PATH]` | `X` | Show or change Copilot's session working directory. Gemini's directory command adds workspace roots instead. |
| `/env` | `/tools [desc\|nodesc]` | Inspect loaded Copilot environment resources or available Gemini tools; these are related diagnostics, not identical output. |
| `/lsp`<br>`[show\|test\|reload\|logs\|help]`<br>`[SERVER-NAME]` | `X` | Manage Copilot language-server configuration and logs. |
| `/ide` | `/ide`<br>`[enable\|disable\|install\|status]` | Connect to or manage IDE integration. Gemini exposes explicit lifecycle subcommands. |
| `/init` | `/init` | Analyze the project and generate repository instructions: Copilot instructions or `GEMINI.md`. |
| `/instructions` | `/memory`<br>`[list\|refresh\|show]` | Inspect loaded Copilot instruction files or Gemini's hierarchical `GEMINI.md` memory. |
| `/agent` | `/agents`<br>`[list\|reload\|enable\|disable\|config]` | Select a Copilot agent or discover, configure, and enable Gemini subagents. |
| `/subagents`, `/agents` | `/agents config <agent-name>` | Configure default or per-agent models and limits. Copilot's command focuses on subagent model settings. |
| `/fleet [PROMPT]` | `X` | Run parts of a Copilot task with parallel subagents. |
| `/tasks` | `/shells`, `/bashes` | View and manage Copilot subagent and shell tasks or Gemini background shells. |
| `/delegate [PROMPT]` | `X` | Delegate repository changes to Copilot coding agent and receive an AI-generated pull request. |
| `/rubber-duck [PROMPT]` | `X` | Ask Copilot's complementary-model agent for a second opinion on plans, code, or tests. |
| `/research TOPIC` | `X` | Run Copilot deep research using GitHub Search and web sources. |
| `/pr [view\|create\|fix\|auto\|automerge]` | `X` | Manage the current branch's pull request with Copilot. |
| `X` | `/setup-github` | Set up GitHub Actions to triage issues and review pull requests with Gemini. |
| `/plugins`, `/plugin`<br>`/extensions`, `/extension` | `/extensions`<br>`[config\|disable\|enable\|explore\|install]`<br>`[link\|list\|restart\|uninstall\|update]` | Manage extensions and plugin ecosystems. Copilot also manages plugins, marketplaces, MCP servers, and skills from its dashboard. |
| `/skills [list\|info\|add\|remove\|reload]` | `/skills [disable\|enable\|list\|reload]` | Discover, enable, and manage reusable agent skills. |
| `X` | `/commands [list\|reload]` | List or reload Gemini custom slash-command definitions. |
| `/settings`, `/config` | `/settings` | View and change CLI settings. Copilot can target user, repository, or local scopes inline. |
| `/theme`<br>`[default\|github\|dim]`<br>`[high-contrast\|colorblind]` | `/theme` | View or change the terminal color theme. |
| `/statusline`, `/footer` | `X` | Configure the Copilot status line or footer. |
| `/terminal-setup` | `/terminal-setup` | Configure terminal keybindings for multiline input. |
| `/voice [on\|off\|models\|devices]` | `X` | Enable voice mode or select a Copilot voice model and microphone. |
| `X` | `/vim` | Enable or disable Gemini's Vim-style input mode. |
| `/keep-alive`, `/caffeinate` | `X` | Keep the machine awake while a Copilot session is active or busy. |
| `/experimental [on\|off\|show]` | `X` | View or toggle Copilot experimental features. |
| `/after [DELAY PROMPT]` | `X` | Schedule one delayed prompt, skill, or schedulable Copilot command in an experimental session. |
| `/every [INTERVAL PROMPT]` | `X` | Schedule a recurring prompt, skill, or schedulable Copilot command in an experimental session. |
| `/fork [NAME]`, `/branch [NAME]` | `X` | Fork the current Copilot session into a separate experimental session. |
| `/worktree [branch\|task]` | `X` | Create and switch to an isolated Git worktree from Copilot. |
| `/move [branch\|task]` | `X` | Move uncommitted changes into a new Copilot Git worktree. |
| `/remote [on\|off]` | `X` | Enable or inspect remote steering for a Copilot session. |
| `/rename [NAME]` | `X` | Rename the current Copilot session. |
| `/restart` | `X` | Restart Copilot CLI while preserving the current session. |
| `/search [QUERY]`, `/find [QUERY]` | `X` | Search the Copilot conversation timeline; available in experimental mode. |
| `/changelog`, `/release-notes` | `X` | View Copilot CLI release notes, optionally with an AI summary. |
| `/downgrade VERSION` | `X` | Restart Copilot at a selected older CLI version; available for team accounts. |
| `/update`, `/upgrade` | `/upgrade` | Update Copilot CLI or open Gemini's Code Assist upgrade page. |
| `/usage` | `/stats [session\|model\|tools]` | Display usage and statistics. Gemini splits session, model, and tool views. |
| `/limits [set\|unset]` | `X` | View or set Copilot's per-response AI-credit limits. |
| `/version` | `/about` | Show CLI version and product information; Copilot also checks for updates. |
| `/user [show\|list\|switch]` | `/auth` | Inspect or switch the Copilot GitHub account; Gemini's authentication dialog changes sign-in method. |
| `/login`, `/logout` | `/auth` | Sign in or out of Copilot, or open Gemini's authentication-method dialog. |
| `/feedback`, `/bug` | `/bug [TITLE]` | Send feedback or file a CLI bug report. Gemini opens an issue in its GitHub repository by default. |
| `/app` | `X` | Launch the GitHub Copilot app or show its download URL. |
| `/clikit [COMPONENT]`, `/tuikit [COMPONENT]` | `X` | Preview Copilot CLI business or design-system components. |
| `X` | `/docs` | Open the Gemini CLI documentation in a browser. |
| `X` | `/editor` | Select a supported editor for Gemini CLI. |
| `X` | `/hooks`<br>`[disable-all\|disable\|enable-all\|enable\|list]` | Manage Gemini lifecycle hooks. |
| `X` | `/policies [list]` | List active Gemini policies by mode. |
| `X` | `/privacy` | View Gemini's privacy notice and choose data-collection consent. |
| `/help`, `?` | `/help`, `/?` | Show interactive-command help. |
| `/exit`, `/quit` | `/exit`, `/quit [--delete]` | Exit the CLI. Gemini can additionally delete the current session history and temporary files. |
| `@ FILENAME` | `@<path_to_<wbr>file_or_directory>` | Add file or directory content to the next prompt. Gemini supports Git-aware directory expansion. |
| `X` | `@` | Send a lone at sign through Gemini unchanged. |
| `! COMMAND` | `!<shell_command>` | Run one shell command without sending it to the model. |
| `!` | `!` | Enter shell mode for multiple shell commands, then return to the CLI. |

#### Thanks! Keep calm and code on! 🚀