---
name: notion-to-hugo-posts
description: "Converts and publishes/updates Notion exported Markdown files into Hugo post-compatible static website content."
---

# Notion To Hugo Post Skill

When migrating Markdown articles exported from Notion into this project, follow these rules to create Hugo page bundles, preserve assets, and repair internal references.

## Page Bundles

1. Create every source article as `content/posts/<slug>/index.md`.
2. Derive `<slug>` from the article series and H1, following the existing naming conventions:
   - `OP-TEE: <topic>` -> `optee-<topic>`
   - `OpenSBI: <topic>` -> `opensbi-<topic>`
   - `Linux Kernel: <topic>` -> `linux-kernel-<topic>`
   - `<Superscalar Processor Overview>` -> `superscalar-overview-ch*`, using `-part1` and `-part2` for split chapters.
3. Copy images and other local assets adjacent to the source article into the same bundle. Rewrite their references to bundle-relative paths, for example `![diagram](image.png)`.
4. Delete each migrated source `.md` file after a successful migration, but retain `Skill.md` in the source directory.
5. Do not modify `themes/` or its submodules.

## Front Matter

Create YAML front matter from the source H1 and metadata:

```yaml
---
date: "YYYY-MM-DDT00:00:00+08:00"
title: "Source H1"
author: "Frank Chang"
categories:
  - "Source category"
tags:
  - "Source tag"
series:
  - "Specified series name"
---
```

- Convert source dates from `YYYY/MM/DD` to ISO 8601 format.
- Preserve the source category and tag values in their original order.
- The task specifies `series`; use `OP-TEE Code Trace` for OP-TEE, OpenSBI, and Linux code-trace articles.
- Remove the source H1 and Notion properties: `type`, `status`, `date`, `summary`, `tags`, `category`, `slug`, and `icon`.

## Markdown Conversion

- Convert Notion asides into blockquotes using this exact format:

```md
> ⚠️ The code is based on: [source](https://example.com)
>
> Commit ID: **`<commit>`**
```

- Preserve prose and code. Repair invalid Markdown, unclosed code fences, and incorrect local asset paths.
- Do not use raw HTML for ordinary text. Goldmark omits raw HTML by default; encode literal angle-bracket text as `&lt;...&gt;`.
- Use `sh` fences for normal shell command examples. Use `bash` only when documenting an actual Bash script.

## Internal Links

1. Replace Notion-export `.md` links with resolvable Hugo internal routes, for example `/posts/optee-rpc/`.
2. Replace a link only when the repository has a clear matching target. Keep external links unchanged.
3. When the target is a heading, link to Hugo's generated fragment:

```md
[target](/posts/<bundle>/#<goldmark-heading-id>)
```

4. When the target is a code-trace bullet, concept definition, or named design without a heading, create and use an inline Hugo shortcode anchor without changing the bullet's appearance:

```md
- {{< anchor id="stable-target-id" >}}`target_function()`
```

The corresponding shortcode is `layouts/shortcodes/anchor.html`:

```html
<span id="{{ .Get "id" }}"></span>
```

5. For an exact function or line reference in a code block, prefer Hugo Chroma line anchors:

```md
[target](/posts/<bundle>/#hl-<code-block>-<line>)
```

`hugo.yaml` must retain `markup.highlight.anchorLineNos: true`, `codeFences: true`, and `lineNos: true`. Hugo generates `hl-*` IDs, so do not create a shortcode with the same ID. Verify the actual code-block and line number in generated HTML.

6. Do not alter Markdown-looking links inside fenced code blocks: they are code text and do not render as clickable links.
7. Use the most precise unique target for same-page and cross-page links. Keep an article-level link when no target bullet, heading, or code line can be unambiguously identified.
8. Do not link a named concept to its parent section when a dedicated target bullet exists. Add or reuse a descriptive kebab-case shortcode anchor on that bullet, then update every matching same-page and cross-page link to the precise fragment.
9. When updating an existing article, audit its broad heading links for this case. Only change links whose label unambiguously matches a target bullet; leave intentional section-level links unchanged.

## Validation

Run the following after migration:

```sh
git diff --check
hugo --cleanDestinationDir
```

Verify all of the following:

- Hugo parses every front matter block.
- Every local image and asset reference exists in its page bundle.
- Every active internal link resolves to an existing route.
- Every heading, shortcode, and `#hl-*` fragment exists and is unique.
- Every named-concept link targets its defining bullet when one exists, rather than a broader parent section.
- No replaceable Notion-export internal `.md` links remain.
- The Hugo build has no Markdown, asset, link, or raw HTML omission warnings.
