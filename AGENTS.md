# AGENTS.md — how to use this repository as an AI agent

This repository is the **full text and full amendment history of Hungarian acts of
Parliament** (4,300+ acts, 1827–today), maintained as a git repository:

- `jogszabalyok/<slug>/szoveg.md` — consolidated text of one act (HEAD = current state)
- `jogszabalyok/<slug>/meta.json` — identifiers, title, source URL, list of point-in-time versions
- `jogszabalyok/<slug>/allapotok.json` — versions of this act: `[{datum, verzio, sha}]`
- `index/jogszabalyok.json` — list of all acts: slug, documentId, megjelölés (official designation), cím (title), rövidítés (abbreviation)
- `index/allapotok.json` — all versions of all acts (4+ MB; prefer the per-act file)
- `index/felvetel/YYYY-MM.jsonl` — **ingestion log**: one line per version added, with the time it entered the repo

One commit = one point-in-time version; **the commit date is the date of entry into force**
(both author and committer date), so `git diff` between two commits is literally the amendment.
The commit date is *not* the time the version was added — use the ingestion log for that.

**Not an authentic source of law.** Derived automatically from https://njt.jog.gov.hu; the
authentic text is there and in Magyar Közlöny. Say so when you quote it. Public domain (CC0).

## Cheapest way in: the API and the MCP server

Do not read whole `szoveg.md` files (0.1–1.4 MB each) into context. The website exposes
section-level (§) endpoints, no key needed:

- MCP (streamable HTTP): `https://gitjog.remenyfarm.hu/api/mcp`
- REST + OpenAPI: `https://gitjog.remenyfarm.hu/api/v1/openapi.json`
- Human-readable guide: `https://gitjog.remenyfarm.hu/llms.txt`

Tools / endpoints (all read-only):

| tool | what it answers |
|---|---|
| `kereses` | "which § regulates X?" and reference resolution ("Ptk. 6:272. §", "2013. évi CXXII. törvény") — returns the § text, ready to cite |
| `szakasz` | one § at a given date; without a § it returns the act's metadata, versions and table of contents |
| `valtozasok` | "what changed since I last looked?" — cursor-based (`felveve_utan`), returns the affected §§ with old/new text |
| `diff` | full §-level comparison of two versions |

## Working from a clone (git-native recipes)

```bash
# only one act, shallow history is fine for the current text
git clone --filter=blob:none --sparse https://github.com/godavid/magyar-jog
cd magyar-jog && git sparse-checkout set jogszabalyok/2013-evi-cxxii-torveny-foldforgalmi index

# every amendment of an act (dates = entry into force)
git log --date=short --format='%ad %h %s' -- jogszabalyok/2013-evi-cxxii-torveny-foldforgalmi/szoveg.md

# what changed between two versions (pick SHAs from allapotok.json)
git diff <sha_old> <sha_new> -- jogszabalyok/<slug>/szoveg.md

# which version last touched a given line
git blame -L 120,140 -- jogszabalyok/<slug>/szoveg.md

# what was added to the repository since a timestamp (all acts)
cat index/felvetel/2026-09.jsonl | jq -c 'select(.felveve > "2026-09-15T00:00:00Z")'
```

Sections (§) are Markdown headings (`##`–`####`). A heading such as `18. § [Elővásárlási jog]`
has the anchor `18-sz-elovasarlasi-jog` on the website (`/jogszabaly/<slug>#<anchor>`).

## Caveats

- Which amending act caused a change is **not** recorded (the consolidated source does not carry it).
- Updated once a day (03:30 UTC); a version appears on the day it enters into force.
- Acts without a consolidated text on njt.jog.gov.hu are missing (about 1,250).
- Dates before 1970 are clamped in git metadata; the real date is in the commit message and `meta.json`.
