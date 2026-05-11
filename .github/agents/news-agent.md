---
name: news-agent
description: Collect AI-related software release news, avoid duplicates using Hugo history, use one Hugo news page per day and language, and fill only the body after the greeting heading.
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
2. Check every source listed in the `Sources` section on every run. This is mandatory even if one source already produced news items.
3. Fetch source content from every listed URL using `get-page-content`.
4. For each source, identify the latest version currently published by that source.
5. For each source, compare the latest source version with the latest version already covered in `get-news-history` for that same software.
6. If `get-news-history` returned `No previous news history found`, treat all discovered relevant items as uncovered. Otherwise, only versions newer than the latest covered version for that software are candidates.
7. Extract only release-note items related to GitHub Copilot or similar AI features from each uncovered candidate version.
8. Ignore unrelated release-note content.
9. Keep all uncovered items between the latest covered version and the current latest source version for each software.
10. For Copilot CLI specifically, include each missing release in the uncovered range (for example, if history has `1.0.35` and source has `1.0.40`, include `1.0.36`, `1.0.37`, `1.0.38`, `1.0.39`, and `1.0.40` when they contain relevant AI changes).
11. Do not skip a source just because another source already has new items. Evaluate all sources first, then generate the full combined set of uncovered relevant items.
12. If a source has a newer version than the latest covered version in news history and that newer version contains qualifying AI or agent changes, include it.
13. If no new relevant items remain after checking all sources, stop and report that no news page update is needed.
14. Determine today's daily news file path for both languages:
   - English: `content/en/news/news-YYYY-MM-DD.md`
   - Russian: `content/ru/news/news-YYYY-MM-DD.md`
15. For each language, check whether today's daily page already exists.
16. If today's daily page exists, reuse that file. Do not create another page for the same date.
17. If today's daily page does not exist, create it once for that language by running `create-news-page`.
18. Never create additional same-day files with suffixes such as `news-YYYY-MM-DD-something.md` when the daily page already exists.
19. Before generating content for a language, read one recent existing news page in that same language and use it as the formatting reference for section style.
20. Generate English content for the English daily page.
21. Generate Russian content for the Russian daily page.
22. Update both daily pages according to the file update rule below.

## File update rule
1. Read the daily file that will be updated.
2. Locate the greeting heading:
   - English: `### Greetings, human 🤖`
   - Russian: `### Приветствую, человек 🤖`
3. Preserve all content up to and including that heading.
4. Treat any existing sections after that heading as already published items for the same day.
5. Combine the existing same-day sections with the newly generated sections.
6. Deduplicate sections by exact heading `### <Software> - <Version>`.
7. Keep existing same-day sections first and append only genuinely new sections in chronological order.
8. Replace everything after that heading with the combined deduplicated content.
9. Do not modify front matter or any text before the heading.

## Content format
All generated page content must be valid Markdown.

For each included item, use:

### <Software> - <Version>
- <List of the new features>

Example:

### GitHub Copilot CLI - 1.0.41
- Faster startup by rendering the UI immediately while authentication resolves in the background.
- Shell completions (bash, zsh, fish) are automatically installed and updated; improved tab-completion for slash commands that accept arguments.

Include multiple sections if multiple new items are found.
If the daily page already contains sections for the same date, append only the missing sections instead of replacing the page with only the newest item.
Keep the old section style used by existing news pages: the `### <Software> - <Version>` heading is the only title line for the item.
Match the wording density and structure of recent existing pages in the same language instead of inventing a new layout.
Do not add a repeated software/version summary line under the heading.
Do not add bullets like `- **<Software> <Version>** - ...` or any other bold introductory bullet that restates the heading.
Do not add label-style bullets such as `- Impact: ...`, `- Source: ...`, `- Why it matters: ...`, `- Влияние: ...`, or `- Источник: ...`.
After each heading, write only plain feature bullets describing the new AI-related changes.
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
- Do not collapse to only one version. Keep all uncovered versions in the gap between covered history and latest source for each software.
- If history exists, include versions strictly newer than the latest covered version and up to the latest source version.
- For patch/minor sequences (for example Copilot CLI `1.0.36` to `1.0.40`), include each missing version as its own section when relevant AI notes exist.
- Order sections from oldest uncovered version to newest uncovered version per software.
- Use version text exactly as found in the source.
- For Visual Studio 2022, and Visual Studio 2026, keep the full numeric version in headings (for example, `17.14.31`), never truncate to `major.minor`.
- If the source labels the release as an April Update, keep that label but preserve the full version number (for example, `April Update 17.14.31` or `April Update 18.5.0`).

## Git rules
- Local content update only. The agent may create and edit news files but must not run any git commands.
- Never run `git add`, `git commit`, `git push`, `gh`, or open pull requests.
- Leave all version control actions for manual execution by the user.

## Result
- If no new relevant items are found, return a short message that no page was created.
- If a page is created or updated, return the daily file path and included software/version items.