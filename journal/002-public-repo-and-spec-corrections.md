# 002 — Went public, then found our own measurement error in the spec

**What we did:** created the public repo and pushed, read the committed spec with fresh
eyes, and fixed three defects. Two were housekeeping. One was a contradiction with our own
evidence, and it is worth keeping as the lesson of this entry.

```bash
GH="/c/Program Files/GitHub CLI/gh.exe"
"$GH" repo create arm-pz/open-agents-course --public --source=. --remote=origin --push \
  --description "A free, open AI agents course built from source-cited lessons..."
# branch 'main' set up to track 'origin/main'
# https://github.com/arm-pz/open-agents-course [PUBLIC]
```

## The self-caught error

The spec's opening line claimed the course differs from "the ~600 similar repos." That 589
came from `search/repositories?q=ai-agents-course+in:name`, which we debunked three hours
earlier in the same session: GitHub tokenises on hyphens, so the count is fuzzy token
matches, not name owners. The real crowding evidence was the incumbent list (four active
courses at 42-77 stars), not the headline number.

Same mistake, new file. Authoritative-sounding numbers get into prose because the number is
easy to remember and the caveat attached to it is not. Fix applied: the line now says
"crowded field of agent courses" and the measured facts stay in the Naming section, where
they are labelled with the method.

Other two fixes: the status header said "awaiting spec review" while the remote already
existed, and Open item 1 described creating the public repo in the future tense. Both moved
to a Resolved during setup section. The license item now says plainly that the corpus rules
depend on it being settled, because a repo cannot claim an open licence it has not applied.

## Design tooling reality

```bash
ls ~/.claude/skills/gstack/bin        # absent
browse/dist/browse -> NEEDS_SETUP     # needs bun, not installed
design/dist/design -> DESIGN_NOT_AVAILABLE
codex -> available
```

gstack's runtime binaries are not on this machine, so visual competitive research and
generated mockups are unavailable. The design work proceeds in text, with WebSearch as the
only research path if we want it. Installing bun for one preview image is not worth the
system change; if you want the visual path later, that is a separate call.

## Next

Design system for the course (typography, colour, layout, spacing) as `DESIGN.md`, then
`superpowers:writing-plans` for slice 1.
