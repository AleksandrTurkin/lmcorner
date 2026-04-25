+++
date = '2026-04-16T00:01:24+02:00'
draft = true
title = 'News Agent'
tags = ["aiAgent", "aiTools"]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

### Overview

* **Tool:** GitHub Copilot CLI
* **OS**: Windows

```
/// Install GitHub Copilot CLI
1. Win + R -> winget install GitHub.Copilot
2. Terminal (CMD) -> copilot update
3. CMD -> copilot -v

/// Check installation path
1. CMD -> where copilot
```

```
/// News Agent structure
.github/
├── copilot-instructions.md     <-- System prompts & project context
├── agents/
│   ├── news-agent.md           <-- News Agent profile
└── skills/
    ├── get-news-history        <-- Skill: Fetch news history
    ├── get-page-content        <-- Skill: Retrieve web page content
    └── create-news-page        <-- Skill: Generate news page
```

```
/// Running the agent
copilot --autopilot --agent=news-agent --allow-all --add-dir='<Blog directory>' --model=gpt-5.4 --no-ask-user
``` 

- [copilot-instructions.md]()
- [news-agent.md]()
- [get-news-history]()
- [get-page-content]()
- [create-news-page]()

## Step-by-Step Implementation

### GitHub Copilot CLI Installation
1. Press `Win + R` and enter `winget install GitHub.Copilot`.
2. After installation, open the terminal (CMD) and run `copilot update` to ensure you have the latest version. Winget may not always provide the most up-to-date release 😕.
3. Verify the installed version of Copilot CLI using `copilot -v`.
4. To locate the installation path, use the command `where copilot`.
5. Open the terminal and run `copilot` to start.
![Copilot CLI](copilotCli_1.png)

*Note: Copilot CLI may require you to sign in to your GitHub account during the first run.*

Official Copilot CLI documentation: https://docs.github.com/copilot/how-tos/copilot-cli

### News Agent

This custom News Agent is designed to automatically aggregate AI-related software release news from GitHub Copilot CLI, Visual Studio Code, and Visual Studio 2022/2026, and generate news pages (**trust, but verify** — auto-commits are disabled for now 😇). The agent filters updates to include only those relevant to AI functionality.

The News Agent and its **skills** are organized as follows:
```
.github/
├── copilot-instructions.md     <-- System prompts & project context
├── agents/
│   ├── news-agent.md           <-- News Agent profile
└── skills/
    ├── get-news-history        <-- Skill: Fetch news history
    ├── get-page-content        <-- Skill: Retrieve web page content
    └── create-news-page        <-- Skill: Generate news page
```

- [**copilot-instructions.md**]() — This file contains the overall project description and guidelines for Copilot. It defines the Hugo stack and version, project structure, localization setup, and important constraints. This ensures that Copilot remains consistent with the project's style when making changes. In my opinion, finding the right balance here is crucial, as Copilot uses this file as context for every prompt, directly impacting token usage.

- [**news-agent.md**]() — This file describes the agent itself, its available skills, information sources, and workflow. It provides detailed instructions on what the agent should do, which skills to use and in what order, which sources to check, how to filter information, and how to generate news pages.

- [**get-news-history**]() — Designed to retrieve the history of already published news. The logic is straightforward: it fetches the last five items and extracts application names and versions for comparison with fresh releases.

- [**get-page-content**]() — Responsible for fetching web page content and converting it into Markdown. This allows the agent to analyze release notes and extract information about new AI features efficiently.

- [**create-news-page**]() — Handles the generation of news pages using predefined Hugo commands, depending on the target language (Russian or English).

### Running the Agent

```
copilot --autopilot --agent=news-agent --allow-all --allow-all-urls --add-dir='<Blog directory>' --model=gpt-5.4 --no-ask-user
```

- `--autopilot` — Enables the agent to operate autonomously without requiring confirmation for each step.
- `--agent=news-agent` — Specifies that we want to run our News Agent.
- `--allow-all` — Grants the agent permission to use all available tools.
- `--add-dir='<Blog directory>'` — Adds the blog directory to the agent's context, allowing it to create and modify files therein.
- `--model=gpt-5.4` — Specifies the use of the GPT-5.4 model.
- `--no-ask-user` — Disables user prompts, allowing the agent to run completely unattended
- ! A complete list of available options can be found [HERE](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference)

### Sources
- https://docs.github.com/en/copilot/concepts/agents/copilot-cli/about-custom-agents#agent-profile-format
- https://docs.github.com/en/copilot/reference/custom-agents-configuration
- https://code.visualstudio.com/docs/copilot/customization/custom-agents
- https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/add-skills
- https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli-agents/delegate-tasks-to-cca
- https://docs.github.com/en/copilot/how-tos/copilot-cli/allowing-tools
- https://docs.github.com/en/copilot/concepts/agents/copilot-cli/autopilot
- https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference#command-line-options

## Blabber

Well, well, well... It's one of those `//todo` pain points I mentioned earlier: I just can't keep up with all these AI-related updates manually 😅.
The News Agent was created for a couple of reasons: first, to help me (and hopefully you) stay on track with minimal effort; and second, because I wanted to test a custom agentic workflow using skills and autopilot mode.
Why is the "Agents + skills" combination so important? Because it allows you to break down complex tasks into smaller, reusable components (skills) that are easier to maintain and update. You can build a skill with explicit logic or even embedded scripts. This provides the most crucial thing, in my opinion — **predictability**. As you know, the same prompt can yield different results on every run.

On the other hand, a skill can exist in the project without being actively used by the agent. The agent triggers a skill only when necessary, ensuring it doesn't bloat the context with irrelevant information. This is one of the most effective token-saving optimizations.

The News Agent is definitely not perfect yet. There are various minor issues and edge cases... For instance, I want to set it up on a schedule. Even in autopilot mode, I still receive requests to confirm access to external web resources, and I'm aiming for 100% autonomy. And last but not least, I’m still committing new pages manually for now. Whether I add an auto-deploy skill later depends entirely on the quality of the agent's output—I need to be sure I can trust it 🤞.

#### Thanks! Keep calm and code on! 🚀