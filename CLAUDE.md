# CLAUDE.md

Guidance for agents working in this repository.

## What this repo is

`open-agents-course` — a free, source-cited course on building AI agents. One module pool, three
learner routes (foundations, practitioner, advanced). The differentiator is that every substantive
claim carries a `[source-id]` citation. Treat that as an invariant, not a style preference.

Read `docs/superpowers/specs/2026-10-13-open-agents-course-design.md` before changing structure.

## Design System

Authoritative source: `DESIGN.md` at the repo root.

- **Colour is reserved for provenance.** `#12655F` (light) / `#4FB3A5` (dark) means "this traces to a
  source". Do not use it for brand flourish, CTAs, decoration, or status. Primary buttons are ink
  (`#17140F`), never accent, never gradient.
- **Fonts:** Fraunces (display) · Source Serif 4 (body prose) · Geist (UI labels) · IBM Plex Mono
  (code, data, source chips). Do not substitute Inter, Roboto, Poppins, Montserrat or Space Grotesk.
- **Light paper is the default** (`#F7F4EE`). Dark mode is a supported alternative, not the baseline.
- **Spacing:** 8px base; prose measure capped at 66ch; page max 1120px; radius 4px controls / 6px cards.
- **Motion:** minimal-functional only — a 120ms fade and chip hover. Respect `prefers-reduced-motion`.
- **Lesson layout:** 200px module nav · prose · 240px sources rail. The rail lists sources, not a table
  of contents. That placement is deliberate.
- `design/preview.html` is a self-contained specimen of the system. Use it to check a new component
  against the tokens before building it into the site.

## Checks that must pass

- `tools/check-citations.py` — fails when a lesson claim has no `[source-id]`. Run before every commit
  that touches `modules/`.
- `tools/check-sources.py` — re-resolves every URL in `corpus/manifest.yaml` and flags drift.
- `corpus/raw/` stays gitignored. Third-party full text is never published; `corpus/notes/` carries
  attributed, condensed notes and marked quotations only.

## Journal

`journal/NNN-<slug>.md` — one entry per build step, written before the commit it describes and shipped
in that same commit. Each entry records what was done, why, the exact command, and its real output.
Counts and claims in the journal must come from a command run in that session; no rounded numbers
presented as measurements.
