# 004 — Writing the Slice 1 plan, and executing it before it existed

Step: `superpowers:writing-plans`, producing
`docs/superpowers/plans/2026-10-14-slice-1-context-engineering.md`.

## What was planned

Twelve tasks covering the spec's Slice 1 definition: citation guard → source liveness checker →
manifest → condensed notes → keyword retriever → lesson → per-track exercises → runnable project →
gate and routes → README → mastery audit → licensing (gated on an explicit user decision).

## The part worth writing down: the plan was executed before it was committed

A plan that has never run is a guess with formatting. So every code block in the plan was extracted
back out of the markdown and run in a scratch directory (`C:\tmp\tmp.YsmFNCVBYR`, since a Windows
Python resolves `/tmp` differently than Git Bash does — worth knowing before the next run):

```
$ PYTHONPATH=. python -m unittest discover -s tests -t tests      # tools + routes
Ran 29 tests in 0.099s
OK

$ cd modules/context-engineering/project && python -m unittest discover -s . -p "test_*.py"
Ran 7 tests in 0.000s
OK

$ python tools/check_citations.py
0 problem(s)
```

That is the plan's own test code, the plan's own implementation code, the plan's own manifest and the
plan's own lesson opening. Nothing was verified by reading.

## Seven defects the run caught, all of them mine

1. **Unimportable tool names.** The spec spelled them `tools/check-citations.py`. A hyphen cannot
   appear in an imported module name, so `from tools import check_citations` fails. Renamed to
   underscores in the plan, and in `CLAUDE.md`, `DESIGN.md` and the spec.
2. **A YAML bomb.** `title: Context Engineering for AI Agents: Lessons from Building Manus` — an
   unquoted colon-space inside a value. `yaml.safe_load` raised `ScannerError: mapping values are not
   allowed here, line 16, column 45`. Now quoted. This is the same gotcha recorded from the plugin
   marketplace work; it bit again, which means it belongs in the repo's own guidance, not just in
   memory.
3. **The guard rejected the lesson's own header.** `A module in open-agents-course · Tracks: …` is a
   prose paragraph with no citation, so the plan's Task 6 would have failed the build it defines.
   Fixed by making that line a blockquote (`> Module in …`), which the guard classifies as structural.
4. **Frontmatter parsing was too strict.** `parse_note` required the file to start with `---`, but
   `textwrap.dedent` fixtures (and real editor-saved files) start with a newline. Every note fixture
   was being silently skipped — `skip anth-ctx.md: no frontmatter id` six times over. Fixed with
   `.lstrip("\n")`, and the skip line stays, because a retriever that quietly drops a source is worse
   than one that reports it.
5. **Wrong arithmetic in my own test expectations.** `conversation(20)` is 21 messages, so
   `compact(..., keep=4)` summarises 16, not the 19 I wrote; with one pinned error it summarises 15,
   not the 14 I wrote elsewhere. I ran the implementation to find out rather than recomputing on paper
   a second time.
6. **Markdown that broke itself.** A ` ```markdown ` block containing a nested ` ```python ` fence
   closes the outer block early, so the exercise text was going to render as mangled plan prose. The
   inner sample is now indented code.
7. **A claim about counts that was wrong.** Task 2 said "15 tests total across both modules" when the
   two files hold 14. The scratch run settled it: 8 + 6 = 14 there, 29 across the four tool/route test
   files. Numbers in a plan are instructions, not flavour — the implementer trusts them.

A dead `QUOTED` regex and a `__import__("sys")` hack were also removed, and the "replace the
placeholder dates" instruction was replaced with the real dates, since a plan should not contain work
that is only described.

## Source URLs verified this session

The manifest's eight URLs were resolved live, and their titles read from the returned HTML rather
than recalled:

```
200  https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
     :: Effective context engineering for AI agents \ Anthropic
200  https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus
     :: Context Engineering for AI Agents: Lessons from Building Manus
200  https://www.trychroma.com/research/context-rot
     :: Context Rot: How Increasing Input Tokens Impacts LLM Performance | Chroma
200  https://www.langchain.com/blog/context-engineering-for-agents   :: Context Engineering
200  https://www.anthropic.com/news/context-management
     :: Managing context on the Claude Developer Platform | Claude by Anthropic
200  https://adk.dev/context/                                        :: <h1> Agent context
200  https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
200  https://x.com/systemdesignone/status/2109625855462822332
```

Rejected, and recorded as rejected rather than quietly dropped:

```
404  https://cdn.openai.com/a-practical-guide-to-building-agents.pdf
404  https://simonwillison.net/2025/Sep/18/context-engineering/
403  https://openai.com/business-guides-and-resources/a-practical-guide-to-building-agents/
```

The second one is the important mistake: I wrote that URL from memory of an article I am fairly
confident exists. It 404'd. Two of nine guessed URLs were wrong, which is roughly the error rate you
would expect from recall, and it is the reason the spec says a module may cite fewer sources than it
planned but may not cite one it has not read.

## One thing deliberately not pre-written

Task 4 (the condensed notes) ships a template and a procedure but **not** the claims themselves.
Pre-writing eight sources' key claims from memory would violate the repo's central rule to make the
plan look complete. The task requires reading `corpus/raw/<id>` and extracting from it, and says what
to do when a fetch turns out to be a bot-wall: mark `status: failed`, cite nothing.

## Environment facts worth keeping

- `python` is 3.11.0 here; `python3` is 3.14.5. Tests run under `python`.
- `pyyaml` 6.0.3 is installed; **pytest is not**, so the plan uses stdlib `unittest`. The discovery
  command was verified before it went into the document:
  `PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
- Piping a test command through `tail` hides its exit code. That surfaced during this session's check
  and is why the plan's steps show full output instead.

## Lesson

Writing the plan forced the decisions that authoring would otherwise make implicitly — what counts as
a claim, whether a blocked URL is a dead URL, whether a route may point at prose that does not exist.
Seven defects in a document that had not been written as code would have shipped as seven broken
instructions to a future session. The plan is now the only artifact in this repo whose code has been
run before it was committed, which is a standard the rest of the repo should hold itself to.
