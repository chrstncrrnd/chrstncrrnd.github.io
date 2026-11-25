# Project guide

This repository is a personal portfolio and blog built with Astro. Keep changes
focused, follow the existing Astro component/page patterns, and avoid adding
frameworks or content infrastructure when the current static setup is enough.

## Stack and commands

- Astro 7, TypeScript strict config (`tsconfig.json`), Markdown pages.
- Node requirement: `>=22.23.4` (`package.json`). The README documents Bun.
- Development: `bun run dev` or `npm run dev`. When starting the dev server,
  use background mode: `astro dev --background`. Manage it with
  `astro dev stop`, `astro dev status`, and `astro dev logs`.
- Do not start or run the site or its dev server; the user runs it in another
  terminal.
- Production build: `npm run build` (or `bun run build`).
- Preview: `npm run preview`.
- There are no test or lint scripts currently defined in `package.json`.
- Astro Markdown processing enables math with `remark-math` and `rehype-katex`
  in `astro.config.mjs`.

## Structure

- `src/pages/index.astro`: homepage, introduction, featured project cards, and
  three recent posts.
- `src/pages/posts.astro`: chronological post archive.
- `src/pages/posts/*.md`: individual Markdown posts. The filename becomes the
  route slug (for example `posts/example.md` becomes `/posts/example`).
- `src/pages/projects.astro`: projects route; currently a placeholder.
- `src/components/PostList.astro`: discovers, sorts, optionally limits, and
  renders post summaries. Both the homepage and archive use this component.
- `src/components/ProjectCard.astro` and `ProjectGrid.astro`: reusable project
  presentation.
- `src/components/Header.astro` and `Footer.astro`: site-wide navigation,
  theme toggle, and footer.
- `src/layouts/BaseLayout.astro`: global document shell and theme variables.
- `src/layouts/PostLayout.astro`: layout for Markdown articles.
- `public/images/`: project thumbnail assets; `public/`: other static assets.

## Blog posts

Posts are regular Markdown page files, not Astro content collections. Add new
posts under `src/pages/posts/` and use the existing post frontmatter pattern:

```yaml
---
layout: ../../layouts/PostLayout.astro
title: "Post title"
description: "Short summary shown in the post lists."
creation: "23 November 2026"
latest_edit: "23 November 2026"
tags:
  - "Topic"
---
```

`title` and `creation` are required for listing. `description`, `latest_edit`,
and `tags` are optional; omit `latest_edit` if the post has not been edited.
Use a human-readable date compatible with `Date.parse` for `creation`, because
`PostList.astro` sorts newest first using that value. Keep tags as a YAML list
of strings. The post layout renders the title, creation date, and an updated
date only when it differs from creation. Math is available in Markdown.

The post list builds each link from the Markdown filename, displays optional
description/tags, and accepts an optional `limit` prop. The homepage uses
`<PostList limit={3} />`; the posts page renders the full list. Update or reuse
this component rather than duplicating post discovery/rendering logic.

## UI conventions

- Shared theme CSS variables (`--page-background`, `--text-color`,
  `--divider-color`) are defined in `BaseLayout.astro`; use them for theme-aware
  colors.
- The site supports light and dark themes. `Header.astro` toggles
  `document.documentElement.dataset.theme` and persists the selection in
  `localStorage`.
- The base site font is Computer Modern. JetBrains Mono and KaTeX styles are
  imported by `PostLayout.astro`.
- Reusable Astro components generally own their markup and scoped styles.
- Keep post list nested-list selectors scoped appropriately: the list rows are
  direct children of `.post-list`, and tags are a nested list within a row.

## Astro documentation

Full documentation: https://docs.astro.build

Consult the relevant guide before changing related behavior:

- [Routing](https://docs.astro.build/en/guides/routing/)
- [Astro components](https://docs.astro.build/en/basics/astro-components/)
- [Framework components](https://docs.astro.build/en/guides/framework-components/)
- [Content collections](https://docs.astro.build/en/guides/content-collections/)
- [Styling](https://docs.astro.build/en/guides/styling/)
- [Internationalization](https://docs.astro.build/en/guides/internationalization/)




