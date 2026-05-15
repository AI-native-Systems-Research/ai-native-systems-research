---
title: BLIS cross-posted blog entries — design spec
date: 2026-05-15
status: approved
---

# BLIS Cross-Posted Blog Entries

## Purpose

Surface blog posts that were originally published on the
[BLIS blog](https://inference-sim.github.io/inference-sim/latest/blog/) inside
the AI-native Systems Research site's Blog tab, without republishing the full
content. Each cross-post is represented locally as a small landing-page stub
that:

- Plugs into Material for MkDocs' `blog` plugin (auto-listing, archive,
  categories, feeds, search).
- Credits all original authors.
- Tells readers that the canonical version lives on the BLIS site.
- Gives the reader a real snippet on the local landing page, then a prominent
  link out to the full post.

Both sites are owned by the same author/team, so the relationship between
them is curated cross-posting, not external syndication.

## User Experience

### Listing card (Blog tab)

A BLIS cross-post appears in the blog listing just like a native post. It
shows: title, publication date, all authors (avatars + names), the `BLIS`
category chip, optional secondary category chips from the site's existing
taxonomy, and a short intro snippet. Visually it is indistinguishable from a
native post on the card — the `BLIS` category chip is the only signal of
provenance at the card level.

### Landing page

Clicking the card takes the reader to a local landing page on the AI-native
site (not directly to BLIS). The landing page shows, in order:

1. The post title.
2. The original post's intro paragraph(s) — mirroring whatever appears before
   the BLIS post's own `<!-- more -->` marker. The author of the stub may
   extend this with curated framing for the AI-native Systems audience when
   useful (the "curated abstract" option).
3. An admonition box at the end:
   > **Cross-posted from the BLIS blog.** This post was originally published
   > on the BLIS blog.
   > [Continue reading on inference-sim →]
   The link inside the admonition renders as a Material button (using the
   built-in `.md-button` class), serving as the call-to-action.

The admonition appears **after** the `<!-- more -->` marker so it does not
show up on the listing card — keeping cards visually indistinguishable from
native posts, with the `BLIS` category chip as the only provenance signal at
the card level.

## Architecture

The design adds no new plugins, scripts, sync jobs, build steps, or template
overrides. It is a pure content-plus-configuration change:

1. **`mkdocs.yml`** — add `BLIS` to `plugins.blog.categories_allowed`.
2. **`docs/blis-post-template.md`** — a template (outside `docs/blog/posts/`
   so the blog plugin ignores it) that authors copy when adding a cross-post.
3. **`docs/blog/posts/blis-<slug>.md`** — one file per cross-post,
   conforming to the template's shape.
4. **`docs/blog/.authors.yml`** — extended on demand whenever a BLIS post
   includes an author not already listed there.

Material's blog plugin handles all listing, routing, category pages, archive,
and feeds.

This design does not attempt search-engine canonicalization (no
`<link rel="canonical">` pointing to the BLIS URL). Each stub is treated as
its own page on this site; readers reach the BLIS original through the
admonition's button. SEO is explicitly out of scope.

## Components

### `mkdocs.yml` change

```yaml
plugins:
  - blog:
      blog_dir: blog
      post_date_format: long
      categories_allowed:
        - Vision
        - llm-d
        - AI Kernels
        - Storage Systems
        - Deep Dives
        - BLIS          # new
```

### Stub-post template (`docs/blis-post-template.md`)

```markdown
---
date: YYYY-MM-DD                       # original BLIS publication date
categories:
  - BLIS                                # always present
  - <Optional secondary>                # from existing categories_allowed
authors:                                # full list verbatim from BLIS source
  - <author_key_1>
  - <author_key_2>
description: >
  One-sentence description from the BLIS post's frontmatter.
---

# <Original post title>

<Intro paragraph(s) mirroring the original BLIS post's pre-`<!-- more -->`
excerpt. Optionally extend with curated framing for the AI-native Systems
audience.>

<!-- more -->

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](<BLIS_post_url>){ .md-button .md-button--primary }
```

Notes on the template's choices:

- The intro paragraph(s) sit *before* `<!-- more -->` so they appear on the
  listing card preview. The admonition sits *after* `<!-- more -->` so it
  only appears on the landing page, keeping cards visually indistinguishable
  from native posts.
- `.md-button` and `.md-button--primary` are built-in Material CSS classes;
  no custom CSS is added.

## Categories

- **Primary, always:** `BLIS`. Added to `categories_allowed`.
- **Secondary, optional, editorial:** drawn from the site's existing
  taxonomy (`Vision`, `llm-d`, `AI Kernels`, `Storage Systems`, `Deep Dives`).
  Chosen per post by whoever adds the stub, based on thematic fit for the
  AI-native Systems audience.
- The BLIS site's own categories (e.g., `Architecture`, `Capacity Planning`,
  `What is BLIS?`) are **not** mirrored into this site's taxonomy. The
  AI-native site's categories reflect its own narrative.

For the two existing BLIS posts, suggested mapping (illustrative; the adder
makes the final call):

| BLIS post | BLIS categories | This site's categories |
|---|---|---|
| Physics of High-Fidelity Distributed Inference Platform Simulation | Architecture, Deep Dives | BLIS, Deep Dives |
| Why Simulate Before You Scale | What is BLIS?, Capacity Planning | BLIS, llm-d |

## Authors

The full author list is copied verbatim from the BLIS source post's
frontmatter into the stub's `authors:` field. No abbreviation, no
"first author only". Material's blog plugin renders avatars and names for
every listed author on both the listing card and the landing page.

If a BLIS author is not already in `docs/blog/.authors.yml`, the workflow
includes adding them there (name, description, GitHub avatar URL, matching
the existing entry shape). Most BLIS contributors are already covered.

## Workflow for adding a new BLIS cross-post

1. Open the BLIS post's source markdown in the inference-sim repo
   (`docs/blog/posts/<slug>.md`).
2. Copy `docs/blis-post-template.md` to
   `docs/blog/posts/blis-<slug>.md` in this repo (filename slug matches the
   BLIS slug for collision visibility).
3. Fill the frontmatter:
   - `date` — original BLIS publication date.
   - `categories` — `BLIS` plus optional secondary categories.
   - `authors` — full list verbatim from the BLIS source.
   - `description` — from BLIS source, or rewritten for this site's audience.
4. Fill the body:
   - H1 title copied from BLIS source.
   - Snippet body: copy the BLIS post's pre-`<!-- more -->` intro
     paragraph(s). Optionally extend with curated framing.
   - Set the admonition's button `href` to the full URL of the published
     BLIS post.
5. If any author isn't in `docs/blog/.authors.yml`, add them.
6. Run `mkdocs serve` and verify card + landing page render as expected.
7. Commit and open a PR.

## Edge cases and failure modes

- **Author missing from `.authors.yml`** — Material's blog plugin fails the
  build with a clear error naming the missing author key. Step 5 of the
  workflow covers this; no defensive code needed.
- **Category not in `categories_allowed`** — build fails. Forces the
  editorial decision (drop the tag, or add it to `categories_allowed`
  intentionally).
- **BLIS post deleted or URL changes** — no automated check. The stub will
  silently link to a 404. Acceptable given low post volume and shared
  ownership of both sites; corrected by updating the stub.
- **Duplicate cross-posts of the same BLIS URL** — convention of
  `blis-<slug>.md` filenames makes accidents visible at the file level. No
  automated dedup.
- **Missing `<!-- more -->` in the stub** — Material auto-truncates by
  length, and the admonition may not appear on the card. The template
  includes the marker; reviewers should keep it.

## Testing

This is a content-and-config change with no application logic. Verification
is observational:

1. **Build smoke test:** `mkdocs build --strict` — fails on any unknown
   category, missing author, or broken internal link.
2. **Local serve:** `mkdocs serve`. Visit the Blog tab and confirm:
   - The new BLIS stub appears in the listing with the right title, date,
     all authors, and the `BLIS` category chip.
   - Clicking the title goes to the local landing page.
   - Landing page shows the snippet body followed by the admonition (with
     embedded "Continue reading" button) at the end.
   - The admonition's button navigates to the original BLIS post URL.
3. **Category page check:** visit the `BLIS` category page and confirm the
   stub is listed.
4. **RSS feed check:** confirm the BLIS stub appears in the blog's `feed.xml`
   with the stub's local URL. (Subscribers see cross-posts alongside native
   posts; acceptable.)

## Scope — explicitly out

- No scraping or automated sync from inference-sim. Adds are manual.
- No special homepage or landing-page treatment for BLIS posts beyond what
  Material's blog plugin provides for all posts.
- No automated detection of BLIS posts that disappear, change URL, or are
  updated after cross-posting.
- No re-export or mirroring of full BLIS post content.
- No changes to native post formatting, workflow, or styling.
- No custom CSS for BLIS cards (the category chip is the only marker on
  cards).
- No SEO / search-engine canonicalization. The stub pages are indexable on
  their own terms; we do not attempt to redirect search-engine credit to
  the BLIS site via `<link rel="canonical">`, hreflang, or similar.

## Initial scope

Two BLIS posts to cross-post on first use of this mechanism:

| BLIS slug | Title |
|---|---|
| `2026/04/09/the-physics-of-high-fidelity-distributed-inference-platform-simulation/` | The Physics of High-Fidelity Distributed Inference Platform Simulation |
| `2026/03/05/why-simulate-before-you-scale/` | Why Simulate Before You Scale |

Full author lists for each will be copied from the inference-sim repo's
source markdown when stubs are written.
