# PRODUCT.md — Markdown viewer: `<picture>`/`<source>` theme-aware images

Issue: https://github.com/warpdotdev/warp/issues/13736
Blocked by: `<img>` sizing support, https://github.com/warpdotdev/warp/issues/13721
(fallback `<img>` rendering must land first — this spec's fallback path builds on it).

## Summary

READMEs commonly use `<picture>` with two `<source media="(prefers-color-scheme: …)">`
variants and a fallback `<img>` to keep a logo or badge legible against both light and
dark backgrounds. Warp's Markdown viewer currently renders the whole block — including
the fallback `<img>` — as literal text, because no parser recognizes `picture`/`source`
and the image model has no concept of more than one candidate source.

This spec covers recognizing a `<picture>` block, selecting the correct `<source>` for the
viewer's active theme (or falling back to the inner `<img>`), and re-selecting live when
the user's system/app theme changes. It is scoped to the one media feature that makes this
pattern useful — `prefers-color-scheme: dark` / `light` — and explicitly excludes the rest
of the `<picture>` media-query surface (responsive/density selection).

Figma: none provided.

## Goals / Non-goals

In scope:

- Recognize a block-level `<picture>…</picture>` element (own lines, per the `<img>` spec's
  block-detection convention) containing zero or more `<source>` elements and exactly one
  fallback `<img>`.
- Evaluate each `<source>`'s `media` attribute for `(prefers-color-scheme: dark)` or
  `(prefers-color-scheme: light)` only, and pick the first matching `<source>`'s `srcset`
  against the viewer's currently active theme.
- If no `<source>` matches (unrecognized/absent media query, or theme detection
  unavailable), render the fallback `<img>` per the `<img>` sizing spec (#13721).
- Re-evaluate source selection and swap the rendered asset when the active theme changes
  while the document is open (no manual reload required).
- Honor the same `width`/`height`/`align` sizing attributes on the fallback `<img>` (and,
  where present, on the selected `<source>`/`<img>` pairing) as the `<img>` spec defines,
  so a themed image sizes the same way a plain one does.
- Degrade gracefully: a `<picture>` with no `<source>` elements (just an `<img>`) renders
  as a plain image; a `<picture>` with no fallback `<img>` at all is a malformed block and
  falls back to literal text (matching the `<img>` spec's own malformed-tag behavior).

Out of scope (explicit non-goals):

- Any `media` feature other than `prefers-color-scheme` — no viewport width, resolution,
  or other responsive media queries.
- `srcset` density/width descriptors (`1x`/`2x`/`480w` candidate lists within a single
  `<source>`). Only the first URL in `srcset` is used; multi-candidate density switching is
  a follow-up if requested.
- `<picture>` used inline (mid-paragraph) rather than as its own block.
- Copy/export round-tripping the full `<picture>`/`<source>` structure losslessly — see
  Behavior §7 for the minimum bar.
- Any change to plain `<img>` or Markdown-native `![]()` image behavior.

## Behavior

1. A Markdown document region delimited by `<picture>` … `</picture>` on their own lines,
   containing one or more `<source>` elements and a trailing fallback `<img>`, renders as a
   single image in the viewer — never as literal text, and never as multiple images.

2. Source selection: each `<source>`'s `media` attribute is evaluated in document order.
   The first `<source>` whose `media` is exactly `(prefers-color-scheme: dark)` and matches
   the viewer's current theme being dark, or exactly `(prefers-color-scheme: light)` and
   matches the current theme being light, wins; its `srcset` (first URL, ignoring density
   descriptors per the non-goals) is the rendered asset.

3. If zero `<source>` elements match — because none are present, none carry a recognized
   `prefers-color-scheme` media query, or the viewer's theme cannot be determined — the
   fallback `<img>`'s `src` renders, exactly as a standalone `<img>` would per #13721.

4. When the user's system theme changes (or Warp's in-app theme override changes) while a
   document containing a `<picture>` block is open, every visible `<picture>` block
   re-evaluates its source selection and swaps to the newly matching asset without
   requiring the user to reload or re-open the file.

5. Sizing attributes (`width`, `height`, `align`) on the fallback `<img>` apply to whichever
   asset is actually rendered (a matched `<source>` or the fallback), so a themed image
   participates in layout identically to a plain `<img>` of the same declared size. If a
   `<source>` element itself carries sizing attributes, the tech spec defines whether those
   are read or ignored (see Non-goals: this spec does not require per-source sizing).

6. A `<picture>` with `<source>` elements but no fallback `<img>` child is treated as
   malformed and renders as literal text (matching the `<img>` spec's malformed-tag
   fallback) rather than picking one `<source>` arbitrarily or rendering nothing. The same
   deterministic literal-text fallback applies to any other malformed `<picture>` block:
   an unclosed `<picture>` (no matching `</picture>`), a stray/malformed `<source>` or
   `<img>` tag within it, or a `<source>` whose `srcset` isn't parseable as a URL (as
   opposed to merely carrying an unrecognized `media` query, which is invariant 3's
   no-match case, not a malformed-input case). None of these states are left unspecified —
   each has exactly one defined outcome: the whole block's raw source is rendered as literal
   text, never a partial render, never undefined behavior.

7. A `<picture>` with a fallback `<img>` but zero `<source>` elements renders exactly as
   that plain `<img>` would — this is a degenerate case, not an error.

8. Copy/export of a document containing a `<picture>` block preserves at minimum the
   fallback `<img>`'s content (matching `<img>` export behavior); the tech spec defines
   whether the full `<picture>`/`<source>` markup survives via canonical re-serialization or
   is collapsed to the single resolved `<img>`. Either way, this is not a byte-exact
   round-trip of the original source — Warp does not guarantee exact source preservation
   anywhere in the markdown pipeline.

9. Loading/error states for the selected asset follow the same behavior as a plain `<img>`
   (per #13721 and the viewer's existing asset-loading conventions) — a broken `<source>`
   URL does not silently fall through to try the next `<source>` or the fallback; it
   surfaces the same broken-image treatment a plain `<img>` would.
