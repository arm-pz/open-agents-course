# Design System — open-agents-course

## Product Context
- **What this is:** a free, openly licensed, source-cited course on building AI agents. One module pool, three learner routes.
- **Who it's for:** non-coders who use AI tools, working developers, and experienced agent builders.
- **Space/industry:** agent education and technical documentation. Measured peers: Hugging Face Agents Course, Anthropic docs, LangChain docs, OpenAI Cookbook.
- **Project type:** editorial learning platform (long-form prose plus runnable code), not an app dashboard.

## The one rule
**Colour is reserved for provenance.** On a lesson page the only saturated element is a source
chip or a source link. If a page is full of teal, that is a claim about how well it is sourced —
it is readable as evidence, not decoration.

## Aesthetic Direction
- **Direction:** scholarly print — a sourced textbook, not a docs console.
- **Decoration level:** minimal. Typography and 1px rule lines carry structure; one soft shadow on elevated cards only.
- **Mood:** calm, trustworthy, printed. It should feel like material someone checked, and like something you would highlight in a book.
- **Dark mode is available, not the default.** Every peer in the category defaults to dark; this is a deliberate departure (see Design Decisions).

## Typography
- **Display/Hero:** Fraunces (300 for page titles, 500 for module and section heads) — a serif with optical sizing, so a 52px title and a 20px wordmark both look drawn rather than scaled.
- **Body:** Source Serif 4, 400/600 — lesson prose. Serif body is the deliberate risk; it puts citation chips in their native register.
- **UI/Labels:** Geist, 400/500/600 — nav, kickers, buttons, form labels, status pills. Uppercase kickers at 11–12px, `letter-spacing: .12em`.
- **Data/Tables:** IBM Plex Mono with `font-variant-numeric: tabular-nums` — counts must align.
- **Code:** IBM Plex Mono, 13px/1.6.
- **Loading:** Google Fonts, one `<link>` for all four families. Fallbacks: `Georgia, serif` for the serifs, `system-ui, sans-serif` for Geist, `ui-monospace, Menlo, monospace` for Plex.
- **Scale:** hero `clamp(2.6rem, 6vw, 4.4rem)` · h2 32px · h3 20px · body 17px/1.65 · small 15px · label 13px · kicker 12px · code 13px · chip 12px.

## Color
- **Approach:** restrained — one accent, a warm neutral ramp, semantic colours quarantined to tool output.
- **Source accent:** `#12655F` (light) / `#4FB3A5` (dark, saturation reduced) — source chips, source links, the citation rail. Nothing else uses it.
- **Neutrals (light):** paper `#F7F4EE` · panel `#FFFDF9` · rule-soft `#EDE7DE` · rule `#E2DBD0` · muted `#6B655C` · ink `#17140F` · code-bg `#F1EDE5`.
- **Neutrals (dark):** paper `#14120F` · panel `#1B1815` · rule-soft `#221F1A` · rule `#2C2823` · muted `#9C9589` · ink `#EAE5DC` · code-bg `#1E1B17`.
- **Semantic:** success `#2F7D4F` / `#5BA77A`, warning `#9A6B00` / `#C79A3E`, error `#B3261E` / `#E0736A`. Used only in `gate.md` results and `tools/check-*.py` output, never as page ornament, so they cannot be confused with the accent.
- **Dark mode strategy:** shift surfaces rather than recolouring them; ink becomes warm off-white, the accent lightens and loses ~15% saturation, shadows drop out entirely.

## Spacing
- **Base unit:** 8px.
- **Density:** comfortable (reading, not scanning).
- **Scale:** 2xs 4 · xs 8 · sm 12 · md 16 · lg 24 · xl 32 · 2xl 48 · 3xl 64.
- **Section rhythm:** 64px vertical padding, separated by a 1px rule instead of a gap or shadow.

## Layout
- **Approach:** hybrid — editorial three-rail for lessons, grid-disciplined for index and corpus pages.
- **Lesson grid:** 200px module nav · fluid prose · 240px sources rail, 32px gaps; collapses to one column below 1000px.
- **Max content width:** 1120px page, 66ch prose measure.
- **Border radius:** controls 4px, cards 6px, status pills 9999px. Nothing else is rounded.

## Motion
- **Approach:** minimal-functional.
- **Specifics:** a source chip swaps to the accent fill on hover; chip hover may reveal the source title; entrance transitions are a 120ms opacity fade. No scroll-driven effects, no parallax, no animated hero.
- All motion is suppressed under `prefers-reduced-motion: reduce`.

## Component notes
- **Source chip:** `[source-id]` rendered as a mono superscript-style chip — `IBM Plex Mono` 12px, `padding: 3px 6px`, accent text on `#DCEAE7` (light) / `#122826` (dark), 3px radius, `vertical-align: .18em`. Links to the module's `sources.md` entry, which links to the original.
- **Sources rail:** right column of a lesson page; lists the source ids used above it with the full title underneath in 11px serif muted. It replaces the conventional table of contents.
- **Card:** panel background, 1px rule border, 6px radius, single soft shadow.
- **Buttons:** primary is ink-filled with paper text; secondary is transparent with a rule border; ghost is accent text with no border. Never gradient, never full-pill.
- **Callouts:** 3px left border in the semantic colour, 8% tint of the same hue on the panel.

## Accessibility
- Body text on paper is `#17140F` on `#F7F4EE` — well above 4.5:1.
- The accent carries meaning, so it is never the only signal: every chip also contains readable text (`[source-id]`), and gate results pair colour with words.
- Focus states use the accent outline at 2px; hit targets are 36px minimum.

## Anti-patterns for this project
No purple or gradient accents, no gradient buttons, no 3-column feature grid with icons in coloured circles, no centred-everything layout, no uniform bubbly radius, no stock-photo hero, no `system-ui` as a display or body face, no "Built for X" copy patterns. Inter, Roboto, Poppins, Montserrat and Space Grotesk are not used as primary faces.

## Design Decisions (measured, not assumed)
Peer styles were pulled from live pages via computed styles on 2026-10-13, not recalled:

| Peer | Body | Heading | Mono | Default bg |
|---|---|---|---|---|
| HF Agents Course | Source Sans Pro 16px | 18px/700 | IBM Plex Mono 14px | `rgb(11,15,25)` dark |
| Anthropic docs | anthropicSans 14px | anthropicSerif 52px/300 | anthropicMono 13px | `rgb(21,21,21)` dark |
| LangChain docs | Inter 16px | TWK Lausanne 700 | IBM Plex Mono 14px | dark |
| OpenAI Cookbook | OpenAI Sans 16px | 16px/400 | SF Mono 16px | `rgb(0,0,0)` dark |

Citation density, also measured: HF Agents Course unit1 introduction — 18 paragraphs, 0 superscript
references, 0 bracketed references, 2 external links in prose (a star count and an "Update on GitHub"
button). LangChain agents page — 8 paragraphs, 0 citation markers. No measured peer traces claims to
sources in prose, which is why the source chip is the visual signature rather than an added feature.

Deliberate departures from the category baseline: light paper default, serif body prose, colour
reserved for provenance, and a sources rail in the position peers reserve for a table of contents.
