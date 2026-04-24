---
name: get-page-content
description: Fetch a webpage by URL and return a Markdown version of its readable content using built-in Windows tools.
allowed-tools: shell
---

## Overview
Use shell commands with built-in Windows PowerShell to retrieve content from a single HTTP or HTTPS URL and return a readable Markdown version.

## Behavior
1. Accept a single absolute HTTP or HTTPS URL as input.
2. Use only shell with built-in Windows/PowerShell capabilities.
3. Retrieve the response content directly in shell.
4. Extract readable content from the response.
5. Return the result as Markdown, preserving headings, paragraphs, and lists where possible.
6. If the content cannot be retrieved or no usable content can be extracted, return a short Markdown error message.

## Rules
- Use shell only.
- Do not use browser-based fetch flows.
- Do not use extra tools, packages, or libraries.
- Prefer readable page content over raw HTML when possible.
- Remove obvious scripts, styles, menus, and boilerplate when possible.
- Preserve the source wording as much as possible.
- Output must always be valid Markdown.

## Implementation Note
Use built-in PowerShell commands such as `Invoke-WebRequest` directly from shell and process the response content without external dependencies.