# Slice 1 — Context Engineering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first complete vertical slice of `open-agents-course` — one source-cited module on Context Engineering — with the corpus, retrieval and citation checks that make it reproducible.

**Architecture:** A local-first pipeline: a YAML source manifest feeds gitignored raw text, which feeds published condensed notes; a keyword retriever over those notes grounds authoring; the module is written against that grounding and every paragraph is machine-checked for a `[source-id]` citation; three track files route learners through the single module pool.

**Tech Stack:** Python 3.11 (stdlib + PyYAML only), `unittest` (no pytest — see Global Constraints), Markdown, Git/GitHub.

**Spec:** `docs/superpowers/specs/2026-10-13-open-agents-course-design.md`. Design system: `DESIGN.md`. Both travel with this plan; executors read all three.

## Global Constraints

Every task implicitly includes these. Values are copied verbatim from the spec.

- Python is `python` (3.11.0 at `C:\Program Files\Python311\python.exe`). `python3` on this machine is 3.14.5 — do not mix the two in one run.
- Dependencies: standard library plus `pyyaml` 6.0.3 only. **No pytest** (not installed here; stdlib `unittest` keeps the pipeline reproducible for anyone who clones the repo).
- Test command, verified on this machine: `PYTHONPATH=. python -m unittest discover -s tests -t tests -v`. Do not pipe it through `tail` when you need the exit status — the pipe masks the failure code.
- Every substantive claim in a lesson paragraph carries a `[source-id]` citation or an explicit `<!-- no-source: reason -->` exemption marker. A module may cite fewer sources than planned; **it may not cite an unread one.**
- `corpus/raw/` stays gitignored. Third-party full text is never published; `corpus/notes/` carries attributed, condensed notes and marked quotations only.
- Fetch failure handling: mark the manifest entry `status: failed`, record the URL, continue. Never substitute model memory silently.
- Thin evidence handling: if retrieval returns weak hits for a topic, the lesson says so in a visible "evidence is thin here" note instead of padding with confident prose.
- Journal rule: `journal/NNN-<slug>.md` is written **before** the commit it describes and shipped **in that same commit**, containing the exact command and its real output. Building without narrating is a defect.
- No rounded numbers presented as measurements. Counts in journal and lessons come from a command run in that session.
- Tool modules are named with underscores (`tools/check_citations.py`, `tools/check_sources.py`), not the hyphens the spec draft used. Reason: tests import them (`from tools import check_citations`) and a hyphen is not importable. `CLAUDE.md` and the spec are updated to match in Task 1.
- Naming: slug `open-agents-course`, author attribution "Abdul Rahman P Z". Reader-facing vocabulary is "course", never "curriculum".
- Code examples target Python in v1.

## File Structure

Created or modified by this plan:

```
corpus/manifest.yaml                    8 verified sources: id, title, author, url, type, accessed, status, license
corpus/raw/<id>.html|<id>.txt           fetched full text — gitignored, authoring only, never committed
corpus/notes/<id>.md                    published condensed notes with frontmatter + marked quotations
tools/__init__.py                       empty; makes `from tools import …` work under PYTHONPATH=.
tools/check_citations.py                paragraph-level citation guard (exit 1 on any violation)
tools/check_sources.py                  re-resolves every manifest URL; ok | dead | blocked
tools/search.py                         keyword retriever over corpus/notes
tests/test_check_citations.py           citation guard behaviour
tests/test_check_sources.py             URL classifier behaviour (offline)
tests/test_search.py                    retriever ranking and note-loading behaviour
tests/test_tracks.py                    route integrity: refs resolve, prereqs are in-route
modules/context-engineering/lesson.md   the module prose, cited
modules/context-engineering/exercises/foundations.md
modules/context-engineering/exercises/practitioner.md
modules/context-engineering/exercises/advanced.md
modules/context-engineering/project/compact_context.py     runnable learner deliverable
modules/context-engineering/project/test_compact_context.py
modules/context-engineering/project/README.md
modules/context-engineering/gate.md     observable readiness criteria
modules/context-engineering/sources.md  links out to the originals
tracks/foundations.yaml                 route 1
tracks/practitioner.yaml                route 2
tracks/advanced.yaml                    route 3
README.md                               how to run the pipeline from a clone
LICENSE.md                              content + code licensing (gated on user confirmation)
journal/005-*.md … journal/016-*.md     one entry per task
```

Responsibility boundaries worth locking: the three `tools/*.py` scripts each do one check and own no content logic; `tests/` validates tools, never prose; `corpus/notes/` is the only place third-party material is paraphrased for publication; `modules/` never repeats prose that a route file also states (routes reference ids only).

---

### Task 1: Citation guard

Build the invariant checker before any lesson exists, using synthetic fixtures. This is the check that keeps the course's one real differentiator intact, so it is written first and tested first.

**Files:**
- Create: `tools/__init__.py` (empty file)
- Create: `tools/check_citations.py`
- Create: `tests/test_check_citations.py`

**Interfaces:**
- Consumes: nothing (self-contained).
- Produces: `manifest_ids(path) -> set[str]`; `paragraphs(text: str) -> list[str]`; `check_lesson(lesson: pathlib.Path, ids: set[str]) -> list[str]`; `main(argv: list[str]) -> int`. Exit code 1 means at least one violation. Later tasks call `python tools/check_citations.py` from the repo root with no argument.

- [ ] **Step 1: Create the package marker**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && : > tools/__init__.py && ls tools
```

- [ ] **Step 2: Write the failing tests**

Create `tests/test_check_citations.py`:

```python
import pathlib
import tempfile
import textwrap
import unittest

from tools import check_citations as cc

MANIFEST = textwrap.dedent("""
    sources:
      - id: anth-ctx
        title: Effective context engineering for AI agents
        url: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
      - id: man-manus
        title: Context Engineering for AI Agents
        url: https://manus.im/blog/example
    """)


def fixture(lesson_body):
    """Build a temp repo root holding one manifest and one lesson."""
    root = pathlib.Path(tempfile.mkdtemp())
    (root / "corpus").mkdir()
    (root / "corpus" / "manifest.yaml").write_text(MANIFEST, encoding="utf-8")
    lesson = root / "modules" / "context-engineering"
    lesson.mkdir(parents=True)
    (lesson / "lesson.md").write_text(textwrap.dedent(lesson_body), encoding="utf-8")
    return root, lesson / "lesson.md"


class ParagraphsTest(unittest.TestCase):
    def test_splits_on_blank_lines_and_drops_code_fences(self):
        text = "# Title\n\nFirst paragraph.\n\n```\ncode here\n```\n\nSecond paragraph."
        self.assertEqual(cc.paragraphs(text), ["First paragraph.", "Second paragraph."])

    def test_joins_wrapped_lines_of_one_paragraph(self):
        self.assertEqual(cc.paragraphs("Line one\nstill line one\n\nNext"),
                         ["Line one still line one", "Next"])


class CheckLessonTest(unittest.TestCase):
    def test_cited_paragraph_passes(self):
        _, lesson = fixture("""
            A model runs out of retrieval, not intelligence.[anth-ctx]
            """)
        ids = cc.manifest_ids(lesson.parents[2] / "corpus" / "manifest.yaml")
        self.assertEqual(cc.check_lesson(lesson, ids), [])

    def test_uncited_paragraph_fails(self):
        _, lesson = fixture("""
            Everything about this claim is unsourced and that is a defect.
            """)
        ids = cc.manifest_ids(lesson.parents[2] / "corpus" / "manifest.yaml")
        problems = cc.check_lesson(lesson, ids)
        self.assertEqual(len(problems), 1)
        self.assertIn("uncited", problems[0])

    def test_exemption_marker_passes(self):
        _, lesson = fixture("""
            Our own worked example, stated as reasoning. <!-- no-source: our own example -->
            """)
        ids = cc.manifest_ids(lesson.parents[2] / "corpus" / "manifest.yaml")
        self.assertEqual(cc.check_lesson(lesson, ids), [])

    def test_unknown_source_id_fails(self):
        _, lesson = fixture("""
            A claim pointing at a source that does not exist.[nope-not-real]
            """)
        ids = cc.manifest_ids(lesson.parents[2] / "corpus" / "manifest.yaml")
        problems = cc.check_lesson(lesson, ids)
        self.assertIn("unknown source ids", problems[0])

    def test_structural_lines_are_not_claims(self):
        _, lesson = fixture("""
            - Exercise one
            - Exercise two
            """)
        ids = cc.manifest_ids(lesson.parents[2] / "corpus" / "manifest.yaml")
        self.assertEqual(cc.check_lesson(lesson, ids), [])


class MainTest(unittest.TestCase):
    def test_exit_code_one_on_violation_and_zero_on_clean(self):
        root, lesson = fixture("""
            Uncited prose that must be caught by the guard.
            """)
        self.assertEqual(cc.main(["check_citations.py", str(root)]), 1)
        lesson.write_text("Cited prose.[anth-ctx]\n", encoding="utf-8")
        self.assertEqual(cc.main(["check_citations.py", str(root)]), 0)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
Expected: FAIL / ERROR with `ModuleNotFoundError: No module named 'tools.check_citations'` (the package dir exists but the module does not).

- [ ] **Step 4: Write the implementation**

Create `tools/check_citations.py`:

```python
#!/usr/bin/env python
"""Fail when a lesson paragraph carries no citation and no exemption marker.

Rule, deliberately paragraph-level so it is deterministic:
  * a claim-bearing paragraph is any blank-line-separated prose block outside fenced code
  * headings, and blocks whose first line starts with - * | ! or > are structural, not claims
  * a claim-bearing paragraph must contain at least one [source-id], or the marker
    <!-- no-source: reason --> for the module author's own worked examples
"""
import pathlib
import re
import sys

import yaml

EXEMPT = "<!-- no-source:"
CITE = re.compile(r"\[([a-z0-9][a-z0-9-]*)\]")
STRUCTURAL = ("-", "*", "|", "!", ">")


def manifest_ids(path):
    """Every source id declared in corpus/manifest.yaml."""
    data = yaml.safe_load(pathlib.Path(path).read_text(encoding="utf-8"))
    return {s["id"] for s in data["sources"]}


def paragraphs(text):
    """Claim-bearing prose blocks, in order, with code fences and headings removed."""
    out, buf, in_fence = [], [], False
    for line in text.splitlines():
        if line.strip().startswith("```"):
            in_fence = not in_fence
            continue
        if in_fence or line.startswith("#"):
            if buf:
                out.append(" ".join(buf))
                buf = []
            continue
        if line.strip() == "":
            if buf:
                out.append(" ".join(buf))
                buf = []
            continue
        buf.append(line.strip())
    if buf:
        out.append(" ".join(buf))
    return [p for p in out if not p.startswith(STRUCTURAL)]


def check_lesson(lesson, ids):
    """Problem strings for one lesson file; empty list means it passes."""
    lesson = pathlib.Path(lesson)
    problems = []
    for para in paragraphs(lesson.read_text(encoding="utf-8")):
        cited = CITE.findall(para)
        if EXEMPT in para:
            continue
        if not cited:
            problems.append(f"{lesson.name}: uncited paragraph: {para[:70]!r}")
            continue
        unknown = [c for c in cited if c not in ids]
        if unknown:
            problems.append(f"{lesson.name}: unknown source ids {unknown} in: {para[:70]!r}")
    return problems


def main(argv):
    root = pathlib.Path(argv[1]) if len(argv) > 1 else pathlib.Path(".")
    ids = manifest_ids(root / "corpus" / "manifest.yaml")
    problems = []
    for lesson in sorted(root.glob("modules/*/lesson.md")):
        problems.extend(check_lesson(lesson, ids))
    for problem in problems:
        print("FAIL", problem)
    print(f"{len(problems)} problem(s)")
    return 1 if problems else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
Expected: PASS, 8 tests (2 paragraph-splitting, 5 lesson checks, 1 exit code), `OK`.

- [ ] **Step 6: Write the journal entry**

Create `journal/005-citation-guard.md`. It must contain: what the guard enforces and why it is the repo's central invariant, the two commands from Steps 3 and 5, and **their real output pasted verbatim** — including the failing run. Close with the lesson: the guard is paragraph-level because sentence-level detection is guesswork, and a deterministic check that can be explained to a learner beats a clever one that can't.

- [ ] **Step 7: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add tools/__init__.py tools/check_citations.py tests/test_check_citations.py \
        CLAUDE.md docs/superpowers/specs/2026-10-13-open-agents-course-design.md \
        journal/005-citation-guard.md
git commit -m "Add paragraph-level citation guard, tested before the first lesson exists"
git push origin main
```

`CLAUDE.md` and the spec already carry the underscore rename from Global Constraints; commit them with the tool they describe rather than in a separate chore commit.

---

### Task 2: Source liveness checker

Agent documentation churns fast; a dead URL in a source-cited course is a defect. This task builds the checker with offline unit tests only — the live run happens in Task 3 once the manifest exists.

**Files:**
- Create: `tools/check_sources.py`
- Create: `tests/test_check_sources.py`

**Interfaces:**
- Consumes: `corpus/manifest.yaml` entries with keys `id`, `url` (created in Task 3).
- Produces: `classify(status: int) -> str` returning `"ok"`, `"dead"` or `"blocked"`; `load_sources(path) -> list[dict]`; `probe(url: str) -> int`; `report(rows: list[tuple]) -> list[str]`; `main(argv) -> int`. Exit 1 when any source is `dead`; `blocked` never fails the run.

- [ ] **Step 1: Write the failing tests**

Create `tests/test_check_sources.py`:

```python
import pathlib
import tempfile
import textwrap
import unittest

from tools import check_sources as cs

MANIFEST = textwrap.dedent("""
    sources:
      - id: anth-ctx
        url: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
      - id: broken-one
        url: https://example.com/does-not-exist
    """)


class ClassifyTest(unittest.TestCase):
    def test_success_and_redirect_are_ok(self):
        for status in (200, 206, 301, 302, 307, 308):
            self.assertEqual(cs.classify(status), "ok", status)

    def test_not_found_is_dead(self):
        self.assertEqual(cs.classify(404), "dead")
        self.assertEqual(cs.classify(500), "dead")

    def test_forbidden_and_throttle_are_blocked_not_dead(self):
        """A 403 from a bot-protected host says nothing about the page being gone."""
        self.assertEqual(cs.classify(403), "blocked")
        self.assertEqual(cs.classify(429), "blocked")


class LoadSourcesTest(unittest.TestCase):
    def test_reads_id_and_url_pairs_in_manifest_order(self):
        path = pathlib.Path(tempfile.mkstemp(suffix=".yaml")[1])
        path.write_text(MANIFEST, encoding="utf-8")
        rows = cs.load_sources(path)
        self.assertEqual([r["id"] for r in rows], ["anth-ctx", "broken-one"])
        self.assertTrue(rows[0]["url"].startswith("https://"))


class ReportTest(unittest.TestCase):
    def test_flags_dead_and_blocked_and_counts(self):
        lines = cs.report([("anth-ctx", "ok", 200), ("broken-one", "dead", 404),
                           ("gated", "blocked", 403)])
        joined = "\n".join(lines)
        self.assertIn("anth-ctx ok 200", joined)
        self.assertIn("broken-one dead 404", joined)
        self.assertIn("gated blocked 403", joined)
        self.assertEqual(lines[-1], "3 sources: 1 ok, 1 dead, 1 blocked")

    def test_blocked_is_reported_as_inconclusive_not_pass(self):
        lines = cs.report([("gated", "blocked", 403)])
        self.assertIn("check by hand", lines[-2])


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
Expected: ERROR `ImportError: cannot import name 'check_sources' from 'tools'`.

- [ ] **Step 3: Write the implementation**

Create `tools/check_sources.py`:

```python
#!/usr/bin/env python
"""Re-resolve every URL in corpus/manifest.yaml and flag drift.

Run per module completion and before publishing. A blocked response is never
reported as a pass: it is inconclusive and needs a human with a browser.
"""
import pathlib
import sys
import urllib.request

import yaml

UA = "Mozilla/5.0 (compatible; open-agents-course source check)"


def classify(status):
    if 200 <= status < 400:
        return "ok"
    if status in (403, 429, 999):
        return "blocked"
    return "dead"


def load_sources(path):
    data = yaml.safe_load(pathlib.Path(path).read_text(encoding="utf-8"))
    return data["sources"]


def probe(url):
    request = urllib.request.Request(url, headers={"User-Agent": UA})
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            return response.getcode()
    except Exception as error:  # HTTPError carries the code we need
        return getattr(error, "code", 0)


def report(rows):
    counts = {"ok": 0, "dead": 0, "blocked": 0}
    lines = []
    for source_id, verdict, status in rows:
        counts[verdict] += 1
        lines.append(f"{source_id} {verdict} {status}")
        if verdict == "blocked":
            lines.append(f"  inconclusive: {source_id} returned {status}, check by hand")
    lines.append(f"{len(rows)} sources: {counts['ok']} ok, {counts['dead']} dead, "
                 f"{counts['blocked']} blocked")
    return lines


def main(argv):
    root = pathlib.Path(argv[1]) if len(argv) > 1 else pathlib.Path(".")
    sources = load_sources(root / "corpus" / "manifest.yaml")
    rows = []
    for source in sources:
        if source.get("status") == "failed":
            rows.append((source["id"], "dead", 0))
            continue
        status = probe(source["url"])
        rows.append((source["id"], classify(status), status))
    for line in report(rows):
        print(line)
    return 1 if any(verdict == "dead" for _, verdict, _ in rows) else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
Expected: PASS, 14 tests total across both test modules, `OK`.

- [ ] **Step 5: Write the journal entry**

Create `journal/006-source-liveness-checker.md`: the 403-versus-404 reasoning (in Step 1's test docstring), the two commands and their real output, and the note that this checker will be the first thing to break when agent-docs vendors move house — which is the point.

- [ ] **Step 6: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add tools/check_sources.py tests/test_check_sources.py journal/006-source-liveness-checker.md
git commit -m "Add source liveness checker that separates dead from inconclusive"
git push origin main
```

---

### Task 3: The corpus manifest, with URLs verified this session

Eight sources for Context Engineering. Every URL below was resolved live on 2026-10-13 and its `<title>` or `<h1>` read back; two candidates that failed (an old `cdn.openai.com` path and a guessed Simon Willison permalink) are recorded as rejected rather than quietly dropped.

**Files:**
- Create: `corpus/manifest.yaml`
- Create (gitignored): `corpus/raw/anth-ctx.html`, `corpus/raw/man-manus.html`, `corpus/raw/chroma-ctx-rot.html`, `corpus/raw/lc-ctx.html`, `corpus/raw/anth-ctx-mgmt.html`, `corpus/raw/adk-ctx.html`, `corpus/raw/openai-agents-guide.txt`, `corpus/raw/kim-thread.html`
- Modify: `.gitignore` (already lists `corpus/raw/` — verify, do not re-add)

**Interfaces:**
- Consumes: `tools/check_sources.py` (`load_sources`, `probe`, `classify`).
- Produces: `corpus/manifest.yaml` with a top-level `sources:` list whose entries carry `id`, `title`, `author`, `url`, `type`, `accessed`, `status`, `license`. The eight ids are the citation vocabulary for every later task: `anth-ctx`, `man-manus`, `chroma-ctx-rot`, `lc-ctx`, `anth-ctx-mgmt`, `adk-ctx`, `openai-agents-guide`, `kim-thread`.

- [ ] **Step 1: Write the manifest**

Create `corpus/manifest.yaml`:

```yaml
# Bibliographic record for the course corpus. Readers are sent to originals;
# full fetched text stays in corpus/raw/ (gitignored) for authoring only.
# status: fetched | failed — a failed entry is never cited.
corpus: open-agents-course
updated: 2026-10-14
sources:
  - id: anth-ctx
    title: Effective context engineering for AI agents
    author: Anthropic (Claude Code team)
    url: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
    type: primary-essay
    accessed: 2026-10-14
    status: fetched
    license: Anthropic copyright; cited and linked, not redistributed
  - id: man-manus
    title: "Context Engineering for AI Agents: Lessons from Building Manus"
    author: Manus (Yichao "Peak" Ji)
    url: https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus
    type: primary-postmortem
    accessed: 2026-10-14
    status: fetched
    license: author copyright; cited and linked, not redistributed
  - id: chroma-ctx-rot
    title: "Context Rot: How Increasing Input Tokens Impacts LLM Performance"
    author: Chroma (Kelly Hong et al.)
    url: https://www.trychroma.com/research/context-rot
    type: primary-research
    accessed: 2026-10-14
    status: fetched
    license: Chroma publication; findings cited with attribution
  - id: lc-ctx
    title: Context Engineering
    author: LangChain (Harrison Chase)
    url: https://www.langchain.com/blog/context-engineering-for-agents
    type: secondary-overview
    accessed: 2026-10-14
    status: fetched
    license: LangChain copyright; cited and linked, not redistributed
  - id: anth-ctx-mgmt
    title: Managing context on the Claude Developer Platform
    author: Anthropic
    url: https://www.anthropic.com/news/context-management
    type: primary-product-notes
    accessed: 2026-10-14
    status: fetched
    license: Anthropic copyright; cited and linked, not redistributed
  - id: adk-ctx
    title: Agent context
    author: Google Agent Development Kit docs
    url: https://adk.dev/context/
    type: primary-docs
    accessed: 2026-10-14
    status: fetched
    license: Google docs; open licence, link out rather than copy
  - id: openai-agents-guide
    title: A Practical Guide to Building Agents
    author: OpenAI
    url: https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
    type: primary-guide
    accessed: 2026-10-14
    status: fetched
    license: OpenAI publication; cited and linked, not redistributed
  - id: kim-thread
    title: Ten guides for building AI agents (thread)
    author: Neo Kim (@systemdesignone)
    url: https://x.com/systemdesignone/status/2109625855462822332
    type: index
    accessed: 2026-10-14
    status: fetched
    license: post copyright; used as an index, each item resolved to its primary
```

- [ ] **Step 2: Confirm the liveness checker agrees with the hand check**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && python tools/check_sources.py`
Expected: eight lines of `ok 200` (some hosts answer 2xx after redirects), then `8 sources: 8 ok, 0 dead, 0 blocked` and exit 0. `adk-ctx` may show `blocked` — if it does, record the status code, verify the page by hand in a browser, and leave the entry in place; do not delete a source because a bot was refused.

- [ ] **Step 3: Pull the raw text into the gitignored directory**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && mkdir -p corpus/raw
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/anth-ctx.html        https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/man-manus.html       https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/chroma-ctx-rot.html  https://www.trychroma.com/research/context-rot
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/lc-ctx.html          https://www.langchain.com/blog/context-engineering-for-agents
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/anth-ctx-mgmt.html   https://www.anthropic.com/news/context-management
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/adk-ctx.html         https://adk.dev/context/
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/kim-thread.html      https://x.com/systemdesignone/status/2109625855462822332
curl -sSL --max-time 60 -A "Mozilla/5.0" -o corpus/raw/openai-agents-guide.pdf https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
pdftotext corpus/raw/openai-agents-guide.pdf corpus/raw/openai-agents-guide.txt
ls -l corpus/raw
```

Expected: each HTML file is tens of kilobytes or more (a handful of bytes means a bot-wall, not the article), and `openai-agents-guide.txt` exists. `pdftotext` is on PATH on this machine.

- [ ] **Step 4: Prove raw material cannot be published**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && git status --short && git check-ignore -v corpus/raw/anth-ctx.html`
Expected: `git status --short` lists `corpus/manifest.yaml` and nothing under `corpus/raw/`; `git check-ignore -v` prints the `.gitignore` line matching it. If `corpus/raw/` shows up in status, stop and fix `.gitignore` before committing — this is the guard that keeps redistribution out of the public repo by construction.

- [ ] **Step 5: Write the journal entry**

Create `journal/007-manifest-and-fetch.md`. Record the commands and real output from Steps 2-4, the file sizes from `ls -l`, and the two rejected candidate URLs with the response codes that killed them:

```
404  https://cdn.openai.com/a-practical-guide-to-building-agents.pdf        (old path; the guide now sits under /business-guides-and-resources/)
404  https://simonwillison.net/2025/Sep/18/context-engineering/             (permalink guessed from memory; guessing is why it broke)
403  https://openai.com/business-guides-and-resources/a-practical-guide-to-building-agents/  (bot-gated HTML mirror of the PDF)
```

- [ ] **Step 6: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add corpus/manifest.yaml journal/007-manifest-and-fetch.md
git commit -m "Add verified eight-source context engineering corpus manifest"
git push origin main
```

---

### Task 4: Condensed notes — the published half of the corpus

Read what was fetched, then write eight notes. This is where the spec's hard rule bites: a note's key claims must come from the text in `corpus/raw/`, not from memory of the article. The plan deliberately does **not** pre-write these claims; pre-writing them from recall is exactly the failure mode the repo exists to prevent.

**Files:**
- Create: `corpus/notes/anth-ctx.md`, `corpus/notes/man-manus.md`, `corpus/notes/chroma-ctx-rot.md`, `corpus/notes/lc-ctx.md`, `corpus/notes/anth-ctx-mgmt.md`, `corpus/notes/adk-ctx.md`, `corpus/notes/openai-agents-guide.md`, `corpus/notes/kim-thread.md`

**Interfaces:**
- Consumes: `corpus/raw/<id>.html|txt`, `corpus/manifest.yaml`.
- Produces: one note per source whose first block is YAML frontmatter with `id`, `title`, `url`, `accessed`, then headings `## Key claims`, `## Definitions`, `## Quotations`. `tools/search.py` (Task 5) reads `id`/`title`/`url` from that frontmatter and treats the whole body as searchable text.

- [ ] **Step 1: Read each source before writing about it**

For each id, open its raw file and read it end to end. HTML is noisy; navigate by heading rather than scrolling blind:

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
grep -o '<h[12][^>]*>[^<]\{3,90\}' corpus/raw/chroma-ctx-rot.html | sed 's/<[^>]*>//'
```

Run that for each `.html`, then `Read` the raw file (or the `.txt` for the PDF) to get the substance under each heading. Record in the journal the number of headings you enumerated per source — evidence that the document was walked, not skimmed.

- [ ] **Step 2: Write each note in this shape**

Use this template, substituting only values you can point at in the raw text. Frontmatter keys must match exactly — Task 5's loader depends on them:

```markdown
---
id: chroma-ctx-rot
title: "Context Rot: How Increasing Input Tokens Impacts LLM Performance"
url: https://www.trychroma.com/research/context-rot
accessed: 2026-10-14
---

## Key claims

- <claim as we understand it, one sentence, traceable to a section of the source> [chroma-ctx-rot]

## Definitions

- **<term as the source uses it>** — <the source's own sense, not ours>

## Quotations

> "<short verbatim span, under 40 words>" — Chroma, Context Rot

## Where this is thin

<what the source does not establish, so a later lesson cannot over-claim on it>
```

Rules for all eight notes:
- At least 5 `Key claims` bullets for the essays and the research report; at least 3 for the index entry (`kim-thread`), which is provenance for the guide list rather than a source of doctrine.
- Quotations are marked with `>` and attributed in the same block. Nothing longer than 40 words. Full-text redistribution stays out of the repo.
- `Where this is thin` is mandatory. The spec's error handling requires that weak evidence be visible rather than padded; a note that says nothing is wrong is a note nobody will trust later.
- No claim may appear in a note that you cannot locate in the raw file. When the raw file was a bot-wall (Step 3 of Task 3 sizes), set the manifest entry to `status: failed` and write no note — do not reconstruct from memory.

- [ ] **Step 3: Self-check the notes against the sources**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
for f in corpus/notes/*.md; do
  printf '%s ' "$f"; grep -c '^- ' "$f"; done
grep -L '^id: ' corpus/notes/*.md || echo "every note has frontmatter"
```

Expected: a bullet count per note (5+ for the seven substantive sources, 3+ for `kim-thread`), and no file listed by `grep -L`, meaning every note declares its `id`.

- [ ] **Step 4: Write the journal entry**

Create `journal/008-condensed-notes.md`: the heading-enumeration command and its real output, the note count and bullet counts from Step 3, which claims were dropped because they could not be located in the fetched text, and the lesson that fetching and reading are two different steps that only the second one justifies a citation.

- [ ] **Step 5: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add corpus/notes/ journal/008-condensed-notes.md
git commit -m "Add eight attributed condensed notes as the published corpus layer"
git push origin main
```

---

### Task 5: Local keyword retriever over the notes

Retrieval in v1 serves authoring, not the learner, and it must work offline with no external dependency — the reason QMind was demoted to enrichment in the spec (`add_source` requires a `notebookId` and no notebook can be created from here).

**Files:**
- Create: `tools/search.py`
- Create: `tests/test_search.py`

**Interfaces:**
- Consumes: `corpus/notes/*.md` with the Task 4 frontmatter.
- Produces: `parse_note(path: pathlib.Path) -> dict | None` returning `{"id", "title", "url", "body"}`; `index(corpus_dir: pathlib.Path) -> list[dict]`; `search(notes: list[dict], query: str, top: int = 5) -> list[tuple]` returning `(score, id, title, snippet)`; `main(argv: list[str]) -> int`. Later tasks run `python tools/search.py "<terms>"` from the repo root.

- [ ] **Step 1: Write the failing tests**

Create `tests/test_search.py`:

```python
import pathlib
import tempfile
import textwrap
import unittest

from tools import search as st

NOTE_A = textwrap.dedent("""
    ---
    id: anth-ctx
    title: Effective context engineering for AI agents
    url: https://example.com/anth
    accessed: 2026-10-14
    ---

    ## Key claims

    - The context window is a finite resource and attention degrades as it fills.
    - Compaction summarises older turns to keep the loop running.
    """)

NOTE_B = textwrap.dedent("""
    ---
    id: man-manus
    title: Context Engineering for AI Agents
    url: https://example.com/man
    accessed: 2026-10-14
    ---

    ## Key claims

    - The best agents leave the context recoverable through files rather than memory.
    - KV cache hit rate is the metric that actually drives cost in production.
    """)

NOTE_NO_FRONTMATTER = "just prose, no id, no title, nothing to index\n"


def notes_dir():
    root = pathlib.Path(tempfile.mkdtemp())
    root.mkdir(parents=True, exist_ok=True)
    (root / "anth-ctx.md").write_text(NOTE_A, encoding="utf-8")
    (root / "man-manus.md").write_text(NOTE_B, encoding="utf-8")
    (root / "stray.md").write_text(NOTE_NO_FRONTMATTER, encoding="utf-8")
    return root


class ParseNoteTest(unittest.TestCase):
    def test_reads_frontmatter_and_body(self):
        path = notes_dir() / "anth-ctx.md"
        note = st.parse_note(path)
        self.assertEqual(note["id"], "anth-ctx")
        self.assertEqual(note["url"], "https://example.com/anth")
        self.assertIn("finite resource", note["body"])

    def test_returns_none_without_frontmatter_id(self):
        self.assertIsNone(st.parse_note(notes_dir() / "stray.md"))


class SearchTest(unittest.TestCase):
    def setUp(self):
        self.notes = st.index(notes_dir())

    def test_index_skips_unparseable_notes(self):
        self.assertEqual(sorted(n["id"] for n in self.notes), ["anth-ctx", "man-manus"])

    def test_ranks_the_more_relevant_note_first(self):
        results = st.search(self.notes, "context window fills and attention")
        self.assertEqual(results[0][1], "anth-ctx")

    def test_score_counts_term_hits(self):
        results = st.search(self.notes, "context engineering agents")
        for score, source_id, title, snippet in results:
            self.assertGreater(score, 0, source_id)
            self.assertTrue(title)
            self.assertTrue(len(snippet) <= 110)

    def test_query_with_no_hits_returns_empty_not_a_guess(self):
        self.assertEqual(st.search(self.notes, "orthopedic surgery billing"), [])

    def test_empty_query_returns_empty(self):
        self.assertEqual(st.search(self.notes, "   "), [])

    def test_case_and_punctuation_insensitive(self):
        self.assertEqual(st.search(self.notes, "KV CACHE")[0][1], "man-manus")


class MainTest(unittest.TestCase):
    def test_prints_id_score_and_snippet_and_exits_zero(self):
        corpus = notes_dir()
        code = st.main(["search.py", "--corpus", str(corpus), "compaction summarise"])
        self.assertEqual(code, 0)

    def test_zero_hits_exits_one_so_a_missing_topic_is_loud(self):
        corpus = notes_dir()
        code = st.main(["search.py", "--corpus", str(corpus), "kubernetes ingress"])
        self.assertEqual(code, 1)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
Expected: ERROR `ImportError: cannot import name 'search' from 'tools'`.

- [ ] **Step 3: Write the implementation**

Create `tools/search.py`:

```python
#!/usr/bin/env python
"""Keyword retriever over corpus/notes/. No network, no model, no external service.

Deliberately not embeddings: the pipeline has to be reproducible by anyone who
clones the repo, and a keyword index is auditable — you can see why a note scored.
Zero hits exits 1, because "the corpus has nothing on this topic" is information.
"""
import argparse
import pathlib
import re
import sys

import yaml

TOKEN = re.compile(r"[a-z0-9]+")


def parse_note(path):
    """id/title/url/body from one note file, or None if it has no usable frontmatter.

    Leading blank lines are stripped first: a note saved from an editor often starts with
    one, and silently skipping it would drop a whole source out of retrieval.
    """
    text = pathlib.Path(path).read_text(encoding="utf-8").lstrip("\n")
    if not text.startswith("---"):
        return None
    try:
        _, front, body = text.split("---", 2)
    except ValueError:
        return None
    meta = yaml.safe_load(front) or {}
    if not meta.get("id"):
        return None
    return {"id": meta["id"], "title": meta.get("title", meta["id"]),
            "url": meta.get("url", ""), "body": body.strip()}


def index(corpus_dir):
    notes = []
    for path in sorted(pathlib.Path(corpus_dir).glob("*.md")):
        note = parse_note(path)
        if note is None:
            print(f"skip {path.name}: no frontmatter id")
            continue
        note["terms"] = TOKEN.findall(note["title"].lower() + " " + note["body"].lower())
        notes.append(note)
    return notes


def score(query_terms, terms):
    return sum(terms.count(term) for term in query_terms)


def snippet_for(body, query_terms):
    for line in body.splitlines():
        low = line.lower()
        if any(term in low for term in query_terms) and len(line.strip()) > 20:
            clean = line.strip().lstrip("-># ")
            return clean[:110]
    return body.strip().replace("\n", " ")[:110]


def search(notes, query, top=5):
    terms = TOKEN.findall(query.lower())
    if not terms:
        return []
    hits = []
    for note in notes:
        value = score(terms, note["terms"])
        if value > 0:
            hits.append((value, note["id"], note["title"], snippet_for(note["body"], terms)))
    hits.sort(key=lambda row: (-row[0], row[1]))
    return hits[:top]


def main(argv):
    parser = argparse.ArgumentParser(description="Keyword search over corpus/notes")
    parser.add_argument("--corpus", default="corpus/notes")
    parser.add_argument("--top", type=int, default=5)
    parser.add_argument("query", nargs="+")
    args = parser.parse_args(argv[1:])
    notes = index(pathlib.Path(args.corpus))
    hits = search(notes, " ".join(args.query), args.top)
    if not hits:
        print(f"no hits in {len(notes)} notes for {args.query!r} — evidence is thin here")
        return 1
    for value, source_id, title, snip in hits:
        print(f"{value:>3}  {source_id:<20} {title}")
        print(f"     {snip}")
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && PYTHONPATH=. python -m unittest discover -s tests -t tests -v`
Expected: PASS, all three test modules, `OK`. If `test_ranks_the_more_relevant_note_first` fails, check that `index()` lowercases both title and body before tokenising — do not weaken the test.

- [ ] **Step 5: Run it against the real corpus**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
python tools/search.py "context window finite resource"
python tools/search.py "compaction summarise"
python tools/search.py "cache cost"
```
Expected: hits led by the source that actually argues each point. Paste the real output in the journal; if a query returns nothing when it should, the note for that source is too thin and Task 4 needs a fix before authoring starts.

- [ ] **Step 6: Write the journal entry**

Create `journal/009-local-retriever.md`: the failing and passing test runs, the three live queries with real output, and the reason this is not a vector index (reproducible, auditable, offline, and QMind cannot create the notebook from here).

- [ ] **Step 7: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add tools/search.py tests/test_search.py journal/009-local-retriever.md
git commit -m "Add offline keyword retriever over the published corpus notes"
git push origin main
```

---

### Task 6: The lesson

One module, `context-engineering`, written once and read by all three tracks. This is the largest single authoring task in the slice; it is prose, so its "test" is the citation guard plus a human read with `wordsmith`.

**Files:**
- Create: `modules/context-engineering/lesson.md`

**Interfaces:**
- Consumes: `tools/search.py` for grounding, the eight ids from `corpus/manifest.yaml`, `DESIGN.md` for the chip vocabulary.
- Produces: `modules/context-engineering/lesson.md` whose headings the other tasks reference: `## What it is`, `## Why it is the bottleneck`, `## Worked example`, `## The common mistake`, `## Per-track notes`, `## Exercises`, `## Gate`. Track ids `foundations`, `practitioner`, `advanced` appear verbatim under `## Per-track notes`.

- [ ] **Step 1: Assemble the evidence**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
python tools/search.py "context window" > /tmp/hits-ctx.txt
python tools/search.py "compaction" >> /tmp/hits-ctx.txt
python tools/search.py "cache cost" >> /tmp/hits-ctx.txt
python tools/search.py "tool result offload files" >> /tmp/hits-ctx.txt
cat /tmp/hits-ctx.txt
```

Read every hit's note before writing. If a planned section has no supporting note, write the visible "evidence is thin here" line the spec requires instead of confident prose.

- [ ] **Step 2: Write the lesson with this claim map**

Every claim-bearing paragraph gets exactly one row here. Write the paragraph, then append the ids. The opening three sections are given in full as the voice to match; the rest is given as the claim map the author must fill from the notes, section by section.

Write these verbatim, adjusting only wording you can defend:

```markdown
# Context Engineering

> Module in **open-agents-course** · Tracks: foundations, practitioner, advanced · Prereqs: none

## What it is

An agent is a model that acts, then reads what happened, then acts again. Every step of
that loop adds text to the window the model gets to look at: your instructions, the tool
definitions, the tool results, the previous turns. Context engineering is the practice of
deciding what is in that window at each step, and what is deliberately left out.[anth-ctx]

The phrase was popularised as a successor to "prompt engineering" once prompts stopped
being one block of text you wrote once and became a state you curate continuously on every
turn of a loop.[lc-ctx]
```

Then write these sections against this map, one row = one paragraph, with the ids in the right-hand column cited in it:

| Section | Claim to establish | Ids to cite |
|---|---|---|
| Why it is the bottleneck | Attention is a finite resource that degrades as the window fills; performance drops before the token limit is reached, so "it still fits" is not the test | `chroma-ctx-rot`, `anth-ctx` |
| Why it is the bottleneck | Cost and latency track the input you resend each turn, and a cache miss on that prefix is the difference between an agent that is affordable and one that is not | `man-manus` |
| Why it is the bottleneck | The provider's own framing: context is a resource to be managed like memory, budgeted per step rather than filled to capacity | `anth-ctx-mgmt`, `adk-ctx` |
| Worked example | Three strategies in one loop: write out to a file, select back in, and compact when the budget is hit — with our own annotated trace, exempted as our example | `man-manus`, `anth-ctx` + `<!-- no-source: our own worked example -->` on the annotated-trace paragraph |
| Worked example | Compaction is the strategy that keeps the most recent turns verbatim and replaces older ones with a summary, which is why it loses the detail your failed attempts live in | `anth-ctx`, `anth-ctx-mgmt` |
| The common mistake | Learners compact by summarising the middle of the transcript and keeping everything else, which deletes precisely the evidence of what the agent already tried and failed at | `chroma-ctx-rot`, `man-manus` |
| The common mistake | Second mistake: treating the whole window as one pile instead of partitioning instruction, tool definitions and results, so the fixed prefix changes and the cache is thrown away each turn | `man-manus` |
| Per-track notes | Foundations skips the worked example's code and keeps the analogy; practitioner implements `compact()` in the project; advanced reads the compaction API and argues where it is the wrong tool | `anth-ctx-mgmt`, `openai-agents-guide` |
| Exercises | Points at `exercises/`, sized per track; no new doctrine stated here | `<!-- no-source: pointer to our own exercises -->` |
| Gate | Points at `gate.md` and states that advancing means doing, not finishing reading | `<!-- no-source: pointer to our own gate -->` |

Structural requirements the guard depends on:
- the metadata line under the title is a blockquote (`> Module in …`), which the guard treats as
  structural; as plain prose it would be read as an uncited claim and fail the build
- fenced code blocks are exempt automatically, so use one in `Worked example` for the annotated trace
- paragraphs that are our own reasoning take the `<!-- no-source: reason -->` marker; there are at most four in this lesson, and each names what it is our own
- no `[1]`-style numeric footnotes anywhere — ids are the only citation form
- the module's prose lives here once; routes reference this file and never restate it

- [ ] **Step 3: Prove it passes the guard**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && python tools/check_citations.py`
Expected: `0 problem(s)` and exit 0. Then prove the guard is real by deleting one `[chroma-ctx-rot]` citation, re-running, and seeing exactly one `uncited paragraph` line plus `1 problem(s)`; restore the citation and re-run to clean. Paste both runs into the journal — a passing check that was never shown to fail is unverified.

- [ ] **Step 4: Prose pass**

Invoke the `wordsmith` skill on `modules/context-engineering/lesson.md` for the tighten-and-accuracy pass. Do not add facts in this pass, and do not shorten a claim so far that the citation no longer supports it.

- [ ] **Step 5: Write the journal entry**

Create `journal/010-context-engineering-lesson.md`: the search output that grounded each section, the two citation-guard runs (fail and pass, verbatim), the word count before and after the prose pass, and the lesson — writing from notes you read is slower than writing from memory and is the entire reason the course is citable.

- [ ] **Step 6: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add modules/context-engineering/lesson.md journal/010-context-engineering-lesson.md
git commit -m "Add the Context Engineering module, every claim traced to the corpus"
git push origin main
```

---

### Task 7: Per-track exercises

**Files:**
- Create: `modules/context-engineering/exercises/foundations.md`
- Create: `modules/context-engineering/exercises/practitioner.md`
- Create: `modules/context-engineering/exercises/advanced.md`

**Interfaces:**
- Consumes: `## Per-track notes` in `modules/context-engineering/lesson.md` (same three track names).
- Produces: three files, each with `# Exercise <N> — <title>` headings and a `## What "done" looks like` section. Task 9's `gate.md` quotes the completion criteria from these files verbatim, so the criteria must be observable statements here.

- [ ] **Step 1: Write `exercises/foundations.md`** — no code, analogies only. Three exercises:

```markdown
# Exercise 1 — Find the window

Your assistant has been chatting with you for two hours and suddenly answers as though it
has forgotten something you both agreed on in the first half of the conversation.

1. In three sentences to a colleague, explain what probably happened without using the
   words "tokens" or "context window".
2. Name one thing you could do right now, in the app, that would fix it — and why it works.

## What "done" looks like

The explanation says the assistant only gets to look at a limited amount of recent
conversation, and that the agreement fell outside what it was shown again. No jargon.
The fix is starting a fresh conversation with the agreement restated, or asking for it
to be repeated — something a non-coder could actually do.

# Exercise 2 — Sort the pile

Given the list below, mark each item "must be in front of the model every step" or
"can live outside and be fetched when needed", and give one reason per line:
system instructions, tool descriptions, last turn's file contents, a 40-page PDF the
agent read three steps ago, the user's current message, old tool results.

## What "done" looks like

Six classifications with reasons. The current message and system instructions are in-front;
the PDF and old tool results are fetchable. Two answers marked "depends" are acceptable if
the condition is named.

# Exercise 3 — One paragraph to someone else

Explain, in one paragraph someone could repeat at dinner, why an agent that has read a lot
can still do worse than an agent that has read the right amount.

## What "done" looks like

A paragraph that survives being repeated. If it needs a diagram to make sense, it is not done.
```

- [ ] **Step 2: Write `exercises/practitioner.md`** — hands-on Python, sized to be finished in one sitting:

```markdown
# Exercise 1 — Measure before you manage

Take any conversation transcript you have (your own agent logs, or a JSON list you write)
and print the character length of each message plus the running total.

    def running_total(messages):
        total = 0
        for index, message in enumerate(messages):
            total += len(message["content"])
            print(index, len(message["content"]), total)
        return total

Report the point where the total crosses 20,000 characters. That is roughly where the
compaction decision in this module starts to matter.

## What "done" looks like

The function runs, the crossing index is reported, and the learner can say what was in the
messages up to that point.

# Exercise 2 — Keep the failures

Write `keep_error_results(messages) -> list` that returns every message whose `is_error`
key is true, plus the first message. Then answer in two sentences: why is a failed tool
result worth more window space than a successful one from ten turns ago?

## What "done" looks like

A one-line function that passes three hand-made cases, and an answer that says failed
attempts are the only evidence in the transcript that a particular route was already tried.

# Exercise 3 — Two strategies, one loop

Implement write-out and select-back-in: dump the last five tool results to a file, remove
them from `messages`, and add a single message saying where they went. Re-read the file
when a later step asks for it.

## What "done" looks like

The transcript is smaller, the agent still completes the task, and the learner can show the
message that says where the data went.
```

- [ ] **Step 3: Write `exercises/advanced.md`** — tradeoffs only, no introduction repeated:

```markdown
# Exercise 1 — Defend a budget

You are told the per-step context budget is 30,000 tokens for a support agent with 11 tool
definitions averaging 400 tokens each. Decide what stays, what is retrieved, what is
summarised, and what is dropped. Write the decision as a table with a rationale column.

## What "done" looks like

Arithmetic that adds up, a stated reason for every line, and one explicit tradeoff the
designer is accepting rather than hiding.

# Exercise 2 — Argue against compaction

Write 200 words on when automatic compaction is the wrong answer, in a system where the
agent's earlier tool results are load-bearing for correctness. Cite the compaction behaviour
described in the lesson, then say what you would do instead.

## What "done" looks like

A position a reviewer could disagree with. "It depends" without a recommendation is not a
position.

# Exercise 3 — Measure the loss

Design an eval that would tell you whether compaction made an agent worse. Name the task,
the metric, the control condition, and the failure mode you expect to see first.

## What "done" looks like

Something another engineer could implement from the description alone.
```

- [ ] **Step 4: Confirm the guard still passes**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course && python tools/check_citations.py`
Expected: `0 problem(s)` — the guard only scans `modules/*/lesson.md`, so exercise prose is not citation-bound, but the lesson must still be clean.

- [ ] **Step 5: Write the journal entry**

Create `journal/011-per-track-exercises.md`: the three files and their exercise counts, the guard output, and the design note that Foundations tasks are answerable without a terminal, because a track that quietly requires code excludes exactly the learners it names.

- [ ] **Step 6: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add modules/context-engineering/exercises/ journal/011-per-track-exercises.md
git commit -m "Add three per-track exercise sets sized to what each track can do"
git push origin main
```

---

### Task 8: The project — a runnable deliverable, not a demo

The project is what proves the topic. It must run with no API key and no network, because a learner who cannot run it learned nothing observable.

**Files:**
- Create: `modules/context-engineering/project/compact_context.py`
- Create: `modules/context-engineering/project/test_compact_context.py`
- Create: `modules/context-engineering/project/README.md`

**Interfaces:**
- Consumes: nothing outside this directory (stdlib only).
- Produces: `count_tokens(messages: list[dict]) -> int`; `compact(messages: list[dict], threshold: int, summariser) -> list[dict]` where `summariser(block: list[dict]) -> dict` is injected by the caller. `gate.md` (Task 9) requires `python -m unittest discover -s . -p "test_*.py"` to pass from inside `project/`.

- [ ] **Step 1: Write the failing tests**

Create `modules/context-engineering/project/test_compact_context.py`:

```python
import unittest

from compact_context import compact, count_tokens


def fake_summariser(block):
    return {"role": "system", "content": f"summary of {len(block)} messages"}


def conversation(n=10, error_at=None):
    messages = [{"role": "system", "content": "You are a support agent."}]
    for i in range(n):
        message = {"role": "tool" if i % 2 else "assistant", "content": f"turn {i} " + "x" * 200}
        if error_at is not None and i == error_at:
            message["is_error"] = True
        messages.append(message)
    return messages


class CountTokensTest(unittest.TestCase):
    def test_estimates_a_quarter_of_characters(self):
        self.assertEqual(count_tokens([{"content": "a" * 400}]), 100)

    def test_empty_conversation_is_zero_not_negative(self):
        self.assertEqual(count_tokens([]), 0)


class CompactTest(unittest.TestCase):
    def test_no_compaction_below_threshold(self):
        messages = conversation(4)
        self.assertEqual(compact(messages, 10_000, fake_summariser), messages)

    def test_keeps_the_first_message_and_a_summary_and_the_tail(self):
        messages = conversation(20)
        result = compact(messages, 1_000, fake_summariser)
        self.assertEqual(result[0], messages[0])
        self.assertEqual(result[1]["content"], "summary of 16 messages")
        self.assertEqual(result[-1], messages[-1])

    def test_failed_tool_results_survive_compaction(self):
        """The lesson's common mistake: summarising away the evidence of what was tried."""
        messages = conversation(20, error_at=5)
        result = compact(messages, 1_000, fake_summariser)
        pinned = [m for m in result if m.get("is_error")]
        self.assertEqual(len(pinned), 1)
        self.assertEqual(pinned[0], messages[6])

    def test_summary_reports_only_the_messages_it_ate(self):
        messages = conversation(20, error_at=2)
        result = compact(messages, 1_000, fake_summariser)
        # 21 messages total: 1 head + 4 tail + 1 pinned error = 6 kept, 15 summarised
        self.assertEqual(result[1]["content"], "summary of 15 messages")

    def test_tail_size_is_a_parameter_with_a_sane_default(self):
        messages = conversation(30)
        short = compact(messages, 100, fake_summariser, keep=2)
        self.assertEqual(short[-2:], messages[-2:])


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run them to verify they fail**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course/modules/context-engineering/project && python -m unittest -v test_compact_context 2>&1`
Expected: ERROR `ModuleNotFoundError: No module named 'compact_context'`.

- [ ] **Step 3: Write the implementation**

Create `modules/context-engineering/project/compact_context.py`:

```python
"""A compaction routine that keeps what the learner would otherwise throw away.

This is the module's common mistake made unfalsifiable: a naive compact() summarises
the middle of the transcript, and the middle is where the failed attempts live. Pinned
messages (the system prompt and any tool result that errored) are never summarised.
Runs with no network, no API key, and no dependencies outside the standard library.
"""


def count_tokens(messages):
    """Rough token count: four characters per token. Not exact, and that is fine —
    the point is a budget you can compare against, not a tokenizer to trust."""
    return sum(len(message.get("content", "")) for message in messages) // 4


def is_pinned(message):
    return message.get("role") == "system" or message.get("is_error") is True


def compact(messages, threshold, summariser, keep=4):
    """Return a transcript under budget: pinned messages, a summary of the rest,
    and the last `keep` messages verbatim.

    `summariser` takes the dropped block and returns one message dict. It is injected
    so this file never calls a model and never needs a key.
    """
    if count_tokens(messages) < threshold:
        return list(messages)

    head = messages[:1]
    tail = messages[-keep:] if keep else []
    pinned = [m for m in messages if is_pinned(m) and m not in head and m not in tail]
    middle = [m for m in messages if m not in head and m not in tail and m not in pinned]
    return head + [summariser(middle)] + pinned + tail
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course/modules/context-engineering/project && python -m unittest discover -s . -p "test_*.py" -v`
Expected: PASS, 7 tests, `OK`. If `test_summary_reports_only_the_messages_it_ate` fails, the bug is in the plan's arithmetic, not the implementation — recount the 20-message fixture (1 head, 4 tail, 1 pinned, 14 middle) and correct whichever side is wrong, then say in the journal which one moved.

- [ ] **Step 5: Write the project README**

Create `modules/context-engineering/project/README.md`:

```markdown
# Project — a compaction step that survives its own common mistake

Build the step an agent takes when the window is full: keep what matters, summarise the
rest, and do not summarise away the evidence of what already failed.

## Run it

    cd modules/context-engineering/project
    python -m unittest discover -s . -p "test_*.py" -v

No API key, no network, no dependencies beyond Python's standard library.

## What you are building

`compact(messages, threshold, summariser, keep=4)` takes a transcript as a list of message
dicts and returns a shorter one:

1. the first message stays — it is the system prompt
2. one summary message replaces the middle
3. any message with `is_error: True` stays verbatim, because a failed attempt is the only
   record in the transcript that this route was already tried
4. the last `keep` messages stay verbatim, because the agent is mid-action in them

The `summariser` is passed in. You never call a model here; you pass a function that
pretends to, so the behaviour is testable and free.

## Order of work

1. Read `test_compact_context.py` and say out loud what each test demands.
2. Get `count_tokens` right first; every threshold assertion depends on it.
3. Write `compact` so all seven tests pass. Do not add a ninth test until the eighth is green.
4. Now break it on purpose: delete the `is_error` pinning and read which test fails. That
   failure is the lesson's common mistake, demonstrated rather than asserted.
5. Extend it: pin messages by recency instead of error, and record what you lose.

## What to hand in

The passing run, plus three sentences on what step 4 taught you.
```

- [ ] **Step 6: Write the journal entry**

Create `journal/012-compaction-project.md`: the failing run, the passing run, which side moved in Step 4's arithmetic note, and the reason the summariser is injected (testable, free, and no credential required to finish a module).

- [ ] **Step 7: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add modules/context-engineering/project/ journal/012-compaction-project.md
git commit -m "Add runnable compaction project with the common mistake pinned by tests"
git push origin main
```

---

### Task 9: Gate and sources — then prove the routes are honest

**Files:**
- Create: `modules/context-engineering/gate.md`
- Create: `modules/context-engineering/sources.md`
- Create: `tracks/foundations.yaml`, `tracks/practitioner.yaml`, `tracks/advanced.yaml`
- Create: `tests/test_tracks.py`

**Interfaces:**
- Consumes: module directory `context-engineering`; the three track names from `lesson.md`; exercise completion criteria from `exercises/*.md`.
- Produces: route files with keys `track`, `depth`, `ends_when`, `modules` (list of `{id, prereqs}`), `planned`. `tests/test_tracks.py` fails CI-equivalently when a route references a module that does not exist or a prereq that is not itself in the route.

- [ ] **Step 1: Write `gate.md`**

```markdown
# Readiness gate — Context Engineering

Gates are observable, never time-based. You pass by doing the thing, not by finishing the
reading. An auditor (`concept-mastery-validator`) checks that every item here is reachable
from this module and the ones before it.

## Foundations

- Explain to a non-coder, without the words "token" or "context window", why an assistant
  can forget an agreement made two hours earlier. (Exercise 1)
- Classify six transcript items as in-front or fetchable, with a reason each. (Exercise 2)
- Say what a failed tool result is worth, in one sentence, and why it is worth more than a
  successful one from ten turns ago.

## Practitioner

- Everything in Foundations.
- `compact()` passes all seven tests in `project/`, unchanged. (Project, step 4)
- Show, by breaking it deliberately, which test catches the missing `is_error` pinning.
- Write out and select back in over a real transcript: smaller context, same task finished.

## Advanced

- Everything in Practitioner.
- A budget decision table for a 30,000-token per-step agent with 11 tools, arithmetic
  included, with one tradeoff stated as accepted rather than hidden. (Exercise 1)
- A 200-word position against automatic compaction in a system where early tool results are
  load-bearing. (Exercise 2)
- An eval design for measuring compaction loss that another engineer could implement from
  the description alone. (Exercise 3)

## Not passing

- Having read the lesson and not run the project.
- Green tests with no explanation of why the failed results are pinned.
- Any answer that says "it depends" without naming what it depends on and what you choose.
```

- [ ] **Step 2: Write `sources.md`**

Every id used by this module, with the real URL from the manifest, the access date, and what the source is used for:

```markdown
# Sources for this module

The course's rule: a claim is only as good as the source you can click. Short quotations
appear in the corpus notes with attribution; full text stays out of this repo, so these
links are the record.

| id | source | used for |
|---|---|---|
| anth-ctx | [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic, accessed 2026-10-14 | the resource framing, compaction definition |
| man-manus | [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) — Manus, accessed 2026-10-14 | leave-then-retrieve, cache hit rate as the cost metric |
| chroma-ctx-rot | [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://www.trychroma.com/research/context-rot) — Chroma, accessed 2026-10-14 | degradation before the limit is reached |
| lc-ctx | [Context Engineering](https://www.langchain.com/blog/context-engineering-for-agents) — LangChain, accessed 2026-10-14 | the term's framing as successor to prompt engineering |
| anth-ctx-mgmt | [Managing context on the Claude Developer Platform](https://www.anthropic.com/news/context-management) — Anthropic, accessed 2026-10-14 | compaction as a product behaviour, not only a technique |
| adk-ctx | [Agent context](https://adk.dev/context/) — Google ADK docs, accessed 2026-10-14 | provider-neutral partitioning of instruction, state and results |
| openai-agents-guide | [A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) — OpenAI, accessed 2026-10-14 | where context sits inside an overall agent design |
| kim-thread | [Ten guides for building AI agents](https://x.com/systemdesignone/status/2109625855462822332) — Neo Kim, accessed 2026-10-14 | the index this corpus was built from |

Check these links still resolve: `python tools/check_sources.py`
```

The `accessed` values above are copied from `corpus/manifest.yaml` (Task 3 sets them to `2026-10-14`). If the table and the manifest ever disagree, the manifest wins and this table is corrected in the same commit.

- [ ] **Step 3: Write the three route files**

`tracks/foundations.yaml`:

```yaml
# Route: non-coders who use AI tools. Analogy-first, no build required.
track: foundations
depth: analogy-first, no build required
ends_when: >
  explain what an agent is, use one well, describe context and tools
modules:
  - id: context-engineering
    prereqs: []
planned:
  - what-an-agent-is
  - using-an-agent-well
  - tool-design
```

`tracks/practitioner.yaml`:

```yaml
# Route: working developers. Hands-on code in Python.
track: practitioner
depth: hands-on code in Python
ends_when: >
  ship a working agent with tools, memory and retrieval
modules:
  - id: context-engineering
    prereqs: []
planned:
  - tool-design
  - agent-memory
  - rag-for-agents
```

`tracks/advanced.yaml`:

```yaml
# Route: experienced agent builders. No intros, tradeoffs only.
track: advanced
depth: no intros, tradeoffs only
ends_when: >
  defend a production agent architecture with evals and guardrails
modules:
  - id: context-engineering
    prereqs: []
planned:
  - multi-agent-orchestration
  - evaluation
  - guardrails-and-human-review
```

- [ ] **Step 4: Write the failing route-integrity tests**

Create `tests/test_tracks.py`:

```python
import pathlib
import unittest

import yaml

ROOT = pathlib.Path(__file__).resolve().parents[1]
TRACKS = ("foundations", "practitioner", "advanced")


def load(track):
    return yaml.safe_load((ROOT / "tracks" / f"{track}.yaml").read_text(encoding="utf-8"))


class RouteIntegrityTest(unittest.TestCase):
    def test_all_three_routes_exist_and_name_themselves(self):
        for track in TRACKS:
            self.assertEqual(load(track)["track"], track)

    def test_every_referenced_module_has_a_lesson(self):
        """A route must never point at prose that does not exist — that is how silent
        curriculum drift begins."""
        for track in TRACKS:
            for entry in load(track)["modules"]:
                lesson = ROOT / "modules" / entry["id"] / "lesson.md"
                self.assertTrue(lesson.exists(), f"{track} references missing {lesson}")

    def test_every_prereq_is_itself_in_the_route_and_earlier(self):
        for track in TRACKS:
            data = load(track)
            order = [entry["id"] for entry in data["modules"]]
            for entry in data["modules"]:
                for prereq in entry["prereqs"]:
                    self.assertIn(prereq, order, f"{track}: {entry['id']} needs {prereq}")
                    self.assertLess(order.index(prereq), order.index(entry["id"]),
                                    f"{track}: {prereq} must come before {entry['id']}")

    def test_planned_modules_are_not_half_wired(self):
        """A planned id must not already have a lesson on disk; moving an id out of
        `planned` and into `modules` is how a topic officially ships."""
        for track in TRACKS:
            for module_id in load(track).get("planned", []):
                self.assertFalse((ROOT / "modules" / module_id / "lesson.md").exists(),
                                 f"{track}: {module_id} is built but still listed as planned")

    def test_routes_reference_the_same_module_pool_without_copying_prose(self):
        for track in TRACKS:
            text = (ROOT / "tracks" / f"{track}.yaml").read_text(encoding="utf-8")
            self.assertLess(len(text.splitlines()), 40, f"{track} is restating lesson prose")


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 5: Run everything**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
PYTHONPATH=. python -m unittest discover -s tests -t tests -v
python tools/check_citations.py
```
Expected: all tests PASS including the five new route tests, then `0 problem(s)`.

- [ ] **Step 6: Write the journal entry**

Create `journal/013-gates-and-routes.md`: the real output of Step 5, the reasoning that routes are lists of ids rather than documents, so adding a topic means one module plus three line-inserts.

- [ ] **Step 7: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add modules/context-engineering/gate.md modules/context-engineering/sources.md tracks/ tests/test_tracks.py journal/013-gates-and-routes.md
git commit -m "Add readiness gate, source table and three routes guarded by tests"
git push origin main
```

---

### Task 10: README — the repo must reproduce from a clone

The spec's last success criterion is that another person can reproduce the pipeline from the repo alone. That is a README, not a hope.

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: every command from Tasks 1-9.
- Produces: the public entry point. Its opening sentence names the task and the audience (per the `findable` audit), and it carries the install/quickstart line near the top.

- [ ] **Step 1: Write it**

```markdown
# open-agents-course

A free, openly licensed course on **building AI agents**, where every substantive claim
carries a source you can click. Three routes through one pool of modules: for people who
use AI tools, for developers who want to ship, and for people already building agents in
production.

Start here if you want to understand what an agent actually does: `tracks/foundations.yaml`
→ `modules/context-engineering/lesson.md`.

## What is different

1. **Citations are enforced, not encouraged.** `tools/check_citations.py` fails the build
   when a lesson paragraph has no `[source-id]` and no exemption marker. That check, not a
   style preference, is why the course can be read as evidence.
2. **The construction of the course is course material.** `journal/` is one entry per build
   step with the real command and its real output — a "build your own source-grounded
   curriculum" track you can copy.

## Run the checks

    python tools/search.py "context window"     # keyword search over the corpus notes
    python tools/check_citations.py             # exit 1 on any uncited claim
    python tools/check_sources.py               # re-resolve every manifest URL

Requirements: Python 3.11 and PyYAML (`python -m pip install pyyaml`). Nothing else — no
model API key, no network for the tests, no framework.

## Layout

    corpus/manifest.yaml   the bibliographic record; readers are sent to originals
    corpus/notes/          attributed, condensed notes — the published corpus layer
    modules/<topic>/       lesson.md, exercises/, project/, gate.md, sources.md
    tracks/                three ordered routes over the same module pool
    tools/                 citation guard, source liveness, keyword retriever
    journal/               how each step was built, commands and output included
    DESIGN.md              the visual system: colour is reserved for provenance

`corpus/raw/` is gitignored by design. Third-party full text is never redistributed here;
attribution does not settle redistribution, so the repo carries notes and links.

## Status

Slice 1 (Context Engineering) is complete end to end: corpus → notes → retrieval →
cited module → exercises → project → gate → three routes. Topics are added one full
vertical slice at a time, in the order `tracks/*.yaml` lists under `planned`.

## Contribute

Read `docs/superpowers/specs/2026-10-13-open-agents-course-design.md` first. A new topic
means one module written against a source you actually fetched, not a summary of what the
model already believes.

## License

See `LICENSE.md`: CC BY 4.0 for the writing, MIT for the code. Attribution: Abdul Rahman P Z.
```

Note the last block: it describes licensing as settled. It is not yet — Task 12 writes the file, and if the user declines that licensing, change this block in the same commit as Task 12 rather than leaving the README to claim something the repo does not grant.

- [ ] **Step 2: Verify every command in it runs**

Run each of the four fenced command lines exactly as written, from the repo root, and check the outputs match what the README claims. Expected: three checks pass; `python -m pip install pyyaml` should report already satisfied on this machine.

- [ ] **Step 3: Write the journal entry**

Create `journal/014-readme-and-reproducibility.md`: the four commands and their real output, and the test that proves the README is not fiction — a fresh clone would have to work, so state explicitly which of these steps you re-ran rather than assumed.

- [ ] **Step 4: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add README.md journal/014-readme-and-reproducibility.md
git commit -m "Add README so the pipeline can be reproduced from a clone alone"
git push origin main
```

---

### Task 11: Slice audit — does the gate require only what came before it?

The spec assigns this to `concept-mastery-validator`. It is the check that catches the drift a curriculum quietly develops.

**Files:**
- Modify: whatever the audit produces — findings go into `journal/`, and any required lesson change lands in `modules/context-engineering/lesson.md`

**Interfaces:**
- Consumes: `modules/context-engineering/` (lesson, exercises, project, gate) and `tracks/*.yaml`.
- Produces: a recorded verdict per track, and either a clean slice or specific edits. Task 12's commit message quotes the verdict.

- [ ] **Step 1: Run the audit**

Invoke the `concept-mastery-validator` skill with this question, giving it the five module files and the three route files:

> For each track (foundations, practitioner, advanced), does the readiness gate require only
> knowledge that this module or an earlier module in the same route teaches? List every gate
> item whose required knowledge is not present in `lesson.md` or `exercises/` for that track,
> with the file and heading you would have to add to close the gap.

- [ ] **Step 2: Fix what it finds**

For each gap, either add the missing explanation to `lesson.md` (and cite it, if it is a
factual claim) or remove the gate item. Removing a gate item is usually right when the gate
over-reached; adding is right when the lesson skipped a step the project assumed.

- [ ] **Step 3: Re-run the guard and tests**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
python tools/check_citations.py
PYTHONPATH=. python -m unittest discover -s tests -t tests -v
```
Expected: `0 problem(s)` and all tests PASS.

- [ ] **Step 4: Write the journal entry, then commit**

Create `journal/015-mastery-audit.md` with the audit findings verbatim, which gaps you closed and which gate items you cut, and the Step 3 output.

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add journal/015-mastery-audit.md modules/
git commit -m "Audit the Context Engineering gate against what the module actually teaches"
git push origin main
```

---

### Task 12: License the course — only after the user confirms

This task is gated on an explicit yes. The spec records the licensing proposal, not a decision, so do not run these steps on assumption.

**Files:**
- Create: `LICENSE.md`
- Modify: `README.md` (only if the user chose something other than CC-BY-4.0 + MIT)

**Interfaces:**
- Consumes: the user's confirmation of: CC BY 4.0 for written content, MIT for code, attribution "Abdul Rahman P Z".
- Produces: the licence grant `README.md` describes.

- [ ] **Step 1: Ask for the decision**

Confirm with the user, in one message, the three parts: content licence, code licence, attribution string. If they choose differently, use their wording in Step 2 and update the README's License block in the same commit.

- [ ] **Step 2: Write `LICENSE.md`**

```markdown
# License

**Written content** — the lessons, exercises, gates, sources tables, journal entries and this
repository's documentation — is licensed under **Creative Commons Attribution 4.0 International
(CC BY 4.0)**.

You are free to share and adapt this material for any purpose, including commercially, so long
as you give attribution: "Abdul Rahman P Z / open-agents-course", a link to the licence, and an
indication of what you changed. The full legal code is at
https://creativecommons.org/licenses/by/4.0/legalcode.en

**Code** — everything under `tools/` and `modules/**/project/` — is licensed under the MIT
licence:

    MIT License

    Copyright (c) 2026 Abdul Rahman P Z

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.

**Third-party material is not licensed here.** Sources cited by this course remain the work of
their authors; `corpus/notes/` carries attributed summaries and short marked quotations, and
`corpus/raw/` holds fetched full text that is gitignored and never published. Where a note
adapts wording from another MIT-licensed project, the dual notice upstream + this licence
applies, with attribution as stated in that note.
```

- [ ] **Step 3: Commit**

```bash
cd /c/Users/abdul/Downloads/Qoder/tt/open-agents-course
git add LICENSE.md README.md journal/016-licensing.md
git commit -m "License the course: CC BY 4.0 for writing, MIT for code"
git push origin main
```

---

## Success criteria, mapped to tasks

| Spec criterion | Task |
|---|---|
| Slice 1 exists end to end with citations that validate | 3, 4, 5, 6, 7, 8, 9 (guard clean from 6 onward) |
| All three routes render from one module pool without duplicated prose | 9 (`test_routes_reference_the_same_module_pool_without_copying_prose`) |
| Every build step has a journal entry containing a real command and its real output | last-but-one step of every task |
| Citation and source checks pass locally before any publish | 1, 2, verified in 6, 9, 11 |
| Another person can reproduce the pipeline from the repo alone | 10 |
| Gates audited so projects require only what earlier modules taught | 11 |

## Not in this plan

Broadening the topic pool (memory, tool design, RAG, multi-agent, evals, guardrails, skills,
MCP, production), `kb/ingestion.json` and any QMind enrichment (blocked: no notebook can be
created from this session, and v1 ground truth is `corpus/`), the hosted site and
`web-preflight`, the `findable` discovery kit, GitHub Actions, slide decks and video, and any
second language variant of the code examples. Each content item is a later slice against the
same module contract; the spec's rule is that nothing broadens until this slice is genuinely
complete.
