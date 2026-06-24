+++
date = '2026-05-12T01:36:50+02:00'
title = 'Agentic Coding, Part 1: API & Ecosystem'
tags = ["agent-coding", "jardi-tips"]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

### Overview

* **IDE:** Visual Studio 2026
* **Ecosystem:** .NET 10 
* **Framework:** ASP.NET Core Web API
* **Architecture:** Clean Architecture + Vertical Slice Architecture

- [api-features.agent.md](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/agents/api-features.agent.md) - An AI agent for generating {{< term "CRUD" >}} operations
- [db-entity-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/db-entity-creator/SKILL.md) - Skill for creating database entities and migration steps
- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - Skill for creating command and query handlers
- [api-endpoint-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md) - Skill for creating API endpoints

**Agent Skill Boundaries**
```
Solution/
├── src/
│   ├── MyProject.Domain/          # 1. Skill: db-entity-creator
│   ├── MyProject.Application/     # 2. Skill: command-query-handler-creator
│   ├── MyProject.Infrastructure/  # 3. Skill: db-entity-creator
│   └── MyProject.WebApi/          # 4. Skill: api-endpoint-creator
```

## Implementation Details

### Project Structure

The classic `Clean Architecture` was chosen for the project structure. It is important that each layer has clearly defined boundaries and dependencies, making it much easier for the AI agent to generate code while maintaining architectural integrity.

```
Solution/
├── src/
│   ├── MyProject.Domain/          # 1. Core entities and rules (No dependencies)
│   ├── MyProject.Application/     # 2. Application use cases (Depends on Domain)
│   ├── MyProject.Infrastructure/  # 3. Infrastructure implementation (Depends on Application)
│   └── MyProject.WebApi/          # 4. Entry point / Host (Depends on Infrastructure and Application)
```

### AI Agent Responsibilities

Currently, the [**AI agent**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/agents/api-features.agent.md) automates the implementation of new {{< term "CRUD" >}} operations. The workflow can be briefly broken down into the following steps:
1. Creating database entities in the `Domain` layer and generating migration steps in the `Infrastructure` layer.
2. Creating command and query handlers in the `Application` layer to manage these entities.
3. Creating endpoints in the `WebApi` layer to provide an API for these operations.

Each of these steps is implemented as a separate skill, and the AI agent applies them sequentially.

1. [**db-entity-creator**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/db-entity-creator/SKILL.md) - Skill for creating database entities and migration steps within the `Domain` and `Infrastructure` layers.
```
│   ├── MyProject.Domain/
│   │   └── Entities/              # Database entities
│   └── MyProject.Infrastructure/
│       ├── Configurations/        # Database entity configurations
│       └── Migrations/            # Database migration steps
```

2. [**command-query-handler-creator**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - Skill for creating command and query handlers in the `Application` layer.
```
│   ├── MyProject.Application/
│   │   └── Features/                # Functional features
│   │       └── <Entity>/            # Target entity for handlers
│   │           ├── Create<Entity>CommandHandler.cs        # Create command handler
│   │           ├── Update<Entity>CommandHandler.cs        # Update command handler
│   │           ├── Delete<Entity>CommandHandler.cs        # Delete command handler
│   │           ├── Get<Entity>ByIdQueryHandler.cs         # GetById query handler
│   │           └── Get<Entities>QueryHandler.cs           # GetAll query handler
```

3. [**api-endpoint-creator**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md) - Skill for creating API endpoints in the `WebApi` layer.
```
│   ├── MyProject.WebApi/
│   │   └── Endpoints/                 # Minimal API endpoints
│   │       └── <Entity>Endpoints.cs    # Endpoints for managing the entity
```

> Pay attention to the generic [EndpointMapExtensions](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/EndpointMapExtensions.cs) helper used for registering new endpoints. This extension helps standardize the `api-endpoint-creator` skill, ensuring all endpoints follow the exact same template.

### AI Agent Ecosystem

1. [**copilot-instructions.md**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/copilot-instructions.md) - This file contains information about the project structure, architecture, libraries used, and basic guidelines for agents. It provides shared context for every prompt. It is crucial to maintain a balance between detail and conciseness here.

2. [**Microsoft Learn MCP Server**](https://github.com/microsoftdocs/mcp) - An MCP server designed to solve the problem of outdated LLM knowledge.  It provides the AI agent with direct access to the up-to-date `Microsoft Learn` database. For example, the release date of .NET 10 is November 2025, while the model may have been trained earlier and objectively does not have information about the new framework and newer language features. The MCP server is installed as part of your local infrastructure and is not tied to the repository.
![MCP](mcp_vs.png)

3. [**api-features.agent.md**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/agents/api-features.agent.md) - A custom AI agent whose main task is to analyze requirements and implement new API features. It uses the skills mentioned above to seamlessly generate code across all layers:
    - [**db-entity-creator**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/db-entity-creator/SKILL.md)
    - [**command-query-handler-creator**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md)
    - [**api-endpoint-creator**](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md)

## Blabber

In this series of articles, I will try to outline my vision of a proper ecosystem for AI agents to produce predictable results at the output. This is not pure "vibe coding" as many understand it today. It's a hybrid approach where the AI agent writes code within very strict guardrails, while the meatbag defines the architecture and the templates to follow. I want a maintainable application, not a throwaway prototype. Every AI agent run must yield a predictable result. For me, this is also an experiment: to see how fast and effective development can be when driven by a solo, slightly aging programmer 😁.

When implementing the core project structure, finding the right balance was critical. On the one hand, you want to unify all operations so that adding a new CRUD feature is as concise as possible—literally a couple of lines of code. Actually, everything can be implemented via generics, from endpoint mapping to generic handlers. This approach saves tons of boilerplate lines! Aaaaand... unfortunately, it makes reviewing this machinery an absolute nightmare. If something changes in there, you usually need a strong drink just to make sense of it.

On the other hand, why are we trying to save lines of code anyway? The AI agent can generate them for us, and our job is to review the result. We don't need complex, over-engineered mechanics; we need code that is easy to read and understand, even if there's a bit more of it. This is one of the reasons why there is no MediatR or AutoMapper in this project. The fewer magic tricks and hidden abstractions we have, the easier it is for the AI agent to write code and for a human to understand it during review.

*Important note:* The skill responsible for adding database entities **does not run migrations**. I’m not ready to trust an AI agent with automatic DB migrations yet. Reckless bravery is not the path we're taking 😁.

*— Why `Clean Architecture`?*

AI agents perform much better with architectures that offer:
* Small context
* Predictable structure
* High locality of changes
* Explicit dependencies
* Minimal "magic"

An AI agent needs more than prompts - it needs an ecosystem: architecture, instructions, skills, up-to-date documentation, and clearly defined boundaries. The less freedom the agent has in technical decisions, the higher the chance of getting predictable code 😇

#### Thanks! Keep calm and code on! 🚀