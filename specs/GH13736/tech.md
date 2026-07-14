# TECH.md — Markdown viewer: `<picture>`/`<source>` theme-aware images

Product spec: `specs/GH13736/product.md`
GitHub issue: https://github.com/warpdotdev/warp/issues/13736
Blocked by: https://github.com/warpdotdev/warp/issues/13721 (`<img>` sizing) — this spec's
fallback path and sizing behavior are defined in terms of that spec's output and must not
be implemented ahead of it landing.

## Context

Three things are true simultaneously, and together they explain why this needs both parser
and model work, not just a new tag handler:

1. **No parser recognizes `picture`/`source`/`srcset`/`prefers-color-scheme` anywhere.**
   `crates/markdown_parser/src/html_parser.rs` has no `"picture"` or `"source"` arm; unknown
   elements fall to the generic recurse-into-children arm (`_ =>` around `:337-347`, see the
   `<table>` spec's notes on the same fallthrough), and since `<source>` is a void element,
   recursing into its (nonexistent) children yields nothing. `crates/markdown_parser/src/
   markdown_parser.rs` has no HTML block-tag handling for `picture` either — this is the
   same gap the `<img>` spec (#13721) fills for bare `<img>`, just one level up.

2. **The image model is single-source.** `ImageBlockConfig` (`crates/editor/src/render/
   model/mod.rs:1720`) is `{width, height, spacing}` — no candidate list, no variant field.
   `BlockItem::Image` (`:1466-1471`) wraps exactly one `AssetSource` + one `ImageBlockConfig`.
   There is no slot anywhere in the render model for "pick one of N sources based on a
   runtime condition."

3. **Theme state is reachable for reads, but not wired to content re-layout.** This is the
   one piece of good news: `AppContext::system_theme() -> SystemTheme`
   (`crates/warpui_core/src/core/app.rs:4739`, delegating to `platform_delegate.system_theme()`
   — implemented per-platform in `crates/warpui/src/platform/mac/delegate.rs:153-154`,
   `crates/warpui/src/windowing/winit/delegate.rs:264-301` for Linux/Windows/wasm, and
   `crates/warpui/src/platform/headless/delegate.rs:63`) is a plain synchronous read, and
   `AppContext` is already threaded into the layout call sites that would need it
   (`EmbeddedItem::layout(&self, text_layout: &TextLayout, app: &AppContext)`,
   `crates/editor/src/render/model/mod.rs:1490`; `hidden_line_ranges(&self, app: &AppContext)`,
   `:795`). So *reading* the current theme during layout requires no new plumbing.
   What's missing is the *change notification*: `prefers-color-scheme` changes arrive as
   `CustomEvent::SystemThemeChanged` (wasm: `crates/warpui/src/platform/wasm/mod.rs:213-226`
   listens on `match_media("(prefers-color-scheme: dark)")`'s `"change"` event; native:
   `WindowEvent::ThemeChanged` in `crates/warpui/src/windowing/winit/event_loop/mod.rs:770-775`)
   and both funnel into `self.callbacks.os_appearance_changed()` →
   `AppContext::os_appearance_changed()` (`crates/warpui_core/src/platform/app.rs:271-275`),
   which runs the app-wide `on_os_appearance_changed` callback. That callback exists for
   window chrome (titlebar, traffic lights, etc.) — nothing in `crates/editor` currently
   subscribes to it to trigger a markdown re-layout. This is the plumbing gap the issue
   flagged; it's a missing *subscriber*, not a missing *signal*.

### Constraints from the `<img>` spec (#13721) — must build on, not duplicate

- The block-level raw-HTML detector for `<img>` (own-line, in `markdown_parser.rs`) and the
  `width`/`height`/`align`-aware `ImageBlockConfig` construction are this spec's foundation.
  `<picture>` detection should reuse the same "own-line raw-HTML block" grammar hook rather
  than inventing a second detection mechanism.
- Whatever asset-resolution path `<img>` uses (`AssetSource` construction from a `src` URL)
  is reused unchanged for both the selected `<source>`'s `srcset` and the fallback `<img>`'s
  `src` — a picture's resolved asset is not a new kind of source, just a different URL
  chosen at layout time.

## Feasibility summary

- **(i) Parse `<picture>`/`<source>` into a candidate list + fallback: MEDIUM.** New DOM
  handling in `html_parser.rs` (or wherever #13721 lands its `<img>` handling) plus a new
  block detector in `markdown_parser.rs`. Mechanically similar to the `<img>` and `<table>`
  block detectors, but must also parse and normalize the `media` attribute into a
  structured `(prefers-color-scheme: dark|light)` enum (or reject/ignore anything else).
- **(ii) Extend the image model with a candidate list: MEDIUM.** `BlockItem::Image` and
  `ImageBlockConfig` need a variant that carries `Vec<(ColorScheme, AssetSource)>` +
  fallback `AssetSource`, resolved to a single effective `AssetSource` at layout time based
  on `app.system_theme()`. This is additive (a new enum variant or an `Option<ThemeVariants>`
  field), not a rework of the existing single-source path — plain `<img>`/`![]()` images are
  unaffected.
- **(iii) Live re-selection on theme change: MEDIUM-LARGE — the real risk in this spec.**
  Requires an editor-side subscriber to `on_os_appearance_changed` that invalidates/re-lays-
  out any laid-out `BlockItem::Image` blocks with a candidate list. No such subscription
  path exists today from `crates/editor` into that callback; this is new wiring, not a
  reuse of an existing mechanism. Scope this as its own implementation checkpoint since it's
  the part most likely to reveal an architectural surprise (e.g., whether editor state has
  any existing "app-level event → re-layout" channel at all, or whether one needs to be
  built from scratch).

This spec implements (i) + (ii) + (iii). Given blocking on #13721, sequence as two MVP
phases (per the issue body):

- **Phase 1** (unblocks once #13721 lands): parse `<picture>` blocks and render the
  fallback `<img>` only — no theme awareness yet. This alone fixes the "renders as literal
  text" bug and is shippable independently.
- **Phase 2**: add `<source>`/`media` parsing, the candidate-list model extension, initial
  theme-based selection at layout time, and the live re-selection subscriber.

## Proposed changes

### 1. Parser: recognize `<picture>` as a block, `<source>` as a candidate

Add a `<picture>` arm alongside wherever #13721 adds `<img>` handling in `html_parser.rs`
(DOM-based path) and the corresponding own-line block detector in `markdown_parser.rs`
(mirroring the `<table>` spec's approach: detect `<picture>` … `</picture>` on their own
lines, extract the raw HTML, hand it to the DOM-based reader). Within a recognized
`<picture>` block:

- Walk direct `<source>` children in document order, reading `media` and `srcset`
  (`get_attribute`-style helpers already present in `html_parser.rs`).
- Parse `media` narrowly: recognize exactly `(prefers-color-scheme: dark)` and
  `(prefers-color-scheme: light)` (whitespace-tolerant); anything else (missing, or a
  different/compound media query) marks that `<source>` as never-matching rather than
  attempting general CSS media-query evaluation. This keeps the parser from needing a media
  query engine.
- From `srcset`, take the first URL token (before any `,` or density descriptor); per
  product non-goals, multi-candidate density lists are not supported — document this as a
  known simplification in a code comment at the parse site.
- Require exactly one fallback `<img>` (the non-`<source>` child, typically last). If absent,
  the block is malformed — return `None`/fail the block detector so the region falls back to
  literal text (product invariant 6), matching the `<table>` spec's malformed-block
  precedent.
- If there are zero `<source>` children but a valid `<img>`, still produce a value (product
  invariant 7) — this is just the #13721 `<img>` path with an extra wrapper element, so
  reuse whatever `<img>`-config-building the sizing spec exposes rather than re-deriving it.

### 2. Model: candidate list on the image block

Extend the image block representation to optionally carry theme candidates. Recommended
shape (naming indicative, align with whatever #13721 lands for `ImageBlockConfig`):

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ColorScheme { Light, Dark }

#[derive(Debug, Clone)]
pub struct ThemedImageSource {
    pub candidates: Vec<(ColorScheme, AssetSource)>, // from <source> elements, in document order
    pub fallback: AssetSource,                        // from the required <img>
}
```

`BlockItem::Image` gains a way to carry this instead of (or wrapping) a single
`AssetSource` — e.g. `source: ImageSource` where `enum ImageSource { Single(AssetSource),
Themed(ThemedImageSource) }`, or an `Option<ThemedImageSource>` alongside the existing
`asset_source` field used as the resolved/effective source. Either keeps the existing
single-source path (plain `<img>`, `![]()`) completely untouched — this is additive.

### 3. Layout-time resolution

At the layout call site that currently builds `BlockItem::Image` (wherever #13721's `<img>`
config construction happens, which already has access to `app: &AppContext` per the
existing `EmbeddedItem::layout` / `hidden_line_ranges` signatures), resolve a `Themed`
source to a single effective `AssetSource`:

```rust
let scheme = match app.system_theme() {
    SystemTheme::Dark => ColorScheme::Dark,
    SystemTheme::Light => ColorScheme::Light,
    // any future SystemTheme variants: treat as "no match", fall through to fallback
    _ => return themed.fallback.clone(),
};
themed.candidates.iter()
    .find(|(s, _)| *s == scheme)
    .map(|(_, asset)| asset.clone())
    .unwrap_or_else(|| themed.fallback.clone())
```

This satisfies product invariants 2–3 and reuses the existing single-source render/paint
path unchanged below this resolution point — `RenderableTable`-style painters never see
"themed-ness," only the resolved `AssetSource`.

### 4. Live re-selection on theme change (the risk item)

Add an editor-side subscriber to app-level appearance changes so open documents re-resolve
themed images without a reload. Two placement options, to be settled during implementation
based on what re-layout hooks already exist in `crates/editor`:

- **Option A: subscribe at the `on_os_appearance_changed` callback registration site**
  (wherever `AppContext`'s callbacks are wired up for the editor's window) and mark
  documents containing themed images dirty for re-layout, reusing the same invalidation
  path a manual edit would trigger.
- **Option B: a narrower theme-change signal surfaced through `AppContext` itself** (e.g. a
  generation counter bumped on `os_appearance_changed`, checked opportunistically) that the
  existing incremental re-layout machinery already consults on its next pass, avoiding a new
  full subscriber if the editor already has a "did the app-level context change" check.

Recommend starting implementation with Option A (explicit subscription, easier to test in
isolation) and only reaching for Option B if the editor's re-layout triggering turns out to
already have a hook that makes it cheaper. Flag for maintainer input: whether
`crates/editor` has *any* existing precedent for "app-level event invalidates open document
layout" (Mermaid diagram async-load invalidation, `pending_mermaid_asset` at
`render/model/mod.rs:1438`, may be the closest existing analog worth modeling this after,
since it's also "layout now, replace asset later without user action").

Scope note: this must only re-layout documents that actually contain a `Themed` image
block — a document with only plain images/tables must not re-layout on every theme toggle.

### 5. Security

`media` values are matched against a fixed allow-list of two literal strings, not evaluated
as CSS (no media-query parser/engine introduced). `srcset` URL extraction is string
splitting, not URL execution. Asset resolution for both candidate and fallback sources goes
through the same trust boundary `<img>` already established (#13721) — no new source path,
no script/event-handler surface introduced by `<source>`'s other attributes (all non-`media`/
`srcset` attributes on `<source>`, and non-`src`/sizing attributes on the fallback `<img>`,
are ignored).

## Testing and validation

### Parser unit tests (`crates/markdown_parser/src/html_parser_tests.rs`, `markdown_parser_tests.rs`)

- `<picture>` with dark + light `<source>` + fallback `<img>` → candidate list of 2 +
  fallback (invariant 1).
- `<source media="(prefers-color-scheme: dark)">` only, no light source → single candidate;
  light theme falls to fallback (invariants 2, 3).
- `<source>` with an unrecognized/compound `media` → source ignored, never selected
  (invariant 3's "unrecognized media" clause).
- `<picture>` with no `<source>`, just `<img>` → degenerate case, equivalent to plain `<img>`
  parse (invariant 7).
- `<picture>` with `<source>`(s) but no fallback `<img>` → malformed, literal-text fallback
  (invariant 6).
- `srcset` with density descriptors (`img-1x.png 1x, img-2x.png 2x`) → first URL only taken.

### Model/resolution unit tests (`crates/editor/src/render/model/mod_tests.rs`)

- `Themed` source + `SystemTheme::Dark` → resolves to the dark candidate.
- `Themed` source + `SystemTheme::Light` → resolves to the light candidate.
- `Themed` source with only a dark candidate + `SystemTheme::Light` → resolves to fallback
  (invariant 3).
- Sizing attributes on the fallback `<img>` apply regardless of which candidate is resolved
  (invariant 5).

### Live re-selection tests

- Simulate `os_appearance_changed` while a document with a `Themed` image is open → resolved
  `AssetSource` changes on next layout pass, no user action required (invariant 4).
- A document with no themed images does not re-layout on the same event (scope note in
  §4) — assert via a re-layout counter/dirty-flag check, not just absence of a visible bug.

### Integration / manual

Per CONTRIBUTING, before/after screenshots and a short recording toggling the OS/app theme
while the issue's motivating test case (`<picture>` with light/dark placeholder images) is
open, showing the live swap. Also capture: a `<picture>` with only a fallback `<img>`
(should look identical to a plain `<img>` render), and a malformed `<picture>` missing its
fallback (should render as literal text, not panic or vanish).

## Risks and follow-ups

- **The live-swap subscriber (§4) is the actual unknown in this spec.** Parsing and model
  changes are mechanical extensions of the `<img>`/`<table>` spec patterns; wiring
  `os_appearance_changed` into editor re-layout is new territory with no confirmed existing
  precedent. If implementation reveals there's no cheap hook, this could grow from MEDIUM
  to LARGE and may warrant splitting Phase 2 into "static theme selection at load time"
  (ship first) and "live re-selection on theme change" (follow-up), similar to how the
  issue itself already splits fallback-only (Phase 1) from theme-aware (Phase 2).
- **Hard dependency on #13721.** Both the fallback-`<img>` rendering and the sizing-
  attribute reuse assume #13721's `ImageBlockConfig`/`AssetSource` construction exists and
  is exposed in a reusable form. This spec should not begin implementation before #13721
  lands, or must be prepared to rebase substantially if #13721's design changes materially.
- **`srcset` density descriptors are dropped, not degraded gracefully documented to the
  user.** If someone relies on `1x`/`2x` for HiDPI switching (not just light/dark), it will
  silently just use the first candidate. Worth a code comment and possibly a follow-up issue
  if this turns out to matter in practice.
- **`SystemTheme` may not be a two-value enum forever.** The resolution logic in §3 treats
  any non-Light/Dark variant as "no match → fallback," which is safe but should be revisited
  if a third system theme state is ever introduced.
