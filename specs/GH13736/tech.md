# TECH.md — Markdown viewer: `<picture>`/`<source>` theme-aware images

Product spec: `specs/GH13736/product.md`
GitHub issue: https://github.com/warpdotdev/warp/issues/13736
Blocked by: https://github.com/warpdotdev/warp/issues/13721 (`<img>` sizing) — this spec's
fallback path and sizing behavior are defined in terms of that spec's output and must not
be implemented ahead of it landing.

**Dependency status:** #13721 is itself still an unmerged spec draft (PR #13656) as of this
writing — the fallback-rendering and sizing-attribute sections here (§1's `<img>`-config
reuse, §3's sizing pass-through, the sizing fixtures noted in "Testing and validation") will
rebase to whatever #13721 lands with. The novel parts of this spec — `<source>`/`srcset`/
`media` parsing (§1), the theme candidate-list model (§2), and theme resolution/live-swap
(§3–§4) — do not depend on #13721's specific design and are reviewable independently.

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
   model/mod.rs:1735`) is `{width, height, spacing}` — no candidate list, no variant field.
   `BlockItem::Image` (`:1481`) wraps exactly one `AssetSource` + one `ImageBlockConfig`.
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
   `crates/editor/src/render/model/mod.rs:1505`; `hidden_line_ranges(&self, app: &AppContext)`,
   `:795`). So *reading* the current theme during layout requires no new plumbing.
   What's missing is the *change notification*: `prefers-color-scheme` changes arrive as
   `CustomEvent::SystemThemeChanged` (wasm: `crates/warpui/src/platform/wasm/mod.rs:213-226`
   listens on `match_media("(prefers-color-scheme: dark)")`'s `"change"` event; native:
   `WindowEvent::ThemeChanged` in `crates/warpui/src/windowing/winit/event_loop/mod.rs:772`,
   within the broader match arm at `:767-775`)
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
- The same `None`/fail-the-block-detector path handles every other malformed case
  deterministically (product invariant 6): an unclosed `<picture>` (no matching
  `</picture>` found before EOF or the next block boundary), a stray/malformed `<source>`
  or `<img>` tag, or a `srcset` value that doesn't parse as a URL token. None of these are
  left as unspecified parser behavior — all resolve to the literal-text fallback, distinct
  from an unrecognized `media` query on an otherwise well-formed `<source>`, which is a
  no-match (not malformed) case per invariant 3.
- If there are zero `<source>` children but a valid `<img>`, still produce a value (product
  invariant 7) — this is just the #13721 `<img>` path with an extra wrapper element, so
  reuse whatever `<img>`-config-building the sizing spec exposes rather than re-deriving it.

### 2. Model: candidate list threads through three layers, resolves at the last one

A parsed image does not go directly from parser output to the render model — it passes
through an intermediate buffer/editable representation first. Concretely, today's
single-source path is:

1. **Parser output**: `FormattedTextLine::Image(FormattedImage)` (`FormattedImage` at
   `crates/markdown_parser/src/lib.rs:336`, currently `{alt_text, source, title}`).
2. **Buffer layer**: `BufferBlockItem::Image { alt_text, source, title }` — the editable
   in-buffer representation, constructed from parser output at
   `crates/editor/src/content/core.rs:882`, consumed for layout at `edit.rs:726`, serialized
   back out at `markdown.rs:781`, and round-tripped through `text.rs:496` /
   `text.rs:614`. Any candidate list this spec introduces must be representable at *this*
   layer too, or a themed `<picture>` block will lose its candidates the moment it's edited,
   copied, or re-serialized — not just at final render.
3. **Render/layout model**: `BlockItem::Image` (`render/model/mod.rs:1481`), which is what
   §3 resolves to a single effective `AssetSource`.

**Recommended split: resolve to a single `AssetSource` before the buffer layer, at
load/layout time — not by carrying an unresolved candidate list all the way to
`BlockItem::Image`.** Concretely:

- The candidate list (`Vec<(ColorScheme, AssetSource)>` + fallback) is a construct that
  exists transiently during parsing / initial buffer construction, not a field that lives on
  `BufferBlockItem::Image` or `BlockItem::Image` long-term.
- `BufferBlockItem::Image` gains the fields needed to preserve the *source* markup for
  round-trip/export (see product invariant 8 and the "Testing and validation" section below)
  — e.g. retaining the raw `<picture>`/`<source>` structure or at least enough to
  re-serialize it — but the *resolved* asset for rendering is computed once, at the point
  layout first happens (§3), the same point `BlockItem::Image`'s `asset_source` is populated
  today for plain `<img>`.
- This keeps `BlockItem::Image` single-source, as it is today (no new enum variant, no
  `Option<ThemedImageSource>` field needed there) — the theming complexity is contained to
  parsing and buffer construction, and disappears by the time content reaches the render
  model.
- **Live re-selection (§4) re-runs this same resolution step**, not a separate "pick from an
  already-threaded candidate list at render time" path — when the theme changes, the fix is
  to re-resolve from the buffer layer's preserved source markup and produce a new
  `BlockItem::Image`, mirroring how any other content change triggers re-layout. This is why
  the buffer layer must retain enough information to re-resolve, even though the render
  model does not carry a candidate list.

This shape needs one new type at the buffer layer, not at the render layer:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ColorScheme { Light, Dark }
```

`BufferBlockItem::Image` grows a field (naming indicative) carrying the parsed `<source>`
candidates alongside the existing `alt_text`/`source`/`title` fallback fields — e.g.
`themed_candidates: Vec<(ColorScheme, String)>` (source URLs, not yet resolved to
`AssetSource`) — so the information survives edit/serialize round-trips. Resolution to a
concrete `AssetSource` (via the existing `<img>` asset-resolution path per the Constraints
section above) happens once, at the `BufferBlockItem` → `BlockItem::Image` layout step in §3.

### 3. Layout-time resolution

At the layout call site that currently builds `BlockItem::Image` (wherever #13721's `<img>`
config construction happens, which already has access to `app: &AppContext` per the
existing `EmbeddedItem::layout` / `hidden_line_ranges` signatures), resolve a
`BufferBlockItem::Image`'s `themed_candidates` (§2) to a single effective `AssetSource`
for the `BlockItem::Image` being built:

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

### 4. Sequencing: static selection first, live re-selection as a follow-up (the risk item)

**There is no existing `crates/editor` subscriber to app-level appearance changes today.**
The only current consumer of `on_os_appearance_changed` is `AppearanceManager` at the app
layer (`app/src/lib.rs:2708`, calling `refresh_theme_state`) — window chrome and app-wide
theme bookkeeping, not editor content re-layout. Grepping for `os_appearance_changed` across
the repo turns up exactly that app-layer registration plus the platform-side callback
plumbing (`crates/warpui_core/src/platform/app.rs`, `crates/warpui/src/platform/mac/app.rs`,
`crates/warpui/src/windowing/winit/event_loop/mod.rs`) — nothing in `crates/editor` hooks in.
This spec's live re-selection (product invariant 4) is therefore net-new wiring, not a reuse
of an established pattern.

It's tempting to reach for `pending_mermaid_asset` (`render/model/mod.rs:1453`, also present
in `crates/editor/src/content/edit.rs:683/820`) as a precedent, since it's also "layout now,
swap the asset in later without user action." But it isn't the same shape of problem:
`pending_mermaid_asset` is an **async asset-pipeline swap** — a Mermaid diagram lays out with
a placeholder while its SVG renders off-thread, then the resolved asset is pushed back into
the already-laid-out block once ready. It is edit-triggered and per-block, with no app-wide
event or subscriber involved. Theme re-selection is the opposite shape: a single app-wide
event (`os_appearance_changed`) that must fan out to *every* open document and invalidate
*only* the blocks that are theme-sensitive. There is no existing mechanism that does
that fan-out. Do not scope this work assuming `pending_mermaid_asset`'s plumbing is reusable.

Given that, **sequence this spec in two steps, mirroring the issue's own Phase 1/Phase 2
split, promoted here from a risk-mitigation fallback to the primary plan:**

1. **Static selection at load/layout time (ship first).** Resolve a `Themed` source to a
   single effective `AssetSource` at layout time per §3, using `app.system_theme()` as read
   at that moment. This alone satisfies invariants 1–3 and 5–9 and requires no new
   subscriber — it's a pure extension of the existing layout call path. A document opened
   after a theme change already renders correctly; only a *live*, in-place swap while the
   document stays open is deferred.
2. **Live re-selection on an open document (follow-up).** Add the editor-side subscriber
   this section originally proposed as a single step. Two placement options, to be settled
   during implementation based on what re-layout hooks already exist in `crates/editor`:
   - **Option A: subscribe at the `on_os_appearance_changed` callback registration site**
     (alongside `AppearanceManager`'s registration in `app/src/lib.rs`, or via a new
     editor-owned registration) and mark documents containing themed images dirty for
     re-layout, reusing the same invalidation path a manual edit would trigger.
   - **Option B: a narrower theme-change signal surfaced through `AppContext` itself** (e.g.
     a generation counter bumped on `os_appearance_changed`, checked opportunistically) that
     the existing incremental re-layout machinery already consults on its next pass, avoiding
     a new full subscriber if the editor already has a "did the app-level context change"
     check.

   Recommend starting with Option A (explicit subscription, easier to test in isolation) and
   only reaching for Option B if re-layout triggering turns out to already have a hook that
   makes it cheaper. Flag for maintainer input: whether `crates/editor` has any other
   app-level-event-to-re-layout precedent not surfaced by this spec's research.

Scope note (applies to step 2): live re-selection must only re-layout documents that actually
contain a `Themed` image block — a document with only plain images/tables must not re-layout
on every theme toggle.

### 5. TUI surface

This spec's parsing and model changes are shared with `crates/warp_tui`, which renders
Markdown images as text, not pixels: `image_fallback()`
(`crates/warp_tui/src/tui_markdown.rs:295`) turns a `FormattedImage` into an `"Image:
<alt-or-title>"` or `"Image: <source>"` span rather than loading any asset. A themed
`<picture>` block in the TUI therefore never needs pixel-level theme resolution — it renders
the **already-resolved** candidate's source/alt text through the same `image_fallback()`
path a plain `<img>` uses today, once the buffer-layer resolution in §2/§3 has picked a
candidate. No TUI-specific parsing or rendering logic is needed.

**Live re-selection (§4 step 2) is out of scope for the TUI.** There is no TUI-side
appearance-change subscription today, and adding one is not warranted by this issue — a TUI
session showing an `image_fallback()` text span for a themed image is not expected to swap
its resolved candidate if the terminal's color scheme changes mid-session. If the terminal's
theme was static at parse/layout time (as `crates/warp_tui` already assumes for its palette),
the resolved fallback text reflects whatever was current at that point.

### 6. Security

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
- Unclosed `<picture>` (missing `</picture>`), a stray/malformed `<source>`/`<img>` tag, and
  a `srcset` that doesn't parse as a URL → all malformed, literal-text fallback (invariant 6),
  distinct from the unrecognized-`media` no-match case above.
- `srcset` with density descriptors (`img-1x.png 1x, img-2x.png 2x`) → first URL only taken.

### Model/resolution unit tests (`crates/editor/src/render/model/mod_tests.rs`)

- `Themed` source + `SystemTheme::Dark` → resolves to the dark candidate.
- `Themed` source + `SystemTheme::Light` → resolves to the light candidate.
- `Themed` source with only a dark candidate + `SystemTheme::Light` → resolves to fallback
  (invariant 3).
- Sizing attributes on the fallback `<img>` apply regardless of which candidate is resolved
  (invariant 5).
- **Blocked on #13721 for fixtures:** `FormattedImage` (`crates/markdown_parser/src/
  lib.rs:336`) is `{alt_text, source, title}` today — no `width`/`height`/`align` fields.
  These sizing-attribute tests cannot be written against real parser output until #13721
  lands the fields; until then, this test group can only exercise resolution logic against
  hand-constructed `AssetSource`/config fixtures, not an actual parsed `<img>`/`<picture>`.

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

- **The live-swap subscriber (§4 step 2) is the actual unknown in this spec.** Parsing and
  model changes, and static theme selection at load time (§4 step 1), are mechanical
  extensions of the `<img>`/`<table>` spec patterns and the existing layout call path. Wiring
  `os_appearance_changed` into editor re-layout, by contrast, is new territory — confirmed
  during spec review that no existing `crates/editor` subscriber or reusable precedent exists
  (see §4). §4 already sequences this as its own follow-up step rather than bundling it with
  static selection; if implementation reveals there's no cheap re-layout hook, that step
  alone (not the whole spec) could grow from MEDIUM to LARGE without blocking the
  static-selection ship.
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
