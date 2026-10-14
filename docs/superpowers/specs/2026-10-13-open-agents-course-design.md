# open-agents-course — design

Date: 2026-10-13
Status: spec v1, under review. Repo created and public.
Owner: arm-pz (Abdul Rahman P Z)

## What this is

A free, openly licensed course on building AI agents, structured as source-cited lessons
with three learner tracks. Two things set it apart from the crowded field of agent courses:
every substantive claim traces to an identified source, and the construction of the course
is itself documented step by step as reusable "build your own" material.

## Audience and tracks

One module pool, three routes through it. Three tracks does not mean three courses — that
would triple maintenance and guarantee drift.

| Track | Who | Depth treatment | Ends when the learner can |
|---|---|---|---|
| Foundations | Non-coders who use AI tools | analogy-first, no build required | explain what an agent is, use one well, describe context and tools |
| Practitioner | Working developers | hands-on code in Python | ship a working agent with tools, memory and retrieval |
| Advanced | Experienced agent builders | no intros, tradeoffs only | defend a production agent architecture with evals and guardrails |

Each module carries a track tag (which of the three routes use it) and a prerequisite list.
Routes are separate ordered files referencing the same modules, so adding a topic means
writing one module and inserting it into the routes.

## Topic pool

Context engineering, agent memory, tool design, planning and reasoning, RAG for agents,
multi-agent orchestration, evaluation, guardrails and human review, skills and progressive
disclosure, MCP, production concerns (cost, latency, observability). Topics are added
incrementally, one full vertical slice at a time.

## Corpus: scope B, four-part structure

Sources: the 10 guides indexed by Neo Kim's thread, plus authoritative primaries —
Anthropic's context-engineering and multi-agent writing, OpenAI Agents SDK docs, the MCP
specification, Google ADK docs, and two or three evaluation papers. Target 25-35 sources.

```
corpus/
  manifest.yaml        id, title, author, url, type, accessed, status, license note
  notes/<id>.md        our own condensed notes: key claims, definitions, marked quotes
  raw/                 full fetched text, gitignored, authoring-only, never published
```

Verbatim copies of third-party articles stay out of the public repo. Attribution alone does
not settle redistribution, and the safer design keeps the question out of scope: `notes/`
is derived study material with clear attribution, `manifest.yaml` is the bibliographic
record that sends readers to originals, and `raw/` keeps authoring grounded offline without
publishing anyone's work. Short quotations appear inside `notes/` marked as quotations.

X/Twitter posts fetch as summaries, not verbatim text. The thread is treated as an index;
each item resolves to its underlying primary artifact, and provenance is recorded in the
manifest.

## Retrieval

Retrieval serves authoring, not (initially) the learner.

**Constraint found during setup:** QMind's `add_source` requires an existing `notebookId`,
exposes no create-notebook operation, and `list_notebooks` returned zero notebooks. So the
notebook cannot be created from here.

Consequence, and it is not a loss:

1. v1 ground truth is `corpus/`. Retrieval runs locally against `notes/` via a committed
   keyword retriever script (`tools/search.py`), so the pipeline works offline, reproducibly,
   with no external dependency.
2. When a notebook exists, `corpus/notes/*.md` gets ingested into it as enrichment.
   Ingestion is recorded in `kb/ingestion.json` (notebook id, source ids, dates).
3. Citations are source ids either way, so nothing written against the local corpus needs
   rewriting when QMind joins.
4. Learner-facing "ask the corpus" is explicitly v2, after the corpus is real and tested.

## Repository layout

```
corpus/manifest.yaml            bibliographic record and fetch status
corpus/notes/<id>.md            our attributed, condensed notes (published)
corpus/raw/                     full fetched text (gitignored, authoring only)
kb/ingestion.json               QMind notebook id + source ids, once enrichment happens
modules/<topic>/                lesson.md, exercises/, project/, gate.md, sources.md
tracks/foundations.yaml         ordered module list, depth notes
tracks/practitioner.yaml
tracks/advanced.yaml
tools/search.py                 local keyword retriever over corpus/notes
tools/check_citations.py        fails when a claim has no [source-id]
tools/check_sources.py          re-resolves every manifest URL, flags drift
journal/NNN-<slug>.md           one per build step
design/preview.html             self-contained specimen of the design system
DESIGN.md                       colour, type, spacing, layout, motion — source of truth for lessons
CLAUDE.md                       agent guidance, including the colour-means-provenance invariant
docs/superpowers/specs/         this design and successors
README.md  LICENSE.md  .gitignore
```

Routes reference modules; they never copy prose. `corpus/raw` is gitignored, so grounding
material stays local by construction rather than by discipline.

## Module contract

`modules/<topic>/` contains:

- `lesson.md` — plain explanation with analogy, worked example, the common mistake for this
  topic, and `[source-id]` citations on substantive claims
- `exercises/` — tasks sized per track
- `project/` — the build a learner does to prove the topic
- `gate.md` — readiness gate: observable criteria for advancing, never time-based
- `sources.md` — links to the originals used, for readers who want the primary material

Every module also exists at three depths via the track tag, not three files: the lesson
states core once, then per-track sections mark what Foundations skips and what Advanced
gets instead.

Gates are audited by `concept-mastery-validator` (installed) to check that each project
actually requires only what earlier modules taught — the gap that silent curriculum drift
creates.

## Build order

**Slice 1 is Context Engineering, end to end:** manifest entry → fetch → normalize → notes →
local retrieval → module authored with citations → all three routes wired → journal entries
documented → committed. Nothing broadens until the slice is genuinely complete, which keeps
"add topics slowly" honest and proves the pipeline before ten half-built modules exist.

## Journal

`journal/NNN-<slug>.md`, one file per build step: what is being done, why, the exact
command, its real output, and the lesson learned. The journal doubles as course content —
it is the "build your own source-grounded curriculum" track — and it is what makes the
community contribution verifiable rather than merely available.

The journal is written before the commit it describes, and shipped in that same commit.
Building without narrating is treated as a defect, not a speedup.

## Skill orchestration

| Skill | Job |
|---|---|
| deep-tutor (curriculum) | stage structure, readiness gates, reorder-by-goal |
| teach (mattpocock) | interactive lesson voice inside modules |
| module-learning-path-generator | career-role framing of the three routes |
| concept-mastery-validator | audit gates against projects for gaps |
| wordsmith | prose, tighten, accuracy pass on each lesson |
| findable | discovery kit: About, topics, README opening, distribution, re-check |
| superpowers:writing-plans | the implementation plan after this spec |
| Presentations / slides / sites | phase 2 delivery surfaces, after content works |

`course-designer` from partme-ai was selected but does not exist in its repo anymore
("No matching skills found"); its role is covered by deep-tutor plus
module-learning-path-generator.

## Quality and error handling

- **Fetch failure:** mark the manifest entry `status: failed`, record the URL, and continue.
  Never substitute model memory silently. A module may cite fewer sources than planned, but
  it may not cite an unread one.
- **Thin evidence:** if retrieval returns weak hits for a topic, the lesson says so in a
  visible "evidence is thin here" note instead of padding with confident prose.
- **Stale citations:** agent documentation churns fast. `tools/check_sources.py` verifies
  every manifest URL still resolves and flags drift; run per module completion and before
  publishing.
- **Uncited claim:** `tools/check_citations.py` fails the run when a lesson makes a
  factual claim with no `[source-id]` and no marker. This is the check that keeps the
  course's one real differentiator intact. Checks are local Python scripts run before each
  commit; GitHub Actions is a later phase, not v1.
- **Before any public release:** `web-preflight` on the hosted site.

## Naming and discovery

Measured, not guessed. Exact-phrase name collisions on GitHub: `agents-from-scratch` is
dominated by 4846-, 2386- and 1045-star repos; `ai-agents-course` has four near-identical
active courses; `build-your-own-ai-agent` collides exactly with a 97-star adjacent project.
`open-agents-course` has zero exact-name owners. "Curriculum" was rejected as reader-facing
vocabulary; "course" is the plainest available noun and keeps the searchable head term.

Slug: **`open-agents-course`**. GitHub About, topics and the README opening carry the task
queries, since a slug cannot win them and a zero-star repo ranks on copy, not on name.
Re-check queries ~3 weeks after publishing; immediate checks prove little because indexing
lags.

## Open items

1. **Custom domain.** `learn.caffeinity.com` is unverified: `caffeinity.com` answers 200,
   `learn.caffeinity.com` returned nothing, and Qoder Sites exposes only
   `<prefix>-<id>.qoder.website`. Needs confirmation that the apex domain is ours and that
   a custom host is supported. Fallback host: a Qoder Site subdomain.
2. **QMind notebook.** Blocked from here; needs a notebook created in the QMind UI if
   enrichment is wanted.
3. **Licenses.** Proposal, not yet applied: CC-BY-4.0 for written content, MIT for code,
   attribution "Abdul Rahman P Z". The corpus rules above depend on this being settled
   before the repo claims an open licence it does not yet carry.

## Resolved during setup

- **Public remote.** Created 2026-10-13: https://github.com/arm-pz/open-agents-course
  (public, `main`, first commit `f03b332`).

## Out of scope for v1

Learner-facing corpus chat, video versions, slide decks, paid tier, hosted LMS, and any
claim about which framework is best. Code examples target Python in v1; .NET is a possible
later variant rather than a parallel track.

## Success criteria

- Slice 1 exists end to end with citations that validate
- All three routes render from one module pool without duplicated prose
- Every build step has a journal entry containing a real command and its real output
- Citation and source checks pass locally before any publish
- Another person can reproduce the pipeline from the repo alone
