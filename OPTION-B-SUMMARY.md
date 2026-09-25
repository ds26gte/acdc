# `option-B` at a glance

Big-picture summary of what this branch adds over `main`. For repro cases,
root causes, and how to re-verify each item is still needed, see
[`OPTION-B.md`](OPTION-B.md).

## Parser robustness: several classes of valid AsciiDoc no longer crash the whole file

Real-world `.adoc` content routinely hit hard parse errors or silently
wrong output on `main`, in each case because *one specific, common
authoring pattern* wasn't anticipated by a lookahead or boundary check.
Each fix targets one such pattern:

- **Escaped quotes in link titles** (`link:url["a \" quote"]`) no longer
  crash the file.
- **Comment lines starting with `#`** (near-universal in Pyret/Python/shell
  code samples) inside a paragraph or list item no longer get misread as a
  bogus heading, splitting the document's section structure.
- **A list item followed directly by a delimited block** (example,
  listing, sidebar, etc., with no blank line — which AsciiDoc allows) no
  longer produces a bogus "unterminated block" error.
- **An empty table** (`|===` immediately followed by `|===`) now parses as
  an empty table instead of erroring.
- **A comment block right after the document header** no longer gets
  mis-scanned as a one-line comment.
- **A description-list term that wraps across lines** no longer gets
  confused by a `::`-like marker appearing later inside a code sample.

The common thread: acdc's grammar has several places where a cheap
"does this look like X" lookahead gates whether to try the stricter real
rule for X. When the lookahead is looser than the real rule, unusual but
valid content falls into a dead end — nothing left able to parse it. All
of the above are instances of that same failure mode, found empirically by
running acdc against a large real-world curriculum of AsciiDoc content.

## CLI feature: `-B`/`--base-dir`

Adds asciidoctor's `-B`/`--base-dir` flag to `acdc convert` — lets
relative `include::`/`image::` targets resolve against a directory other
than the entry file's own parent. The parser already supported this
internally; the CLI just never exposed it.

## Already caught up by `main` (informational — no longer part of this branch's value-add)

Two early fixes in this branch's history (blank lines around block
metadata, and an index-term lookahead dead end) have since been
independently and more completely fixed upstream. See `OPTION-B.md` for
details if useful as a reference.
