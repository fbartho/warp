# TECH.md — Markdown viewer: `<kbd>` keycap styling

Product spec: `specs/GH13733/product.md`
GitHub issue: https://github.com/warpdotdev/warp/issues/13733

## Context

`<kbd>` has **zero handling anywhere in the workspace** today. It's absent from
`markdown_parser.rs`'s inline tokenizer and from `html_parser.rs`'s `PHRASING_ELEMENT_TAGS`/
`TOP_LEVEL_ELEMENT_TAGS_TO_SKIP` lists. The only `kbd`-adjacent hits in the codebase are
`LMGetKbdType`/doc-comment prose in `crates/computer_use/src/mac/keycode_cache.rs`
(physical-keyboard handling, unrelated).

Two independent inline-HTML pipelines both need the new tag, mirroring how `<u>` (underline)
is already handled in both:

**1. File-viewer Markdown grammar — `crates/markdown_parser/src/markdown_parser.rs`.**
Underline is a first-class delimiter pair, not a generic tag mechanism:

- `DelimiterKind::UnderlineStart` is produced by `parse_inline_token_underline_start`
  matching the literal `tag("<u>")` (`:1626-1636`), and `InlineToken::UnderlineEnd` by
  `parse_inline_token_underline_end` matching `tag("</u>")` (`:1638-1646`).
- `<u>` pushes a `Delimiter` onto `InlineState`'s delimiter stack like `*`/`_`/`~~`; `</u>`
  triggers `parse_underline` (`:1274-1308`), which finds the matching opener via
  `DelimiterKind::UnderlineStart` on the stack, calls
  `state.backtrack_styles(underline_start_node, |styles| styles.underline = true)`, then
  `process_emphasis` to resolve interleaving with other delimiters (bold/italic/strike),
  and removes the delimiter/node.
- **Fallback:** if `</u>` has no matching opener, `parse_underline` pushes the literal text
  `"</u>"` and returns (`:1281-1285`) — no panic, no swallowed content. If a start delimiter
  is inactive (already closed once, preventing nesting), it's dropped and `</u>` renders
  literal too (`:1287-1292`).
- Serialization back to Markdown source emits `DelimiterKind::UnderlineStart.as_str()` →
  `"<u>"` (`:1835`) when round-tripping a styled run back to text.

**2. Paste-to-Markdown / HTML-fragment path — `crates/markdown_parser/src/html_parser.rs`.**
This path DOM-walks pasted HTML (via `html5ever`) and maps recognized tags onto a `Styling`
struct, not delimiter tokens:

- `PHRASING_ELEMENT_TAGS` (`:26-28`) currently lists `span, i, code, strong, em, br, a, s,
  u, ins` — these are queued as `pending_inline_nodes` (`:245-251`) rather than treated as
  block boundaries.
- `parse_phrasing_content` (`:410`) walks queued nodes and switches on tag name to flip
  boolean fields on a `decorated_styling: Styling` value: `"s" => strikethrough = true`,
  `"u" | "ins" => underline = true`, `"code" => inline_code = true` (`:444-446`), with an
  explicit `_ => ()` fallthrough plus a `TODO` marking this switch as intentionally
  incomplete (`:447-449`, referencing `CLD-335`).

**Shared destination — `FormattedTextStyles`, `crates/markdown_parser/src/lib.rs:545-552`:**

```rust
pub struct FormattedTextStyles {
    pub weight: Option<CustomWeight>,
    pub italic: bool,
    pub underline: bool,
    pub strikethrough: bool,
    pub inline_code: bool,
    pub hyperlink: Option<Hyperlink>,
}
```

Both pipelines ultimately set boolean/optional fields on this struct per inline run — there
is no per-tag styling struct beyond this shared one.

**Render precedent — inline code's visual treatment**, the closest existing "chip" look
(`crates/editor/src/render/layout.rs:165-214`, `style_and_font`):

```rust
if text_styles.is_inline_code() {
    styling = styling
        .with_foreground_color(self.rich_text_styles.inline_code_style.font_color)
        .with_background_color(self.rich_text_styles.inline_code_style.background)
        .with_border(TextBorder {
            color: self.rich_text_styles.inline_code_style.background,
            radius: 4,
            width: 1,
            line_height_ratio_override: Some(120),
        });
}
```

driven by `font_family` selection a few lines above (`:171-175`, swaps to
`rich_text_styles.inline_code_style.font_family` — the monospace font — whenever
`is_inline_code()`), and by `InlineCodeStyle` (`crates/editor/src/render/model/mod.rs:1211-1215`):

```rust
pub struct InlineCodeStyle {
    pub font_family: FamilyId,
    pub background: ColorU,
    pub font_color: ColorU,
}
```

A second, independent code path (`markdown_inline_to_text_and_style_runs`, `layout.rs:247-313`,
used in tests/other call sites) also branches on `fragment.styles.inline_code` (`:281-285`)
to apply a background — a second site that would need the analogous `kbd` branch if `kbd`
gets its own flag rather than reusing `inline_code`.

This confirms the issue's read: **`inline_code`'s existing monospace + background + border
treatment is the nearest visual precedent for a keycap**, and the "add a delimiter pair
mapping to a style flag" mechanism `<u>` uses is the nearest structural precedent for
`<kbd>`.

## Proposed changes

### 1. Style representation: new flag vs. reuse `inline_code`

Two options, both minimal:

- **Option A (recommended): add `pub kbd: bool` to `FormattedTextStyles`.** Keeps `<kbd>`
  semantically distinct from `<code>` (they're different HTML elements with different
  meaning) while allowing the renderer to give it the *same or a lightly-differentiated*
  visual treatment. This is what lets a future visual tweak (e.g. slightly different corner
  radius or a "raised" bottom border to read as a physical key) happen without touching
  `<code>`. Round-trip serialization emits a `<kbd>`/`</kbd>` pair distinct from backticks.
- **Option B: reuse `inline_code: bool` directly for `<kbd>`.** Zero new struct field, but
  conflates two distinct HTML semantics into one flag — round-tripping a `<kbd>`-sourced
  run back to Markdown would need to remember it came from `<kbd>` rather than `` ` ``
  somehow, which the current model can't do (the flag alone doesn't carry provenance). This
  breaks product invariant 5 (copy/export preserves the tag) and is **not recommended**.

Recommend Option A. The new field is a one-line struct addition plus the same handful of
call sites `underline`/`strikethrough` already touch (style application, style-run
diffing/equality, any `Default`-derived construction — `FormattedTextStyles` likely derives
`Default`/`PartialEq`, so the new field needs no special-casing there).

### 2. File-viewer grammar (`markdown_parser.rs`)

Mirror the `<u>`/`</u>` delimiter pair exactly:

- Add `DelimiterKind::KbdStart` (or reuse `UnderlineStart`'s shape generically — but a
  distinct variant is clearer and matches how `Underline` itself is distinct from
  `Strikethrough` rather than sharing a variant).
- `parse_inline_token_kbd_start`: `tag("<kbd>")` → `InlineToken::Delimiter { kind:
  DelimiterKind::KbdStart, count: 1 }`, alongside the existing
  `parse_inline_token_underline_start` (`:1626-1636`).
- `InlineToken::KbdEnd` for `tag("</kbd>")`, parsed the same way as
  `parse_inline_token_underline_end` (`:1638-1646`).
- `parse_kbd` mirroring `parse_underline` (`:1274-1308`) line for line: find the innermost
  active `KbdStart` on the delimiter stack, fall back to literal `"</kbd>"` text if none
  (invariant 4), else `backtrack_styles(node, |styles| styles.kbd = true)`,
  `process_emphasis`, remove delimiter/node.
- Wire both new tokens into whatever combinator currently lists
  `parse_inline_token_underline_start`/`_end` among the inline token alternatives, and add
  `DelimiterKind::KbdStart.as_str() → "<kbd>"` to the same match arm as `:1835` for
  round-trip serialization (invariant 5).
- `process_emphasis`'s `can_open`/`can_close` rules (`:1743-1759` for `UnderlineStart` today)
  need a `KbdStart` arm; since `<kbd>` is a tag-delimited pair like `<u>` and not a
  run-counted delimiter like `*`/`_`, it should follow `UnderlineStart`'s existing
  `left_flanking`/`right_flanking` treatment verbatim (`:1751`).

### 3. Paste path (`html_parser.rs`)

- Add `"kbd"` to `PHRASING_ELEMENT_TAGS` (`:26-28`), alongside `s`/`u`/`ins`/`code`.
- Add a `"kbd" => decorated_styling.kbd = true` arm in `parse_phrasing_content`'s tag switch
  (`:444-449`), next to the existing `"code" => decorated_styling.inline_code = true` line.

### 4. Render

- In `style_and_font` (`crates/editor/src/render/layout.rs:165-214`), add a
  `text_styles.is_kbd()` (or equivalent accessor, matching whatever `is_inline_code()`'s
  pattern is) branch. MVP: apply the **same** `TextBorder`/background/font-family treatment
  `is_inline_code()` uses, reusing `inline_code_style` as the source of truth so there is
  zero new theming surface for this small a feature. If product/design later wants visual
  differentiation from code spans (e.g. a "raised" look), that's a follow-up that swaps in
  a dedicated `kbd_style` token — not required for this slice.
- Add the parallel branch to `markdown_inline_to_text_and_style_runs`
  (`layout.rs:247-313`, `:281-285`) for the code path that reads `fragment.styles.kbd`
  directly, mirroring the `inline_code` branch there.
- No layout/measurement changes needed: `<kbd>` is inline text like `<u>`/`` `code` ``, so
  it flows through the existing text-layout/wrap machinery unchanged. No new block type, no
  new `FormattedTextLine` variant.

### 5. Security

`<kbd>` contributes no new attribute surface — content is parsed as ordinary inline
Markdown/phrasing text (bold/italic/code/links can nest inside it per product invariant 2),
inheriting the viewer's existing trust boundary. No attributes on the `<kbd>` tag itself are
read or need to be; any present (`class`, `id`, etc.) are ignored exactly as other phrasing
tags' non-semantic attributes already are.

## Testing and validation

### Parser unit tests

**`markdown_parser_tests.rs`** (mirroring existing `<u>` test cases):

- `<kbd>Cmd</kbd>` → single fragment/run with `styles.kbd == true` (product invariant 1).
- The issue's test case, `Press <kbd>Cmd</kbd>+<kbd>K</kbd> to open the command palette.` →
  five runs: plain "Press ", kbd "Cmd", plain "+", kbd "K", plain " to open the command
  palette." (invariant 3).
- `**<kbd>Cmd</kbd>**` and `<kbd>**Cmd**</kbd>` → both `weight` and `kbd` set on the same
  run (invariant 2), matching whatever existing test asserts `<u>` + bold compose.
- Unmatched `</kbd>` (no opener) → literal text `"</kbd>"` emitted, no panic (invariant 4).
- Round-trip: styled run with `kbd == true` serializes back to `<kbd>…</kbd>` source
  (invariant 5).

**`html_parser_tests.rs`** (paste path):

- Pasted `<kbd>Cmd</kbd>` HTML fragment → `FormattedTextStyles.kbd == true` on the resulting
  run, matching the existing `<code>`/`<u>` paste tests' shape.

### Render tests

- A snapshot/style-run test confirming a `kbd`-styled fragment picks up the same
  `TextBorder`/background/font-family as an `inline_code` fragment (or the dedicated
  treatment, if design differentiates it) — extending whatever existing test covers
  `style_and_font`'s `is_inline_code()` branch.

### Integration / manual

Per CONTRIBUTING, a before/after screenshot rendering the issue's motivating test case
(`Press <kbd>Cmd</kbd>+<kbd>K</kbd> to open the command palette.`), plus a case nesting
`<kbd>` inside a link and inside bold text to confirm composition.

## Risks and follow-ups

- **Genuinely small.** One new `FormattedTextStyles` field, one delimiter pair mirroring
  `<u>` in the typed grammar, one tag-switch arm in the paste path, one render branch
  reusing `inline_code`'s existing style tokens. No new data model, no new block type, no
  new attributes, no layout changes. This matches the issue's own priority framing (1/5)
  and its role as the smallest split from #13652.
- **Visual differentiation from `<code>` is an explicit non-goal for this slice** (product
  spec) but the `kbd: bool` flag (Option A) keeps that door open without extra cost now —
  the alternative (reusing `inline_code` directly, Option B) forecloses it and breaks
  round-trip fidelity, so it's not recommended even though it saves one field.
- **Two independent pipelines must both change.** It would be easy to land only the
  file-viewer grammar change and forget the paste path (or vice versa) — the product spec's
  invariant 1 covers "the viewer," but parity with how `<u>`/`<code>` behave in *both*
  paths is the bar; call this out explicitly in review.
