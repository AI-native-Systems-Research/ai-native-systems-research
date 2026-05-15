# BLIS Cross-Posted Blog Entries Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Surface two existing BLIS blog posts under this site's Blog tab as local landing-page stubs that link back to the originals on inference-sim, and put the mechanism in place to add more later.

**Architecture:** Pure content-plus-config change to a Material for MkDocs site. Add `BLIS` to allowed categories; add missing BLIS authors to `.authors.yml`; ship a template file plus two concrete stub posts. No new plugins, scripts, sync jobs, or theme/template overrides. SEO is explicitly out of scope.

**Tech Stack:** Material for MkDocs (`blog` plugin), Markdown.

**Spec:** `docs/superpowers/specs/2026-05-15-blis-cross-posted-blog-entries-design.md`

---

## File Structure

| Path | Action | Responsibility |
|---|---|---|
| `mkdocs.yml` | Modify | Add `BLIS` to `plugins.blog.categories_allowed`. |
| `docs/blog/.authors.yml` | Modify | Add `namasl`, `atantawi`, `vishakha-r` author entries. |
| `docs/blis-post-template.md` | Create | Template for future BLIS cross-posts. Lives outside `docs/blog/posts/` so the blog plugin ignores it. |
| `docs/blog/posts/blis-why-simulate-before-you-scale.md` | Create | First cross-post stub (Why Simulate Before You Scale). |
| `docs/blog/posts/blis-building-trust-physics-of-simulation.md` | Create | Second cross-post stub (Physics of High-Fidelity Simulation). |

## Reference: BLIS author key mapping

The BLIS blog uses different author keys than this site's `.authors.yml`. When copying `authors:` from a BLIS source post, translate each key as follows:

| BLIS key | Person | This site's key | Status |
|---|---|---|---|
| `dipanwita` | Dipanwita Guhathakurta | `susiejojo` | exists |
| `mert` | Mert Toslali | `toslali-ibm` | exists |
| `jing` | Jing Chen | `jgchn` | exists |
| `nick` | Nick Masluk | `namasl` | **add in Task 2** |
| `michael` | Michael Kalantar | `kalantar` | exists |
| `asser` | Asser Tantawi | `atantawi` | **add in Task 2** |
| `vishakha` | Vishakha Ramani | `vishakha-r` | **add in Task 2** |
| `srini` | Srinivasan Parthasarathy | `sriumcp` | exists |
| `fabio` | Fabio Oliveira | `oliveira` | exists |

## Notes on testing for this plan

This is a content-and-config change with no application logic, so the standard TDD pattern doesn't apply. The verification step for each task is:

- `mkdocs build --strict` — catches unknown categories, missing authors, broken internal links.
- `mkdocs serve` — local visual inspection where the rendered output is the only meaningful test surface (admonition placement, button rendering, category chips, author avatars).

Each task's verification step states explicitly what to look for.

---

## Task 1: Add `BLIS` to allowed blog categories

**Files:**
- Modify: `mkdocs.yml` (lines 75-80, `categories_allowed:`)

- [ ] **Step 1: Add `BLIS` to the list.**

In `mkdocs.yml`, the `categories_allowed:` block (around lines 75-80) currently reads:

```yaml
      categories_allowed:
        - Vision
        - llm-d
        - AI Kernels
        - Storage Systems
        - Deep Dives
```

Change it to:

```yaml
      categories_allowed:
        - Vision
        - llm-d
        - AI Kernels
        - Storage Systems
        - Deep Dives
        - BLIS
```

- [ ] **Step 2: Verify the build still passes.**

Run: `mkdocs build --strict`

Expected: build succeeds. (Nothing references `BLIS` yet, so no behavior change; we're widening the allowed set.)

- [ ] **Step 3: Commit.**

```bash
git add mkdocs.yml
git commit -m "Allow BLIS as a blog category"
```

---

## Task 2: Add new BLIS authors to `.authors.yml`

**Why:** Both BLIS posts list `nick`, `asser`, and `vishakha` in their author lists. Without entries in this site's `.authors.yml`, the build will fail with "unknown author" once we add the stub posts in Tasks 4 and 5.

**Files:**
- Modify: `docs/blog/.authors.yml`

- [ ] **Step 1: Append three author entries.**

Append the following to `docs/blog/.authors.yml` (after the last existing entry, preserving two-space indentation):

```yaml
  namasl:
    name: Nick Masluk
    description: AI-native Systems Researcher
    avatar: https://github.com/namasl.png
  atantawi:
    name: Asser Tantawi
    description: AI-native Systems Researcher
    avatar: https://github.com/atantawi.png
  vishakha-r:
    name: Vishakha Ramani
    description: AI-native Systems Researcher
    avatar: https://github.com/vishakha-r.png
```

These keys (`namasl`, `atantawi`, `vishakha-r`) match the GitHub-handle convention used by the existing entries (e.g., `toslali-ibm`, `susiejojo`).

- [ ] **Step 2: Verify YAML parses and the build still passes.**

Run: `mkdocs build --strict`

Expected: build succeeds. (No post references these authors yet.)

- [ ] **Step 3: Commit.**

```bash
git add docs/blog/.authors.yml
git commit -m "Add BLIS co-authors: Nick Masluk, Asser Tantawi, Vishakha Ramani"
```

---

## Task 3: Create the cross-post template file

**Why:** Future BLIS cross-posts will be added by copy-rename-fill. The template enforces consistent structure (frontmatter shape, admonition wording, button styling, `<!-- more -->` placement).

**Files:**
- Create: `docs/blis-post-template.md` (outside `docs/blog/posts/` so the blog plugin does not pick it up)

- [ ] **Step 1: Create the template file with this exact content.**

Path: `docs/blis-post-template.md`

```markdown
---
date: YYYY-MM-DD
categories:
  - BLIS
  - <Optional secondary category from this site's taxonomy>
authors:
  - <author_key>
  - <author_key>
description: >
  One-sentence description suitable for this site's audience.
---

# <Original BLIS post title>

<Intro paragraph(s) mirroring whatever appears before the BLIS post's
own `<!-- more -->` marker. Optionally extend with curated framing for
the AI-native Systems audience.>

<!-- more -->

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](<BLIS_post_url>){ .md-button .md-button--primary }
```

- [ ] **Step 2: Verify the template is NOT picked up as a post.**

Run: `mkdocs build --strict`

Expected: build succeeds. No new entry for "Original BLIS post title" anywhere. (The blog plugin only scans `docs/blog/posts/`. A file in `docs/` is treated as a normal page only if it's referenced from `nav` or sits under a navigated section. Since this file is not in `nav` and your `mkdocs.yml` does not set `not_in_nav` for it, MkDocs may emit a warning that the file is not in the navigation.)

If MkDocs warns "file is not included in the 'nav' configuration", add the template path to the `not_in_nav` block in `mkdocs.yml`:

```yaml
not_in_nav: |
  /privacy.md
  /blis-post-template.md
```

Re-run `mkdocs build --strict` and confirm the warning is gone.

- [ ] **Step 3: Commit.**

```bash
git add docs/blis-post-template.md mkdocs.yml
git commit -m "Add template for BLIS cross-posted blog entries"
```

(If `mkdocs.yml` was not modified in Step 2, drop it from `git add`.)

---

## Task 4: Create cross-post stub for "Why Simulate Before You Scale"

**Files:**
- Create: `docs/blog/posts/blis-why-simulate-before-you-scale.md`

**Source reference:** `https://raw.githubusercontent.com/inference-sim/inference-sim/main/docs/blog/posts/why-simulate-before-you-scale.md`

**Original BLIS URL:** `https://inference-sim.github.io/inference-sim/latest/blog/2026/03/05/why-simulate-before-you-scale/`

**Per the spec's suggested mapping:** categories are `BLIS` + `llm-d` (the post motivates BLIS as a tool for planning llm-d-style inference deployments).

- [ ] **Step 1: Create the file with this exact content.**

Path: `docs/blog/posts/blis-why-simulate-before-you-scale.md`

```markdown
---
date: 2026-03-05
categories:
  - BLIS
  - llm-d
authors:
  - susiejojo
  - toslali-ibm
  - jgchn
  - namasl
  - kalantar
  - atantawi
  - vishakha-r
  - sriumcp
  - oliveira
description: >
  Why simulate LLM inference infrastructure before scaling: a flight-simulator
  approach to capacity planning that runs on a laptop, no GPUs required.
---

# Why Simulate Before You Scale

Deploying large language models in production is one of the most expensive infrastructure decisions an organization can make. A single high-end GPU costs upwards of $30,000, and a production cluster can run into millions per year. Yet most teams make their first scaling decisions based on rough estimates, vendor benchmarks, or — worst of all — trial and error on live hardware.

What if you could test your deployment plan *before* spending a dollar on GPUs?

<!-- more -->

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](https://inference-sim.github.io/inference-sim/latest/blog/2026/03/05/why-simulate-before-you-scale/){ .md-button .md-button--primary }
```

- [ ] **Step 2: Build to verify frontmatter and category/author references resolve.**

Run: `mkdocs build --strict`

Expected: build succeeds. No warnings about unknown authors or categories. A new post page is generated under `site/blog/`.

- [ ] **Step 3: Visual check via local serve.**

Run: `mkdocs serve` (in background or another terminal). Open `http://127.0.0.1:8000/blog/` and confirm:

- The new entry appears in the blog listing.
- Card shows: title "Why Simulate Before You Scale", date "March 5, 2026", all 9 author avatars, and two category chips (`BLIS` and `llm-d`).
- Card preview text shows the intro paragraphs ending with "*before* spending a dollar on GPUs?".
- Card does **not** show the admonition box (it lives after `<!-- more -->`).

Click the card. On the landing page, confirm:

- Title, then the two intro paragraphs.
- Below the intro, the cyan info admonition titled "Cross-posted from the BLIS blog" with a "Continue reading on inference-sim →" button.
- Clicking the button opens the BLIS post URL.

Stop `mkdocs serve` when done.

- [ ] **Step 4: Commit.**

```bash
git add docs/blog/posts/blis-why-simulate-before-you-scale.md
git commit -m "Add BLIS cross-post: Why Simulate Before You Scale"
```

---

## Task 5: Create cross-post stub for "Physics of High-Fidelity Distributed Inference Platform Simulation"

**Files:**
- Create: `docs/blog/posts/blis-building-trust-physics-of-simulation.md`

**Source reference:** `https://raw.githubusercontent.com/inference-sim/inference-sim/main/docs/blog/posts/building-trust-physics-of-simulation.md`

**Original BLIS URL:** `https://inference-sim.github.io/inference-sim/latest/blog/2026/04/09/the-physics-of-high-fidelity-distributed-inference-platform-simulation/`

**Per the spec's suggested mapping:** categories are `BLIS` + `Deep Dives`. Author list is the same as Task 4 (same nine co-authors).

- [ ] **Step 1: Create the file with this exact content.**

Path: `docs/blog/posts/blis-building-trust-physics-of-simulation.md`

```markdown
---
date: 2026-04-09
categories:
  - BLIS
  - Deep Dives
authors:
  - susiejojo
  - toslali-ibm
  - jgchn
  - namasl
  - kalantar
  - atantawi
  - vishakha-r
  - sriumcp
  - oliveira
description: >
  What it takes to build a discrete-event simulator faithful enough to guide
  LLM inference capacity planning and configuration search at production scale.
---

# The Physics of High-Fidelity Distributed Inference Platform Simulation

Production LLM inference platforms are distributed systems where routing policies, admission control, autoscaling, and engine-level scheduling all interact to determine latencies and throughput. How do you explore how different policies and configurations affect these KPIs before deploying to production? Testing a new routing policy or autoscaling threshold on live traffic risks cascading bugs across the fleet, while building separate test environments burns GPU-hours and still cannot predict interactions between cluster-level policies and engine-level batch dynamics.

The answer is **end-to-end simulation**: model the entire distributed inference stack to explore how policies and configurations affect latencies and throughput for your workloads. *What does it take to build a simulator accurate enough to guide these decisions?* The challenge lies in capturing the right mechanisms. At the engine level, batches process together — all requests wait for the slowest operation to finish, so KV cache fills trigger preemptions and long prompts stall short decodes. At the cluster level, routing policies operate on stale cache state, admission control gates overload, and prefill/decode disaggregation trades utilization for latency. At the control plane, autoscalers react to lagged metrics, creating oscillations. When these couplings are not modeled, predictions diverge: a back-of-the-envelope model might predict 50ms time-to-first-token while production measures 200ms.

<!-- more -->

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](https://inference-sim.github.io/inference-sim/latest/blog/2026/04/09/the-physics-of-high-fidelity-distributed-inference-platform-simulation/){ .md-button .md-button--primary }
```

- [ ] **Step 2: Build to verify frontmatter and category/author references resolve.**

Run: `mkdocs build --strict`

Expected: build succeeds.

- [ ] **Step 3: Visual check via local serve.**

Run: `mkdocs serve`. Open `http://127.0.0.1:8000/blog/` and confirm:

- The new entry appears in the blog listing (in addition to the Task 4 entry and the existing native posts).
- Card shows: title, date "April 9, 2026", all 9 author avatars, chips for `BLIS` and `Deep Dives`.
- Card preview ends with "production measures 200ms.".
- Card does not show the admonition.

Click the card and confirm the landing page renders the title, two intro paragraphs, and the admonition with the button. The button navigates to the BLIS post URL.

Stop `mkdocs serve` when done.

- [ ] **Step 4: Commit.**

```bash
git add docs/blog/posts/blis-building-trust-physics-of-simulation.md
git commit -m "Add BLIS cross-post: Physics of High-Fidelity Distributed Inference Platform Simulation"
```

---

## Task 6: End-to-end verification

**Why:** Each previous task verified its own piece. This task confirms the system as a whole behaves correctly.

- [ ] **Step 1: Clean build from scratch.**

Run:
```bash
rm -rf site
mkdocs build --strict
```

Expected: build succeeds. No warnings about unknown authors, unknown categories, missing files, or broken links.

- [ ] **Step 2: Verify the `BLIS` category page exists and lists both stubs.**

Run: `ls site/blog/category/blis/ 2>/dev/null && echo "OK" || echo "MISSING — investigate"`

Expected: `OK`.

Inspect the category page:
```bash
grep -o '<h2[^>]*>[^<]*</h2>\|<a href="[^"]*blis-[^"]*"' site/blog/category/blis/index.html | head -20
```

Expected: both stub titles appear and links to both stubs are present.

- [ ] **Step 3: Verify the `BLIS` category chip appears on both stubs' cards.**

Run: `grep -c 'category/blis/' site/blog/index.html`

Expected: at least `2` (one link per stub card). Exact count depends on Material's link emission — a value of 2 or more is correct.

- [ ] **Step 4: Verify the RSS feed includes both stubs.**

First locate the feed file (the default name is `feed_rss_created.xml` but configurations vary):

```bash
ls site/blog/feed*.xml
```

Then grep for stub references in the feed file(s) listed:

```bash
grep -c 'blis-' site/blog/feed*.xml
```

Expected: at least `2` (URLs of the two stubs appear in the feed). If `ls` returns no files, the blog plugin's feed generation is disabled by configuration — note this and skip.

- [ ] **Step 5: Final visual smoke test via `mkdocs serve`.**

Run: `mkdocs serve`. Visit:

- `http://127.0.0.1:8000/blog/` — confirm all four posts (two native + two BLIS) appear in the listing.
- `http://127.0.0.1:8000/blog/category/blis/` — confirm both BLIS stubs are listed.
- `http://127.0.0.1:8000/blog/category/deep-dives/` — confirm Physics-of-Simulation and the sim2real native post are both listed (both have Deep Dives category).
- `http://127.0.0.1:8000/blog/category/llm-d/` — confirm Why-Simulate-Before-You-Scale and the native llm-d posts both appear.

Stop `mkdocs serve` when done.

- [ ] **Step 6: Push.**

Confirm the branch is clean and push:

```bash
git status
git log --oneline -8
git push
```

(If working on a feature branch, push to that branch and open a PR. If on `main` directly per repo convention, push to origin/main.)
