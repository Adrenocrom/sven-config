---
name: markdown_citations
description: 'How to express citations in Markdown: footnotes, Pandoc @key citations, inline links, and HTML cite tags'
tags:
- markdown
- citations
- footnotes
- pandoc
- academic-writing
created_at: '2026-09-24T14:01:15.174029708+02:00'
---

# Citations in Markdown

Markdown has no native citation syntax. Options by use case:

## 1. Footnotes (GitHub, Pandoc, PHP Markdown Extra)
```markdown
Some claim[^1].

[^1]: Smith, J. (2020). *Title*. Publisher.
```
Renders as clickable superscript; reference list appears at bottom. Supported natively on GitHub.

## 2. Pandoc citation keys (academic writing)
Requires a .bib file (BibTeX/CSL JSON) and `--citeproc`:
```markdown
Effect shown [@smith2020, p. 15].
Multiple sources [@smith2020; @jones2021].
```
Pandoc formats citations per style (APA, IEEE...) and auto-generates the bibliography. Prefixes, suffixes, and page numbers go inside the brackets.

## 3. Inline links
```markdown
According to [the spec](https://www.w3.org/), ...
```

## 4. HTML <cite> tag
```markdown
As stated in <cite>Smith 2020</cite>, ...
```
Works in most Markdown flavors that allow inline HTML.

## Caveat
`> blockquote` is NOT a citation — it only renders an indented quote block with no citation semantics.

## Recommendation
- General docs: footnotes.
- Academic papers with bibliography: Pandoc `[@key]` + citeproc.
