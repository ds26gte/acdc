# What `option-B` does over and above `main`

This branch exists to close a set of real, empirically-verified compatibility
gaps between `acdc` and `asciidoctor` that surfaced while running `acdc`
against a large, real-world AsciiDoc corpus (the [Bootstrap curriculum
repo](../curriculum)). `main` moves fast and has independently absorbed two
of these already (see "Already absorbed by main" below) — expect more of
this list to become redundant over time. **Before re-implementing anything
here, re-verify it's still needed** using the method below; don't just
cherry-pick the old diffs, which have already twice conflicted badly with
main's own independent (and sometimes better) fixes to the same code.

## How to re-verify this list at a future date

For each item below there's a minimal `.adoc` repro. To check whether it's
still needed:

```bash
# Build main alone in a throwaway worktree, isolated from any local changes.
git worktree add /tmp/acdc-main-check main
cd /tmp/acdc-main-check && cargo build --release -p acdc-cli

# Run the repro through main's own binary.
/tmp/acdc-main-check/target/release/acdc convert -e -o - repro.adoc

# Compare against real asciidoctor — it's the actual reference, not option-B.
asciidoctor -e -o - repro.adoc

# Clean up when done.
cd - && git worktree remove /tmp/acdc-main-check --force
```

**Do not trust exit codes alone.** Several of these bugs "look fixed" on
main (no warning, no crash) while actually producing *silently wrong*
output — e.g. a list item gets split by a bogus heading instead of staying
literal text, or an index-term match swallows/mangles content. Always diff
the actual rendered output (`-e -o -`) against real `asciidoctor`, not just
check the exit status.

If main's output now matches asciidoctor's, the fix is redundant — drop it
and let main's version win any merge conflict. If main still diverges from
asciidoctor, the fix is still needed — but consider re-implementing it
against main's *current* code shape rather than porting the old diff (see
"Notes for a more concise re-implementation" at the end).

## Status summary

| # | Fix | Status as of this doc |
|---|---|---|
| 1 | Backslash-escaped quotes in link titles | **Still needed** |
| 2 | List item + heading-shaped `#` comment, no blank line | **Still needed** |
| 3 | List item immediately before a delimited block | **Still needed** |
| 4 | Paragraph + heading-shaped `#` comment, no blank line | **Still needed** |
| 5 | Empty table (adjacent `|===`/`|===`) | **Still needed** |
| 6 | Comment block immediately after document header | **Still needed** |
| 7 | Description-list term scan crossing a delimited block | **Still needed** |
| 8 | `-B`/`--base-dir` CLI flag | **Still needed** (feature, not a bug fix) |
| 9 | Block metadata + blank line before its target | ~~Needed~~ — **superseded by main** |
| 10 | Index-term lookahead dead end | ~~Needed~~ — **superseded by main**, and main's version is *more* correct than ours was |

---

## Still needed

### 1. Backslash-escaped quotes in link titles

**Symptom:** hard parse error (fails the *entire file*), not a warning.

**Repro:**
```asciidoc
link:pass:[https://example.com]["quoted \" title"]
```

**Root cause:** the link-macro title rule's double-quoted branch
(`link_title()` in `acdc-parser/src/grammar/inlines.rs`, as of this writing)
scans for the first literal `"` to close the title, with no awareness of
`\"` escaping. It stops at the escaped quote's `"` character, truncating
the title early; the text left over then fails to match the macro's
closing `]`, and since the macro's own lookahead-match rule (used to decide
whether to commit to trying it at all) is more permissive than the real
rule, there's no fallback — a dead end.

**What asciidoctor actually does:** keeps the backslash in the rendered
title (`asciidoctor renders <a>...quoted \" title...</a>`, not an unescaped
`"`). Our original fix (`dbfc4f3c`) matched everything except that last
detail — a known, documented gap at the time (see that commit's changelog
entry). Worth fixing properly (strip the backslash) rather than just
porting the old behavior forward.

### 2 & 4. Heading-shaped `#` line ends a paragraph/list item without a blank line

These two are the same underlying issue in two different grammar rules —
treat them as one when re-implementing.

**Symptom:** no warning, but silently wrong structure. A paragraph or list
item containing an ordinary comment line starting with `#` (very common in
Pyret/Python/shell code samples) gets split there, and the `#` line is
reparsed as a bogus level-0 ATX-style section — corrupting the section tree
for everything that follows, which can surface much later in the document
as an unrelated-looking "section title out of sequence" warning.

**Repro (paragraph):**
```asciidoc
Here's an example:
```
# a Pyret comment, not a heading
fun f(x): x
```
```

**Repro (list item):**
```asciidoc
- item one
# not a heading, just a comment line
- item two
```

**What asciidoctor actually does:** treats the `#`/`=` line as ordinary
continuation text unless a genuine blank line precedes it. A single line
break is not enough.

**Root cause:** the heading-boundary check in each rule's continuation-line
exclusion list had no blank-line requirement at all — any `=`/`#`-shaped
line ended the scan immediately.

**A subtlety that broke this exact fix during the `main` merge**: don't
conflate "stop at a *bare* heading" (needs a real blank line) with "stop at
an *attribute/anchor metadata line, then* a heading" (does *not* need a
blank line before the metadata itself — see item 9's discussion). These
are genuinely different cases needing different blank-line requirements;
collapsing them into one shared check breaks one or the other. Use two
separate alternatives, not one.

### 3. List item immediately before a delimited block

**Symptom:** `unterminated example block` (or listing/literal/etc.)
warning, anchored at the block's real *closing* delimiter — misleading,
since the actual problem is earlier.

**Repro:**
```asciidoc
- second item
[.role]
=====

Example block content

=====
```

**Root cause:** a list item's continuation-line scan had no check for a
following delimited block (optionally preceded by a role/attribute line).
Unlike a heading, asciidoctor does *not* require a blank line before a
delimited block — so without this check, the block's opening delimiter
gets swallowed as ordinary list text, leaving the real closing delimiter
to be misparsed as a fresh, unterminated block.

**Fix shape:** a lookahead recognizing "optional attribute/anchor lines,
then any delimited-block opener" (all delimiter types: example, listing,
literal, sidebar, quote, pass, table, markdown-code, comment, open), added
to the list continuation-line exclusion list.

### 5. Empty table (adjacent `|===`/`|===`)

**Symptom:** `unterminated table block` warning; everything after the
table gets swallowed as mis-parsed table content.

**Repro:**
```asciidoc
[cols=1]
|===
|===
```

**Root cause:** the table content-scanning rule's stop-lookahead only
fires *after* crossing at least one newline — so when the closing
delimiter is on the very next line with nothing in between, the scan never
gets a chance to recognize it as a boundary and instead starts consuming
its characters as ordinary cell content.

### 6. Comment block immediately after the document header

**Symptom:** `unterminated comment block` warning.

**Repro:**
```asciidoc
= Title

////
comment content
////

Some text.
```

**Root cause:** header-trivia-skipping logic mistook the `////`-block
opener for a one-line `//` comment while scanning past the header,
swallowing just the delimiter and leaving the rest to be misparsed.

### 7. Description-list term scan crossing a delimited block

**Symptom:** `unterminated <X> block` warning, again anchored at the real
closing delimiter, not the actual problem site.

**Repro:**
```asciidoc
Here's an example:
```
species-dot :: (Row) -> Image
fun species-dot(r): 1
```
```

**Root cause:** a description-list term can legitimately wrap across
multiple lines before its `::`/`;;` marker (asciidoctor allows this), so
the term-scanning rule scans forward across newlines looking for the
marker — but with no check for delimited-block boundaries. A `::` inside
a *later* fenced code sample (e.g. a Pyret type annotation comment,
extremely common) gets mistaken for the current paragraph's dlist marker,
swallowing that block's opening delimiter as term text.

**Fix shape:** same delimiter-boundary exclusion list as item 3, added to
the term-scanning rule's own stop condition.

### 8. `-B`/`--base-dir` CLI flag

Not a bug fix — a missing CLI feature. `acdc-parser`'s `Options` already
has `with_base_dir()`; `acdc-cli`'s `convert` subcommand never exposed it.
Needed to match `asciidoctor -B`/`--base-dir`: overrides the entry file's
parent directory as the resolution root for relative `include::`/`image::`
targets. Trivial to re-add if main still lacks it — just plumb the
existing parser option through to a new CLI arg.

---

## Already absorbed by main — no longer needed

### 9. Block metadata + blank line before its target

Asciidoctor allows a blank line between an attribute/anchor/title line and
the block it applies to (verified directly: `[.role]` then a blank line
then a list still applies `role` to that list). Main's `427107cf` fixes
this comprehensively — more generally than our original fix did (it
tolerates blank lines around *every* metadata line, not just the last one
before the target), and its output for the role-application case matches
asciidoctor exactly, which our own original fix never fully achieved.

**If re-deriving from scratch, model it on main's shape, not ours**: a
dedicated `attribute_or_anchor_line_match()` lookahead-only rule (handles
`[[id]]` double-bracket anchors explicitly, unlike the bare
`anchor()`/`attributes_line()` pair we originally used) is the reusable
building block worth keeping.

### 10. Index-term lookahead dead end

**Repro:** `` `(((a)(a)))` `` (monospace content containing something that
*looks* like but isn't a valid index term).

Main's `6cc09ccd`/`6a0094a0` fix this, and — important — main's resulting
behavior actually matches real `asciidoctor` (`<code></code>`, i.e. the
whole thing is swallowed as a concealed index term) where our original fix
did not (we rendered `<code>(a)(a)</code>`, avoiding the crash but
diverging from the reference). Don't resurrect the old approach even as a
starting point; main's is simply more correct.

---

## Notes for a more concise re-implementation

A few patterns recurred across these fixes that suggest a *more general*
fix would be smaller and more robust than the sum of these individual
patches:

1. **Lookahead/real-rule mismatch dead ends.** Several bugs (index-term,
   link-title, and in spirit the delimited-block ones) share one shape: a
   cheap lookahead rule decides whether to *commit* to trying a real,
   stricter parsing rule; when the lookahead is more permissive than the
   real rule and the real rule then fails, nothing else can consume that
   position, and the resulting error is misleading (points at some
   unrelated later position, not the actual mismatch). Worth checking
   *every* lookahead/real-rule pair in the grammar for this asymmetry
   rather than fixing them one at a time as they're discovered.

2. **Blank-line tolerance needed re-deriving separately in four unrelated
   places** (block metadata, paragraphs, list-item continuations,
   description-list terms) — each with its own copy of similar-but-subtly-
   different logic, and each capable of breaking the others when merged
   with upstream changes to the same area (this happened twice in one
   merge — see items 2/4's "subtlety" note and the `at_section_start()`
   auto-merge silently dropping a check with no conflict marker). A single
   shared combinator — something like `at_boundary(requires_blank_line:
   bool)` — parameterized per call site, might collapse all four into one
   well-tested implementation instead of four independently-evolving ones.

3. **Auto-merge is not a substitute for testing.** More than once during
   merges into this branch, git resolved a hunk with *no* conflict marker
   at all — because only one side had touched that exact line — yet the
   result was still wrong, because the *other* side's related-but-
   non-overlapping change assumed the removed/changed line was still
   there. Always run the full test suite after a merge, not just resolve
   the flagged conflicts; the dangerous regressions are in the
   unconflicted regions.
