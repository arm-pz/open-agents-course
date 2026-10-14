# 003 — Design system, measured from the category

Step: gstack `design-consultation`, producing `DESIGN.md`, `CLAUDE.md` and `design/preview.html`.

## What we decided first

Two questions, answered before any visual work: research the landscape first rather than propose from
memory, and the one thing the course should be memorable for is **"Every claim has a source you can
click."** Everything below is chosen to serve that sentence.

## How the research was actually done

`WebFetch` returns converted markdown — it strips CSS, so any font or colour claim made from it would
have been a guess. So the evidence came from the browser reading **computed styles** off live pages.

Tool calls (MCP `browser-use`, via `mcp_call`):

```
navigate_page -> https://huggingface.co/learn/agents-course/unit1/introduction
evaluate_script -> getComputedStyle on body/h1/code/pre/a + documentElement custom properties
navigate_page -> https://platform.claude.com/docs/en/home        (docs.anthropic.com 301s here)
evaluate_script -> same probe + set of distinct font families in use
navigate_page -> https://docs.langchain.com/
evaluate_script -> same probe
navigate_page -> https://cookbook.openai.com/                    (redirects to developers.openai.com/cookbook)
evaluate_script -> same probe + histogram of link colours across 60 links/buttons
```

Real output, trimmed to the fields that matter:

```json
{ "title": "Introduction to Agents · Hugging Face",
  "bodyFont": "\"Source Sans Pro\", -apple-system, ...",
  "bodySize": "16px", "bodyColor": "oklch(0.928 0.006 264.531)", "bodyBg": "rgb(11, 15, 25)",
  "h1Size": "18px", "h1Weight": "700",
  "codeFont": "\"IBM Plex Mono\", ui-monospace, ...", "codeSize": "14px" }

{ "url": "https://platform.claude.com/docs/en/home",
  "bodyFont": "anthropicSans, ... system-ui, ...", "bodySize": "14px",
  "bodyColor": "rgb(240, 239, 236)", "bodyBg": "rgb(21, 21, 21)",
  "heading": { "font": "anthropicSerif, ... serif", "size": "52px", "weight": "300" },
  "code": { "font": "anthropicMono, ...", "size": "13px" } }

{ "title": "Home - Docs by LangChain",
  "bodyFont": "Inter, sans-serif", "bodySize": "16px", "bodyColor": "rgb(255, 255, 255)",
  "heading": { "font": "\"TWK Lausanne\", sans-serif", "weight": "700" },
  "code": { "font": "\"IBM Plex Mono\", monospace", "size": "14px" },
  "fontFamiliesUsed": ["\"TWK Lausanne\"", "Inter", "\"IBM Plex Mono\""] }

{ "url": "https://developers.openai.com/cookbook",
  "bodyFont": "\"OpenAI Sans\", sans-serif", "bodySize": "16px", "bodyBg": "rgb(0, 0, 0)",
  "heading": { "font": "\"OpenAI Sans\", sans-serif", "size": "16px", "weight": "400" },
  "code": { "font": "\"SF Mono\", SFMono-Regular", "size": "16px" },
  "linkColorHistogram": { "rgb(255, 255, 255)": 17, "rgb(185, 185, 185)": 17,
                          "rgb(220, 220, 220)": 23, "rgb(143, 143, 143)": 1, "rgb(13, 13, 13)": 2 } }
```

Three findings, all useful:

1. **Every peer defaults to dark**, backgrounds from `rgb(0,0,0)` to `rgb(21,21,21)`.
2. **Links are greys, not colours.** The OpenAI histogram is the clearest case: the top link colours on
   the page are `#FFFFFF`, `#DCDCDC`, `#B9B9B9`, `#8F8F8F`. Nobody in the category distinguishes
   "this goes to a source" from "this goes to another docs page".
3. **Mono is settled.** IBM Plex Mono on two of four peers, SF Mono / anthropicMono on the others. So
   code is a safe choice, not a place to spend a risk.

## The claim worth checking instead of asserting

"Nobody cites sources in prose" is the whole premise of the course, so it got measured rather than
assumed. Second `evaluate_script` on the HF unit1 page, counting citation markers inside `main`:

```json
{ "paragraphs": 18, "superscriptRefs": 0, "paragraphsWithBracketRefs": 0,
  "externalLinksInProse": 2, "externalLinkTexts": ["33,560", "Update on GitHub"],
  "hasSourceSection": [] }
```

And on `docs.langchain.com/oss/python/langchain/agents`, scoped per container because the first probe
returned almost nothing:

```json
{ "totalP": 8,
  "main":  { "p": 2, "sup": 0, "bracket": 0, "ext": 15 },
  "body":  { "p": 8, "sup": 0, "bracket": 0, "ext": 38 } }
```

18 paragraphs of flagship course prose with zero references; the two outbound links are a star counter
and an edit button. The LangChain page has 38 external links in `body` and 0 citation markers — all of
that is navigation, none of it is provenance. Two peers measured, both 0, so the differentiation claim
is supported to the depth we actually checked. Not claimed beyond that.

## What followed from the evidence

The category baseline is dark + neutral + settled mono. Coherence is free there; every peer has it and
every peer looks the same. So the risks were spent on the differentiator instead:

- **Light warm paper as default.** Reads as a book, and a book is what a sourced text looks like.
- **Serif body prose** (Source Serif 4). Precedent exists only for serif *display* (Anthropic, 52px/300).
- **Colour reserved for provenance** — teal `#12655F` means "this traces somewhere" and nothing else.
  A page full of accent is a page full of citations, so the marketing claim becomes visually verifiable.
- **Sources rail where peers put the table of contents.**

Safe choices kept deliberately safe: dark mode exists, IBM Plex Mono for code and tabular data, left
module nav, 1px-rule grid, contrast well above 4.5:1.

## Files written

- `DESIGN.md` — the system, plus the measured peer table so the next agent does not re-guess it.
- `CLAUDE.md` — repo guidance with the colour-reservation rule stated as an invariant.
- `design/preview.html` — self-contained specimen: type specimens, palette, components, a full lesson
  page with citation chips and sources rail, and a corpus-coverage table. Light/dark toggle.

## Commands

```
# no gstack designer CLI on this machine (verified in journal 002: NO_BUN, DESIGN_NOT_AVAILABLE),
# so Phase 5 ran the HTML-preview path rather than AI mockups.
```

## Lesson

A design doc written from memory of what docs sites look like would have said "dark, monospace, blue
links" — and blue links is exactly the pattern that fails to separate a source from a nav item.
Reading computed styles off the real pages is what turned a vibe into a rationale: the accent rule only
makes sense once you can see that four of four peers use pure grey for every link.

Second lesson: measure your own differentiator before you sell it. The citation count took one extra
`evaluate_script` and is now the strongest line in `DESIGN.md`.
