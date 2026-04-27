# lmcorner

Source code for https://lmcorner.net, a bilingual Hugo blog (English + Russian) built with the PaperMod theme.

## Tech Stack

- Hugo (extended recommended)
- Theme: PaperMod
- Custom styling in `assets/css/extended/custom.css`
- Multilingual content: English at root (`/`), Russian under `/ru`

## Project Layout

- `content/en` - English posts, news, pages
- `content/ru` - Russian posts, news, pages
- `data/en/glossary.yml` - English glossary terms
- `data/ru/glossary.yml` - Russian glossary terms
- `layouts/shortcodes/term.html` - glossary tooltip shortcode
- `layouts/_default/glossary.html` - glossary page layout
- `assets/css/extended/custom.css` - UI overrides
- `public/` - generated output (do not edit manually)

## Prerequisites

1. Install Hugo Extended (latest stable).
2. Verify installation:

```powershell
hugo version
```

## Local Development

Run local server with drafts enabled:

```powershell
hugo server -D
```

Default URL: http://localhost:1313

Stop the server with `Ctrl+C`.

## Build

Generate static site into `public/`:

```powershell
hugo
```

## Content Workflow

### Create a post

English:

```powershell
hugo new content en/posts/my-new-post.md --kind posts
```

Russian:

```powershell
hugo new content ru/posts/moy-novyy-post.md --kind posts
```

### Create a news item

English:

```powershell
hugo new content en/news/news-YYYY-MM-DD.md --kind news
```

Russian:

```powershell
hugo new content ru/news/news-YYYY-MM-DD.md --kind news
```

### Publish content

In front matter, set:

- `draft = false`
- valid `date`
- meaningful `tags`

## Glossary System

Glossary data is stored per language in:

- `data/en/glossary.yml`
- `data/ru/glossary.yml`

Each entry format:

```yaml
- term: "LLM"
  definition: "Large Language Model"
```

Use term tooltip in content:

```go-html-template
{{< term "LLM" >}}
```

If a term is missing in glossary data, the shortcode renders a visible "Not found" warning.

## Localization Notes

- English is the default language and is served from root `/`.
- Russian is served from `/ru`.
- Menus and home page intro text are configured per language in `hugo.toml`.

## Theme and Customization

- Theme files live in `themes/PaperMod`.
- Prefer overriding via `layouts/` and `assets/css/extended/custom.css`.
- Avoid editing theme source directly unless absolutely necessary.

## Deployment Notes

- `baseURL` is set to `https://lmcorner.net/`.
- `CNAME` is present for custom domain usage.
- `public/` is generated output; regenerate from source instead of hand-editing files.

## Common Commands

```powershell
# local dev with drafts
hugo server -D

# production build
hugo

# build including future-dated content
hugo --buildFuture
```
