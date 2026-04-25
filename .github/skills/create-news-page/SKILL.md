---
name: create-news-page
description: Create a new Hugo news page for the current date in Russian or English using the Hugo CLI.
allowed-tools: shell
---

## Overview
Create a new Hugo news page using the appropriate Hugo command for the requested language.

## Behavior
1. Accept a language input: `ru` or `en`.
2. Determine the current date in `YYYY-MM-DD` format.
3. Run exactly one of these commands:

   - Russian:
     `hugo new content/ru/news/news-YYYY-MM-DD.md`

   - English:
     `hugo new content/en/posts/news-YYYY-MM-DD.md`

4. Return the created file path.
5. If the language is unsupported or the command fails, return a short error message.

## Rules
- Use the current date at execution time.
- Do not create pages for any language other than `ru` or `en`.
- Do not modify the generated file content.
- Return the exact created path when successful.