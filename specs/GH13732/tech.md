# TECH.md — Markdown viewer: `<br>` explicit line breaks

Product spec: `specs/GH13732/product.md`
GitHub issue: https://github.com/warpdotdev/warp/issues/13732
Related specs: `specs/GH13652/tables/` (raw HTML `<table>`, shares the cell-model
question this spec resolves for the narrower `<br>`-in-GFM-cell case).

## Context

### The `.md` inline tokenizer has no line-break concept

`InlineToken` (`crates/markdown_parser/src/markdown_parser.rs:1675-1694`) is:

```rust
enum InlineToken<'a> {
    Delimiter { kind: DelimiterKind, count: usize },
    Text(&'a str),
    BackslashEscape(char),
    HtmlEntity(char),
    CodeSpan(&'a str),
    AutoLink(String),
    LinkEnd,
    UnderlineEnd,
}
```

No `LineBreak`/`br` variant. `parse_inline_token` (`:1510-1550`) dispatches through an
`alt((...))` chain (`backslash_escape`, `html_entity`, `code_span`,
`parse_inline_token_link_start/end`, asterisk/underscore/strikethrough delimiters,
autolink, `parse_inline_token_underline_start/end`, `whitespace`, `text`,
`unmatched_char`). Verified: neither `<br>` nor a trailing-double-space/backslash hard
break is recognized anywhere in this chain — a `<br>` today falls through to `text`/
`unmatched_char` and is emitted as literal characters. This is a new inline token, wired in
alongside the existing `underline_start`/`underline_end` pair (`<u>`/`</u>`) as the closest
precedent for an HTML-tag-shaped inline token.

### GFM table cells go through the same tokenizer — confirmed, not assumed

`parse_table_row` → `parse_table_cell` (`markdown_parser.rs:568-619`) extracts each cell's
raw text (explicitly rejecting embedded `\n`/`\r` at `:596-597`, since rows are line-based),
then `parse_table_row` maps every cell through `parse_cell_content` (`:669`), which calls
`parse_inline::<E>` (`:974-1029`, the same function paragraph text goes through) and falls
back to `FormattedTextFragment::plain_text(cell)` only on a full parse failure. So: whatever
inline-token change is made to handle `<br>` in paragraphs is automatically picked up inside
table cells too, with **one caveat verified below** — the cell's *model* can't hold a second
line even if the tokenizer recognizes the tag.

### The cell (and paragraph inline run) is structurally single-line — this is the real constraint

`FormattedTextInline = Vec<FormattedTextFragment>` (`lib.rs:501`), and
`FormattedTextFragment { text: String, styles: FormattedTextStyles }` (`lib.rs:505-508`) —
a flat run of styled text fragments, no break concept. `FormattedTable`
(`lib.rs:355-359`) is `headers: Vec<FormattedTextInline>`, `rows: Vec<Vec<FormattedTextInline>>`
— each cell is exactly one `FormattedTextInline`.

Verified at the render/measurement layer that this "single line" is not just a type-level
artifact but an actual behavioral constraint today: `measure_table_cells`
(`crates/editor/src/content/edit.rs:260-323`) builds each cell's `StyledBufferRun`s by
concatenating every fragment's `text` into one `line.text` and laying it out once via
`layout.layout_text_with_options(..., f32::MAX, ...)` — a single unbounded-width text frame.
The multi-line `CellLayout` machinery that *does* exist (`render/model/mod.rs:1736-1776`,
`from_text_frame`, populating `line_heights` from a `TextFrame`'s wrapped lines) is a
**render-time word-wrap artifact** at final column width, not an authored break — there is
no code path today that inserts a forced line break into a cell's text before that wrap
happens. A literal `\n` character embedded in `fragment.text` is not filtered out by this
measurement code, but nothing produces one today, and downstream code that treats a cell as
single-line (`parse_table_cell` rejecting raw `\n` in the source syntax, `to_internal_format`
tab/newline-delimited serialization at `lib.rs:396-411`) means an unvetted embedded `\n`
would silently corrupt round-tripping rather than "just work." This matches
`specs/GH13652/tables/tech.md`'s finding for the same constraint and reaches the same
conclusion: **a cell needs a small model change to support an authored break, it cannot be
retrofitted as pure rendering.**

### The `<br>` → `LineBreak` "reuse precedent" is real but solves a different-shaped problem

`html_parser.rs:335` maps `"br" => FormattedTextLine::LineBreak`, and `"br"` is in
`PHRASING_ELEMENT_TAGS` (`:27`) alongside `span`/`i`/`code`/`strong`/`em`/`a`/`s`/`u`/`ins`.
`FormattedTextLine::LineBreak` (`lib.rs:163`) is a variant of `FormattedTextLine` — Warp's
**block-level** line enum (siblings: `Heading`, `Line(FormattedTextInline)`, `CodeBlock`,
`Table`, etc.), not of `FormattedTextInline`/`FormattedTextFragment`. In the paste path, a
`<br>` produces an entire standalone block-line in the document's line list. That is the
right shape for "a `<br>` in pasted HTML becomes its own line in the buffer," but it is the
**wrong shape** for this issue's two asks: an inline break *inside* a paragraph's existing
`FormattedTextInline` run, and — more acutely — a break *inside one cell* of a
`FormattedTable` row, where inserting a new top-level `FormattedTextLine` is not
structurally possible (a cell is `Vec<FormattedTextFragment>`, it cannot contain a
`FormattedTextLine`). So this spec cites `html_parser.rs:335` as evidence Warp already
treats `<br>` as a semantic break elsewhere (precedent for *intent*), but the `.md`-path
implementation needs its own inline-level construct — it cannot literally call the same
code.

### No existing hard-break handling

Confirmed by search: no `to_plain_text`-adjacent or tokenizer logic anywhere in
`markdown_parser.rs`/`lib.rs` recognizes CommonMark's trailing-double-space or
trailing-backslash hard break. This is a clean addition, not a fix to something partially
built.

## Feasibility summary

- **(i) Recognize `<br>` as an inline token, forced break inside a paragraph: SMALL.** One
  new `InlineToken`/fragment-level construct plus a parser branch in the `alt(...)` chain,
  following the `<u>`/`</u>` precedent. Rendering a forced break within a single paragraph's
  flow is a well-trodden shape (multi-line paragraphs already exist as sibling
  `FormattedTextLine::Line` entries or via soft-wrap); the main work is deciding *how* the
  break is represented within one `FormattedTextInline` (see below) and threading it through
  paragraph layout.
- **(ii) `<br>` (and optionally hard breaks) inside a GFM table cell: MEDIUM.** Same token
  recognition as (i), but the cell model (`FormattedTextInline`, flat fragment list) has no
  slot for a break, and `measure_table_cells` currently produces one unbounded text frame per
  cell. Needs the same category of change `specs/GH13652/tables/tech.md` proposed for
  HTML-table cells — this issue reaches the identical fork in the road for GFM cells,
  independently of whether HTML tables ever land.
- **(iii) CommonMark hard breaks (trailing double-space / backslash): SMALL for paragraphs,
  same MEDIUM-cell caveat as (ii) if extended to cells.** Product spec scopes hard breaks
  as standalone-only by default; cell support is optional and gated on the same model change
  as `<br>`.

## Proposed changes

### 1. Represent a forced break at the fragment/inline level

Introduce the break as a **fragment-level construct**, not a block-level one (ruling out
reusing `FormattedTextLine::LineBreak` directly, per the Context section). Two structurally
equivalent options, mirroring the fork `specs/GH13652/tables/tech.md` already identified for
the HTML-table case — resolving it here for the narrower, higher-priority `<br>` case:

- **Option A: sentinel fragment.** Add a way to mark a `FormattedTextFragment` (or a
  dedicated zero-width fragment) as a hard break — e.g. a `FormattedTextStyles` flag, or a
  reserved fragment shape consumers check for. `FormattedTextInline` stays
  `Vec<FormattedTextFragment>`; a break is just a specific fragment in that flat list.
  Smaller ripple: paragraph layout, `to_plain_text`, `to_internal_format`, and cell
  measurement each need one new match arm, but no type signature changes.
- **Option B: line-structured cell/paragraph.** Change the unit to
  `Vec<Vec<FormattedTextFragment>>` (a list of lines) wherever a break needs to live —
  paragraphs and/or table cells. More invasive (touches the shared `FormattedTable` type
  exactly as GH13652's Option A would, plus paragraph `FormattedTextLine::Line`), but makes
  "this content is multi-line" a first-class, harder-to-miss type rather than something
  every consumer must remember to scan a flat list for.

**Recommend Option A** for this issue specifically: the ask is one forced break, not
general multi-line-as-a-type-level-concept, and Option A is the smaller, more local change.
Note for maintainer review: if `specs/GH13652/tables/` (HTML tables, which also needs a
cell-model change for the same underlying reason) lands around the same time, converging on
one option between the two specs is worth doing explicitly rather than shipping two
different mechanisms for "a cell has 2 lines." Flag this cross-spec decision rather than
silently picking one.

Whichever option: the sentinel/break must be excluded from `text` concatenation in
`measure_table_cells` (`edit.rs:260-323`) in favor of starting a new `StyledBufferRun`/line,
must be skipped or translated (not literal `\n`) in `to_internal_format`
(`lib.rs:396-411`, which is tab/newline-delimited and cannot hold a literal newline inside a
cell without corrupting row boundaries — same hazard flagged in the Context section), and
must round-trip through `to_plain_text` as either `<br>` or the GFM hard-break syntax
(product invariant 6 leaves the exact choice to this spec: recommend emitting `<br>` for
cell breaks, since GFM prose export has no hard-break syntax that survives a pipe-table
cell's line-based grammar, and CommonMark hard-break syntax for standalone paragraph breaks
since that achieves canonical re-serialization — the break survives semantically — and is
more idiomatic Markdown; Warp does not guarantee byte-exact preservation of the original
source here or elsewhere in the `.md` pipeline).

### 2. Tokenizer: recognize `<br>` and hard breaks

Add a `parse_inline_token_br` parser (self-closing/space/case variants: `<br>`, `<br/>`,
`<br />`, `<BR>`, …) to the `alt((...))` chain in `parse_inline_token`
(`markdown_parser.rs:1529-1549`), following the `parse_inline_token_underline_start`
precedent (`:1622-1631`) for tag matching. Add a hard-break parser that recognizes ≥2
trailing spaces or a trailing backslash immediately before a line ending — this needs to
run at the point where line-ending whitespace is currently just consumed as `Text`
(`whitespace` combinator, `:1526`), so it likely needs to inspect trailing context rather
than slot into the same per-character `alt`; the implementation should look at how
line-ending detection already works in block parsing (`parse_line_ending` usage) to decide
whether this is cleanest as a pre-pass on line text before tokenizing, or a lookahead within
the tokenizer. Both `<br>` and hard breaks produce the same fragment-level break construct
from item 1.

Product invariant 4 (inert inside code spans/code blocks): free — `parse_code_span`
(`:594-609`) already consumes an entire code span as one atomic token before the new
`<br>`/hard-break parsers ever see its contents, matching how all other inline syntax is
already suppressed inside code.

### 3. Table-cell wiring

Because `parse_table_row`/`parse_cell_content` already route every cell through the same
`parse_inline` (`markdown_parser.rs:669` → `:974`), no *tokenizer* wiring is needed for
cells specifically — item 2 covers both automatically (this is the "confirmed, not assumed"
fact from Context). What cells need is item 1's model change reaching the cell-measurement
path: `measure_table_cells` (`edit.rs:260-323`) must stop concatenating every fragment's
`text` into a single `StyledBufferRun`/frame and instead start a new run/line at each break
sentinel, then `CellLayout` (`render/model/mod.rs:1736-1776`, already built to hold multiple
`line_heights`) absorbs the result — this part is genuinely small since the multi-line
*render* plumbing already exists and only needs an authored break fed into it instead of
relying solely on word-wrap.

Whether hard-break syntax (not just `<br>`) is honored inside a cell is a product call
(invariant 3 flags it as likely unreachable syntactically, since `parse_table_cell` rejects
raw `\n`/`\r` in the cell source at `:596-597` — a trailing-backslash hard break is
theoretically typeable before the cell-terminating `|`, but a trailing-double-space hard
break requires a line ending immediately after, which a table cell doesn't have mid-row).
Recommend: implement `<br>`-in-cell (the primary ask, and the only mechanism that's
actually authorable in a pipe-table cell); skip hard-break-in-cell as effectively
unreachable rather than building support for syntax that can't occur.

### 4. Security

`<br>` recognition reads no attributes — any attributes on a malformed `<br class="x">` are
ignored (product invariant on graceful degradation), matching `PHRASING_ELEMENT_TAGS`
handling elsewhere in `html_parser.rs` which already discards non-structural attributes. No
new script/navigation surface; this is a pure text-shape change.

## Testing and validation

### Parser unit tests (`crates/markdown_parser/src/markdown_parser_tests.rs`)

- `<br>`, `<br/>`, `<br />`, `<BR>` mid-paragraph → forced break (invariant 1); verify the
  break sentinel/shape chosen in item 1, not literal text.
- `<br>` inside a GFM pipe-table cell → cell holds two segments (invariant 2).
- Trailing double-space before a line ending, and trailing backslash before a line ending →
  forced break (invariant 3); single trailing space, and mid-line backslash → no break.
- `<br>` and hard-break syntax typed inside a code span/code block → literal text
  (invariant 4).
- `<br><br>` → two breaks, not a paragraph boundary (invariant 5).
- Malformed `<br class="x">`-shaped variants → still recognized, attributes ignored.

### Round-trip (`crates/editor/src/content/text_tests.rs` or equivalent)

- Paragraph with an authored break → serialized form (CommonMark hard break or `<br>`, per
  item 1's recommendation) → re-parsed, break preserved (invariant 6).
- Table cell with `<br>` → internal/export format → re-parsed, break preserved as `<br>`
  (invariant 6), not collapsed and not corrupting the row/column structure of
  `to_internal_format`'s tab/newline-delimited format (the hazard flagged in Context).

### Layout / render tests (`crates/editor/src/render/model/mod_tests.rs`)

- A cell with an authored `<br>` renders ≥2 lines and the row height grows to fit,
  independent of column width (i.e., still breaks at width `f32::MAX` during measurement,
  not only when wrapping kicks in) — the case that most directly exercises the
  `measure_table_cells` change in item 3.
- Existing single-line cells (no `<br>`) are unaffected — no regression to
  `measure_table_cells`/`CellLayout` for the common case.

### Integration / manual

Per CONTRIBUTING, before/after screenshots of the issue's own test case: the standalone
mid-paragraph `<br>` and the GFM pipe-table-cell `<br>`, plus a hard-break paragraph
example. Verify against the issue's motivating screenshot (`<br>` currently rendering as
literal text in both positions).

## Risks and follow-ups

- **The fragment-vs-line model fork (item 1, Option A vs. B) recurs in
  `specs/GH13652/tables/`.** That spec faces the identical decision for HTML-table cells.
  If both land close together, converging on one mechanism is worth a maintainer call
  rather than shipping two independent "cell has 2 lines" representations — flagged here
  and there.
- **Hard-break-in-cell is scoped out** as syntactically unreachable given
  `parse_table_cell`'s existing `\n`/`\r` rejection (`:596-597`); if that rejection is ever
  loosened by a future change, this decision should be revisited.
- **Export serialization choice (CommonMark hard break vs. `<br>`) is a judgment call**,
  not dictated by existing precedent — flagged in item 1 rather than asserted as settled,
  since Warp's export path doesn't currently have to choose between the two for any other
  feature.
