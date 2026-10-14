# 001 — Setup, feasibility probes, and the name decision

**What we did:** scoped the project, installed assisting skills, tested the two
infrastructure assumptions the design rested on, audited candidate names against real
collision data, initialised the repo, and wrote the design spec.

**Why this order:** two of the plan's pillars were unverified assumptions — that a QMind
notebook could hold our corpus, and that we could push to GitHub. Both were cheap to check
and one turned out to be blocked. Finding that out before writing ten modules is the whole
point of probing first.

## Commands run, and what they told us

Skills discovered and installed at user scope:

```bash
npx --yes skills@1.5.22 find -- 'curriculum'
npx --yes skills@1.5.22 add 'mattpocock/skills@teach' -g -a qoder -y
npx --yes skills@1.5.22 add 'jorgealves/agent_skills@module-learning-path-generator' -g -a qoder -y
npx --yes skills@1.5.22 add 'jorgealves/agent_skills@concept-mastery-validator' -g -a qoder -y
npx --yes skills@1.5.22 list -g -a qoder
```

Installed: `teach`, `module-learning-path-generator`, `concept-mastery-validator`.
`partme-ai/full-stack-skills@course-designer` was selected but its own repo no longer
exposes it: `No matching skills found for: course-designer`. Dropped, not worked around.

Feasibility check 1 — can we create the RAG notebook?

```bash
mcp__plugin_qoder-qmind_qoder-qmind__list_notebooks   -> totalSize: 0
mcp_get add_source -> required: ["notebookId","source"]
```

No. `add_source` needs an existing `notebookId` and QMind exposes no create operation. The
design changed: v1 retrieval runs against `corpus/notes/` through a committed local script,
and the notebook becomes later enrichment. Citations are source ids either way, so nothing
gets rewritten when it arrives.

Feasibility check 2 — can we push?

```bash
which -a gh git
# /c/Users/abdul/AppData/Roaming/npm/gh   <- broken npm stub
# /c/Program Files/GitHub CLI/gh          <- real CLI

"/c/Program Files/GitHub CLI/gh.exe" auth status
# ✓ Logged in to github.com account arm-pz
# Token scopes: 'gist', 'read:org', 'repo'
```

Yes, via the full path. The `gh` earlier on PATH is an unrelated Node package that throws
`TypeError: Cannot read properties of undefined (reading 'options')`.

Name audit — raw counts first, then honest counts:

```bash
"$GH" api "search/repositories?q=agents-from-scratch+in:name&per_page=1" --jq .total_count   # 243
"$GH" api "search/repositories?q=ai-agents-course+in:name&per_page=1"     --jq .total_count   # 589
"$GH" api "search/repositories?q=build-your-own-ai-agent+in:name&per_page=1" --jq .total_count # 31
```

Then the incumbents behind those numbers, which is what actually matters:

```bash
"$GH" api "search/repositories?q=agents-from-scratch+in:name&sort=followers&per_page=6"
# 4846 pguso/ai-agents-from-scratch
# 2386 langchain-ai/agents-from-scratch
# 1045 pguso/agents-from-scratch
#  847 langchain-ai/deep-agents-from-scratch
#  192 nerdai/llm-agents-from-scratch
```

And the correction that changed the recommendation: GitHub tokenises on hyphens, so
`in:name` counts are fuzzy. Phrase-matching is the real measure:

```bash
"$GH" api "search/repositories?q=%22open-agents-course%22+in:name&per_page=1" --jq .total_count   # 0
"$GH" api "search/repositories?q=%22open-agent-course%22+in:name&per_page=1"  --jq .total_count   # 2
curl -s -o /dev/null -w "%{http_code}" https://api.github.com/repos/arm-pz/open-agents-course     # 404 = free
```

`open-agents-course`: zero exact-name owners, singular variant has two. "Curriculum" was
rejected on reader grounds, not search grounds — plain "course" is understood the same way
by a first-time visitor, a non-native speaker, and a crawler.

## Lesson

Counts lie in both directions. The `ai-agents-course` number that looked crowded (589) and
the `build-your-own-ai-agent` number that looked free (31) both described token matches, not
name owners — and one of the small counts had a direct competitor at 97 stars with an exact
name match. The decisive evidence was the incumbent list per candidate, not the headline
count. Same trap as the earlier `list_notebooks` probe: an empty result looked like "no
knowledge base exists" when the real finding was "no creation path exists."

## Next

Slice 1 on Context Engineering: manifest entries → fetch → `corpus/notes` → local retrieval
→ `modules/context-engineering/` with citations → three routes wired → `tools/check-citations.py`.
