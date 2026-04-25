# Copilot Instructions for lmcorner.net

## Project Overview

- Framework: Hugo version v0.150.1.
- Theme: PaperMod, with custom overrides in `layouts/` and `assets/css/extended/custom.css`.
- English content is the primary language and lives at `content/en`.
- Russian content lives at `content/ru`.

## Source of Truth

- Edit source files, not `public/`.
- Prefer `content/`, `layouts/`, `data/`, `i18n/`, `assets/`, and `hugo.toml`.
- Prefer local overrides over editing `themes/PaperMod/` directly.

## Content

- Keep English and Russian content aligned when the change affects both languages.
- Use existing front matter conventions from `archetypes/posts.md` and current posts.
- When adding news items, follow the conventions in `archetypes/news.md` and the front matter conventions used in the existing content within `content/ru/news` and `content/en/news`.
- Keep the writing practical and direct; avoid generic filler.

## Localization

- Navigation and UI text may require updates in both `hugo.toml` and `i18n/`.
- English is rooted at `/`; Russian is under `/ru/`.

## Glossary

- Glossary data lives in `data/en/glossary.yml` and `data/ru/glossary.yml`.
- The `term` shortcode depends on the current glossary data format; do not change it casually.
- If glossary behavior changes, check `layouts/shortcodes/term.html` and `layouts/_default/glossary.html`.
