---
name: get-news-history
description: Scan recent Hugo news posts to identify the latest software versions already covered.
allowed-tools: shell
---

## Overview
Scan `content/en/news` for files matching `news-YYYY-MM-DD.md` and return the latest already covered version for each software.

## Behavior
1. If the directory does not exist, has no matching files, or no usable entries are found, return exactly:
   `No previous news history found`
2. Sort matching files by the date in the filename, descending.
3. Inspect only the 5 most recent matching files.
4. For each file, extract software `name` and `version` entries using this priority:
   - structured metadata, if present
   - canonical news item headings in the body
5. Treat canonical news item headings as the primary fallback format:
   `### <Software> - <Version>`
6. A single file may contain multiple software entries.
7. Skip entries where `name` or `version` cannot be determined confidently.
8. While processing files from newest to oldest, keep only the first entry found for each software `name`.
9. Return a bullet list in this format:
   - `<name> <version> — <filename>`

## Canonical heading rules
- Accept only level-3 headings that exactly follow:
  `### <Software> - <Version>`
- `name` is the text before ` - `.
- `version` is the text after ` - `.
- Preserve version text exactly as found.
- Do not infer versions from arbitrary prose outside the canonical heading format.
- Ignore headings that do not match confidently.

## Recency rules
- Determine "latest covered" by file date order, not by comparing version strings.
- The first valid entry seen for a software while scanning newest files first is the latest covered entry for that software.
- Do not semver-sort, normalize, or truncate version text.

## Rules
- Ignore files that do not match the naming pattern.
- Use filename date, not modified time.
- Skip malformed or unreadable files.
- Do not merge distinct software entries from the same file.
- Prefer conservative behavior: skip uncertain matches rather than guessing.