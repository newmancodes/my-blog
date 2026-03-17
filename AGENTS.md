# AGENTS.md

Instructions for AI coding agents working in this repository.

## Project Overview

This is a Hugo static site blog ("Engineering Horizons") using the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme (loaded as a
Git submodule). The site is deployed to GitHub Pages via GitHub Actions.

- **Framework**: Hugo (extended edition)
- **Theme**: PaperMod (`themes/PaperMod`, git submodule)
- **Config**: `hugo.toml` (TOML format)
- **Language**: en-gb
- **Author**: Stephen Newman

## Repository Structure

```
hugo.toml              # Site configuration
content/posts/         # Blog posts organized by year/month
archetypes/default.md  # Template for new posts
layouts/               # Custom layout overrides for PaperMod
  _default/_markup/    # Render hooks (mermaid, external links)
  partials/            # Partial overrides (mermaid script loading)
static/                # Static files served as-is
themes/PaperMod/       # Theme (git submodule - do NOT edit directly)
public/                # Build output (gitignored)
```

## Build / Serve / Deploy Commands

```bash
# Local development server with live reload and drafts
hugo server -D

# Local development server (published posts only)
hugo server

# Production build (minified)
hugo --minify

# Production build with custom base URL (as used in CI)
hugo --minify --baseURL "https://example.com/"

# Create a new blog post
hugo new content/posts/YYYY/MM/post-slug.md
```

There are no test or lint commands. Hugo will report build errors at build time.
Validate changes by running `hugo server -D` and checking the output in a
browser.

### CI/CD

The GitHub Actions workflow (`.github/workflows/main.yml`) runs on push to
`main`: checks out with submodules, sets up Hugo (latest extended), builds with
`hugo --minify`, and deploys to GitHub Pages.

## Content Conventions

### Post File Location

Posts live in `content/posts/YYYY/MM/post-slug.md` (organized by year/month).

### Front Matter Format

Use TOML front matter (delimited by `+++`), not YAML (`---`).

**Published post example:**
```toml
+++
date = '2025-11-17T00:00:00+00:00'
draft = false
title = 'Solving Number Problems with Artificial Intelligence'
tags = ['programming', '.net', 'artificial intelligence']
+++
```

**Post with table of contents:**
```toml
+++
date = '2026-03-03T08:27:32Z'
draft = true
title = 'Public Countdown Solver'
tags = ['programming', 'rust', 'wasm', 'python', 'azure', 'github actions', 'artificial intelligence']
ShowToc = true
TocOpen = true
+++
```

**Headless content (data-only, not rendered as a page):**
```toml
+++
headless = true
+++
```

### Front Matter Fields

| Field     | Required | Notes                                          |
|-----------|----------|-------------------------------------------------|
| `date`    | Yes      | ISO 8601 format with timezone                   |
| `draft`   | Yes      | `true` for WIP, `false` for published            |
| `title`   | Yes      | Title case, single-quoted string                 |
| `tags`    | No       | Lowercase array of single-quoted strings         |
| `ShowToc` | No       | PaperMod: show table of contents                 |
| `TocOpen` | No       | PaperMod: expand TOC by default                  |
| `headless`| No       | Set `true` for supporting data files             |

### Content Style

- Posts begin with an introductory paragraph or `## Introduction` heading
- Use `##` for top-level sections within a post (not `#`, which is the title)
- External links open in new tabs (handled by the custom render-link hook)
- Use relative links for internal cross-references: `[text](../other-post)`
- Mermaid diagrams are supported via fenced code blocks with the `mermaid` language identifier
- Raw HTML is allowed in markdown (configured in `hugo.toml`)
- Images and static assets for a post live alongside the `.md` file in the same directory
- Tags are lowercase and use hyphens for multi-word concepts (e.g., `artificial intelligence` is an exception - use the existing tag conventions)

### Existing Tags

`programming`, `.net`, `artificial intelligence`, `career`, `lessons-learned`,
`progression`, `rust`, `wasm`, `python`, `azure`, `github actions`

Reuse existing tags where applicable before creating new ones.

## Layout Customizations

The following PaperMod layouts are overridden:

- **`layouts/_default/_markup/render-link.html`**: External links (http/https) get `target="_blank"`.
- **`layouts/_default/_markup/render-codeblock-mermaid.html`**: Renders mermaid fenced code blocks as `<pre class="mermaid">` and sets a page store flag.
- **`layouts/partials/extend_head.html`** and **`layouts/partials/single.html`**: Conditionally load the Mermaid JS library from CDN when a page contains mermaid diagrams.

## Important Guidelines

- **Do NOT edit files in `themes/PaperMod/`**. It is a git submodule. Override
  theme behavior by placing files in the top-level `layouts/` directory instead.
- **Do NOT commit the `public/` directory**. It is gitignored and regenerated
  on build.
- When cloning or working with submodules, use `git clone --recurse-submodules`
  or run `git submodule update --init --recursive`.
- The `draft = true` flag prevents a post from appearing in production builds.
  Use `hugo server -D` to preview drafts locally.
- Date values should include timezone information (UTC `Z` suffix or explicit
  offset like `+00:00` or `+01:00`).
- Utilise conventional commit style messages, the project hasn't done this to
  date but we should apply them going forward.

## VS Code

Recommended extensions: `budparr.language-hugo-vscode` (Hugo language support),
`ban.spellright` (spell checking, configured for English).
