# PRODUCT.md — Markdown viewer: `<br>` explicit line breaks

Issue: https://github.com/warpdotdev/warp/issues/13732

Split from: #13652 (bulk raw-HTML-subset request). Sibling specs from the same split:
`specs/GH13732/img-sizing/` (#13721), `specs/GH13652/anchor-links/` (#13725),
`specs/GH13652/tables/` (#13726, raw HTML `<table>`), `specs/GH13652/details-summary/`
(#10259).

## Summary

Warp's Markdown viewer currently renders a raw `<br>` tag as literal text rather than a
line break — both mid-paragraph and inside a GFM pipe-table cell. `<br>` is the *only* way
to author a forced line break inside a pipe-table cell (pipe-table syntax is line-based, so
a literal newline ends the row instead), which makes this gap the more consequential half
of #13726's raw-HTML-table request even before HTML tables themselves are supported.

This spec covers converting `<br>` into a real line break in the `.md` render path,
standalone and inside GFM pipe-table cells, and — since it is one token-type addition,
adjacent, and already flagged as a gap by the same code path — CommonMark's own hard-break
syntax (trailing double-space or backslash before a newline).

Figma: none provided.

## Goals / Non-goals

In scope:

- Recognize `<br>` (and its self-closing/uppercase variants — `<br/>`, `<br />`, `<BR>`) as
  an inline token in the `.md` parser and render it as a forced line break wherever it
  appears in paragraph text.
- Honor `<br>` inside a GFM pipe-table cell as a forced line break within that cell, so the
  cell renders as genuinely multi-line content rather than one line with a literal `<br>`
  substring.
- Support CommonMark hard breaks (a line ending preceded by two or more trailing spaces, or
  by a backslash) as an equivalent forced line break, standalone. Whether hard breaks are
  also honored inside table cells is a call for the tech spec, since GFM pipe-table cells
  are line-based and a literal trailing-newline hard break inside a cell is likely
  unreachable syntactically — `<br>` remains the primary and only reliably-authorable
  mechanism inside a cell.
- Degrade gracefully: an unrecognized `<br …>` variant with attributes Warp doesn't
  understand still produces a line break (attributes are ignored, not rejected). A tag that
  doesn't parse as a `<br …>` shape at all — unclosed (no terminating `>`), or stray `<`
  followed by non-tag content — falls back to literal text, deterministically, not undefined
  or unspecified behavior.

Out of scope (explicit non-goals):

- Any other raw HTML tag (covered by sibling specs listed above).
- Full raw-HTML `<table>` parsing (`specs/GH13652/tables/`) — this spec only makes `<br>`
  work inside the *existing* GFM pipe-table cell path.
- Preserving `<br>` as literal HTML on export/copy where a semantic line break is
  representable — copy/export should produce a real break (e.g. GFM's own hard-break
  syntax, or `<br>` inside a cell), not literal escaped text. Exact serialization choice is
  the tech spec's call.
- Multiple consecutive `<br><br>` producing extra vertical space beyond a single-line gap
  (CommonMark hard breaks don't add blank-line spacing; a `<br>` is one forced break, and
  Warp should not special-case doubled tags into a paragraph break).

## Behavior

1. A standalone `<br>` (or `<br/>`, `<br />`, case-insensitive) appearing mid-paragraph
   forces a line break at that point: text before it and text after it render as two lines
   within the same paragraph, not as a literal `<br>` substring.

2. A `<br>` inside a GFM pipe-table cell forces a line break within that cell. The cell's
   row grows tall enough to fit the extra line, other cells in the row are unaffected
   except for the shared row height, and the break is positioned exactly where the `<br>`
   was authored — not wrapped for width reasons (word-wrap and the authored break are
   independent; wrapping still applies on top of the authored line(s) as normal).

3. A line ending immediately preceded by two or more trailing spaces, or by a backslash,
   forces a line break equivalent to invariant 1 (CommonMark hard break). A single trailing
   space, or a backslash not immediately before a line ending, does not.

4. `<br>` and hard breaks are inert inside inline code spans and code blocks — a `<br>` or
   trailing-backslash sequence typed inside backticks renders as literal text, matching how
   all other inline Markdown syntax is already suppressed inside code.

5. Multiple consecutive `<br>` tags each produce their own forced break (e.g. `<br><br>`
   produces two line breaks, i.e. one visually blank line between content), not a paragraph
   boundary and not collapsed into one break.

6. Copy and export of a document containing an authored line break preserves it as a
   genuine break in the copied/exported form — not literal `<br>` text and not silently
   dropped — using whichever serialization (GFM hard-break syntax, `<br>`, or both
   depending on context) the tech spec specifies as round-trip safe.
