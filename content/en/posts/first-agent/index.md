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


## Blabber

#### Thanks! Keep calm and code on! 🚀