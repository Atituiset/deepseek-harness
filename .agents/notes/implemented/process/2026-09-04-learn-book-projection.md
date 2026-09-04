# Agent Note: Publish the learning book through the docs-site projection

Status: implemented

English | [中文](2026-09-04-learn-book-projection.zh.md)

## Problem

The 15-chapter Agent-development tutorial under `learn/` shipped as a separate mdBook site. Deploying it meant the Pages workflow installed a pinned mdBook binary, built `learn/book/` as a second artifact tree beside `website/.dist`, and mounted it at `/learn/` with no shared navigation, search, or locale handling. The tutorial lived outside the publication manifest, so its pages could not link into published documentation by route and the site's link, outline, and fragment verification never saw them.

## Decision

The mdBook layer is gone. `learn/src/` stays the canonical Markdown home (a tutorial owns its tier like `docs/user/` owns guides), and publication goes through the existing projection: [website/docs.ts](../../../../website/docs.ts) declares all 16 pages via `mirroredPages` — both locale routes intentionally project the available Chinese source under a new `learn/` top-level module with its own sidebar collections (`zh-learn`/`en-learn`) and navigation entry (教程 / Learn).

The projector rewrites the chapters' cross-references into site routes, so a chapter link to `docs/architecture.md` behavior stays code-text (open the source beside the book) while chapter-to-chapter links navigate inside the site. `pnpm run doc-sync` now covers the tutorial with the same gates as every other published page, including built-site fragment verification.

The pairing spec accepted one generalization: a `zh-CN` page whose name is not `.zh.md` may still publish when the manifest mirrors it into both locale routes — the intended `mirroredPages` contract the Cordis `inherited.md` entry already used in the English direction. Pair-named sources keep their stricter sibling assertion.

## Alternatives considered

**Keep mdBook and embed its output.** Rejected: two build systems, two artifact trees, and a tutorial the site gates never verify.

**Rename `learn/src/**` to `.zh.md` and write English stubs.** Rejected: an English counterpart should exist only when someone writes one, and empty stubs would trip the pairing corpus.

**Move the chapters under `docs/`.** Rejected: the tutorial is a standalone learning path, not subsystem reference; a dedicated tier keeps its chapter numbering and its own outline depth (`[2, 3]`).

## Consequences

One build produces the whole site; the workflow lost its mdBook install steps and `learn/book/`, `book.toml`, and `src/SUMMARY.md` are deleted. The tutorial gains site navigation, local search, and the locale switcher for free. Cross-locale readers of `/en/learn/` see the Chinese source today — the documented `mirroredPages` fallback — and converting the entry to `pairedPages` when an English translation lands is a manifest-only change.
