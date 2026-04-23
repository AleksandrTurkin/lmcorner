---
name: get-page-content
description: Fetch a webpage by URL and return a Markdown version of its readable content using built-in Windows tools.
allowed-tools: shell
---

## Overview
Fetch the content of a webpage from a single HTTP or HTTPS URL and return a readable Markdown version.

## Behavior
1. Accept a single absolute URL as input.
2. Use only built-in Windows/PowerShell capabilities to fetch the page.
3. Extract readable content from the response.
4. Return the result as Markdown, preserving headings, paragraphs, and lists where possible.
5. If the page cannot be fetched or no usable content can be extracted, return a short Markdown error message.

## Rules
- Do not use extra tools, packages, or libraries.
- Prefer readable page content over raw HTML when possible.
- Remove obvious scripts, styles, menus, and boilerplate when possible.
- Preserve the source wording as much as possible.
- Output must always be valid Markdown.

## Implementation Note
Prefer built-in PowerShell commands such as `Invoke-WebRequest` and process the response content without requiring external dependencies.