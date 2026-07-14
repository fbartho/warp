# PRODUCT.md — Markdown viewer: `<kbd>` keycap styling

Issue: https://github.com/warpdotdev/warp/issues/13733

## Summary

Warp's Markdown viewer renders `<kbd>` tags as literal text today — there is no handling
for the tag anywhere in the workspace. `<kbd>` is the conventional way to document
keyboard shortcuts in Markdown (there's no Markdown-native equivalent), and docs/READMEs
that use it — including Warp's own — render with visible raw tags instead of the small
keycap-styled badge every other renderer (GitHub included) gives it.

This is the smallest tag in the #13652 raw-HTML-subset split: a single inline style flag,
not a new block type, new attributes, or a new data model. Priority on the source issue is
low (1/5 — "not too important").

Figma: none provided.

## Goals / Non-goals

In scope:

- Recognize inline `<kbd>…</kbd>` in Markdown source and render its content as a small,
  monospace, keycap-styled badge — subtle border/background, matching the visual language
  Warp already uses for inline code spans (`` `code` ``).
- Support this in both places Warp parses HTML-flavored Markdown today: the file/tab
  Markdown viewer and the paste-to-Markdown path, matching how `<u>`/`<s>`/`<code>` already
  work in both.
- Nest correctly with adjacent/enclosing inline formatting (bold, italic, links, inline
  code) the same way `<u>` does today — i.e. `<kbd>` composes as one more inline style flag,
  not a competing mechanism.
- Multiple `<kbd>` tags in the same line render as separate badges (e.g. `<kbd>Cmd</kbd>+<kbd>K</kbd>`
  from the issue's test case renders as two distinct keycaps joined by a literal `+`).

Out of scope (explicit non-goals):

- Any visual distinction between `<kbd>` and `<code>` beyond what's needed to read as a
  "keycap" rather than a "code span" — e.g. no icon glyphs for modifier keys, no platform-
  aware symbol substitution (⌘ for `Cmd`), no attempt to parse/validate shortcut syntax.
  `<kbd>` is styled text, not a semantic shortcut-recorder; the content is rendered
  as-authored.
- Nested `<kbd>` (e.g. `<kbd><kbd>Cmd</kbd></kbd>`) — undefined/degraded is acceptable, no
  crash requirement beyond the general no-panic bar.
- Block-level or standalone `<kbd>` (a `<kbd>` element with block children, or one spanning
  multiple lines) — `<kbd>` is a phrasing/inline element per HTML semantics; multi-line or
  block content inside it is not a target case.
- Any change to `<code>` styling or the existing inline-code style tokens.

## Behavior

1. `<kbd>text</kbd>` in Markdown source renders `text` as a small, monospace, keycap-styled
   badge: subtle background, subtle border, rounded corners — reusing Warp's existing
   inline-code visual treatment (monospace font, background chip, bordered) as the closest
   available precedent, so it reads as consistent with the rest of the viewer's inline
   styling rather than a bespoke new visual language.

2. Plain inline formatting nests inside `<kbd>` and `<kbd>` nests inside other inline
   formatting: `**<kbd>Cmd</kbd>**`, `<kbd>**Cmd**</kbd>`, and `<kbd>Cmd</kbd>` inside a
   link or list item all render with both styles applied, matching how `<u>` already
   composes with bold/italic/links today.

3. The test case from the issue —

   ```markdown
   Press <kbd>Cmd</kbd>+<kbd>K</kbd> to open the command palette.
   ```

   — renders as: the word "Press", a keycap badge reading "Cmd", a literal "+", a keycap
   badge reading "K", then "to open the command palette." No raw angle-bracket tags are
   visible.

4. An unterminated or malformed `<kbd>` (missing `</kbd>`, or `</kbd>` with no matching
   open) degrades to literal text for the unmatched tag — matching `<u>`'s existing
   fallback behavior (an unmatched `</u>` renders as the literal string `</u>`) — never a
   panic and never silently swallowed content.

5. Copy / export preserves the tag choice: a `<kbd>`-styled run canonically re-serializes
   back to `<kbd>…</kbd>` source (matching how an `<u>`-styled run re-serializes to `<u>`
   today), not silently downgraded to plain or code-span text. This is style-fidelity, not
   byte-exact source preservation — Warp does not guarantee reproducing the document's
   original source formatting verbatim.

## Priority framing

Per the issue (importance 1/5) and its role as the smallest split from #13652, this spec
intentionally does not propose new data-model fields, new render primitives, or new
attributes support. The tech spec should confirm the "reuse `inline_code`'s visual style,
add one more `FormattedTextStyles` flag" framing holds up; if it doesn't, that's a signal
this issue is more expensive than its priority suggests and worth re-scoping down further
(e.g. punt on paste-path support) rather than growing to match #13652's siblings.
