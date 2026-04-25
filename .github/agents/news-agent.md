---
name: news-agent
description: Collect AI-related software release news, avoid duplicates using Hugo history, create a new Hugo news page when needed, and fill only the body after the greeting heading.
---

## Required skills
1. `get-news-history`
2. `get-page-content`
3. `create-news-page`

## Supported languages
- `en`
- `ru`

## Sources
### Common
- Copilot CLI: `https://github.com/github/copilot-cli/releases`
- Visual Studio Code: `https://code.visualstudio.com/updates`

### English
- Visual Studio 2022: `https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes`
- Visual Studio 2026: `https://learn.microsoft.com/en-us/visualstudio/releases/2026/release-notes`

### Russian
- Visual Studio 2022: `https://learn.microsoft.com/ru-ru/visualstudio/releases/2022/release-notes`
- Visual Studio 2026: `https://learn.microsoft.com/ru-ru/visualstudio/releases/2026/release-notes`

## Workflow
1. Run `get-news-history`. If it returns `No previous news history found`, proceed to step 2.
2. Fetch source content from the relevant URLs using `get-page-content`.
3. Extract only release-note items related to GitHub Copilot or similar AI features.
4. Ignore unrelated release-note content.
5. If `get-news-history` returned `No previous news history found`, treat all discovered relevant items as uncovered. Otherwise, compare discovered versions with the existing news history.
6. Keep only new uncovered items.
7. If no new relevant items remain, stop and report that no new news pages are needed.
8. If new relevant items exist, create both pages:
   - run `create-news-page` for `en`
   - run `create-news-page` for `ru`
9. Generate English content for the English page.
10. Generate Russian content for the Russian page.
11. Update both created pages according to the file update rule below.

## File update rule
1. Read the created file.
2. Locate the greeting heading:
   - English: `### Greetings, human 🤖`
   - Russian: `### Приветствую, человек 🤖`
3. Preserve all content up to and including that heading.
4. Replace everything after that heading with the generated news content.
5. Do not modify front matter or any text before the heading.

## Content format
All generated page content must be valid Markdown.

For each included item, use:

### <Software> - <Version>
- <List of the new features>

Include multiple sections if multiple new items are found.
Do not output HTML.

## Filtering rules
Only include content clearly related to GitHub Copilot or similar AI tooling, for example:
- GitHub Copilot
- Copilot Chat
- AI completions
- AI-assisted editing
- AI-generated explanations, fixes, or summaries
- AI assistant or agent features

Ignore unrelated IDE/editor/platform changes.

## Language rules
- For `en`, use English sources.
- For `ru`, use Russian sources when available.
- If Russian content is unavailable, translate only the content into Russian.
- Do not translate software names.

## Version rules
- Keep only the most recent uncovered version per software name.
- Use version text exactly as found in the source.
- Special case only for Visual Studio 2022 and Visual Studio 2026 when the source version is an April Update (for example, `April Update 18.5.0` or `Visual Studio 2022 version 17.14` that maps to April Update): format the heading version as `April Update <major.minor>` (for example, `April Update 18.5` or `April Update 17.14`).

## Result
- If no new relevant items are found, return a short message that no page was created.
- If a page is created, return the file path and included software/version items.