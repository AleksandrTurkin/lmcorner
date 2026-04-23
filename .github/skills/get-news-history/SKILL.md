---
name: get-news-history
description: Scan recent Hugo news posts to identify the latest software versions already covered.
allowed-tools: shell
---

## Overview
Scan `content/en/news` for files matching `news-YYYY-MM-DD.md`.

## Behavior
1. If the directory does not exist, has no matching files, or no usable entries are found, return exactly:
   `No previous news history found`
2. Sort matching files by the date in the filename, descending.
3. Inspect only the 5 most recent matching files.
4. For each file, extract all software `name` and `version` entries from structured metadata or content, preferring front matter, then title, then body if needed.
5. A single file may contain multiple software entries.
6. Skip entries where `name` or `version` cannot be determined confidently.
7. While processing files from newest to oldest, keep only the first entry found for each software `name`.
8. Return a bullet list in this format:
   - `<name> <version> — <filename>`

## Rules
- Ignore files that do not match the naming pattern.
- Use filename date, not modified time.
- Preserve version text exactly as found.
- Skip malformed or unreadable files.
- Do not merge distinct software entries from the same file.