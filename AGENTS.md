# AGENTS.md — how to use this repository as an AI agent
<!-- GENERATED FILE. Source: godavid/gitjog packages/pipeline/src/sablonok.ts (AGENTS_MD). Do not edit here: a daily run that adds a version overwrites it. -->

This repository is the **full text and full amendment history of Hungarian acts of
Parliament** (4,300+ acts, 1827–today), maintained as a git repository:

- `jogszabalyok/<slug>/szoveg.md` — consolidated text of one act. HEAD is the **latest ingested
  version**, not necessarily the law in force (see "What HEAD means" below)
- `jogszabalyok/<slug>/meta.json` — identifiers, title, source URL (`forras`), list of point-in-time versions
- `jogszabalyok/<slug>/allapotok.json` — versions of this act: `[{datum, verzio, sha}]` — **take commit SHAs from here**
- `index/jogszabalyok.json` — list of all acts: `[{slug, documentId, megjeloles, cim, rovidites}]` (official designation, title, abbreviation)
- `index/allapotok.json` — all versions of all acts, keyed by slug (3+ MB; prefer the per-act file)
- `index/felvetel/YYYY-MM.jsonl` — **ingestion log**: one line per version added, with the time it entered the repo
- `index/enumeralas.json` — `documentId` → layer: `aktiv` (has a version in force or a future one) |
  `lezart` (repealed) | `nincs-szoveg` (no consolidated text on njt, so not in this repo)
- `index/kihagyottak.json` — snapshot from the initial backfill of acts without a consolidated text,
  with reasons (`[{documentId, slug, ok}]`); the daily run cleans it but does not maintain it. The
  authoritative list of missing acts is the `nincs-szoveg` layer of `enumeralas.json`, not this file

Key names are ASCII (no accents): `datum`, `felveve`, `verzio`, `megjeloles`, `cim`, `rovidites`, `kovetkezo_felveve_utan`.

**Not an authentic source of law.** Derived automatically from https://njt.jog.gov.hu; the
authentic text is there and in Magyar Közlöny. Say so when you quote it. Public domain (CC0).

## What HEAD means

HEAD of `jogszabalyok/<slug>/szoveg.md` is the last version the pipeline ingested for that act
(the last SHA in its `allapotok.json`). It is **not necessarily the law in force today**:

- a repealed act (`lezart` layer) keeps its last text at HEAD;
- a version that entered into force recently may not be in the repo yet (see Caveats: lag).

The daily run does not ingest a version before its entry-into-force date.

To get the text in force on a given day, pick from `allapotok.json` the entry with the largest
`datum` that is `<=` that day, and read it with `git show <sha>:jogszabalyok/<slug>/szoveg.md`.

## How commits work

- **One state commit per entry-into-force day in each run**, carrying every act ingested for that day
  in that run. A later run may add another commit for the same day. The commit
  date (author and committer) is the **date of entry into force**, not the time the version was
  added (the ingestion log has that). Dates before 1970 are clamped to 1970-01-01 in git metadata;
  the real date is in the commit message and in `meta.json`/`allapotok.json`.
- `Index frissítés …` and `Enumerálás-térkép frissítés` commits are pipeline bookkeeping with
  wall-clock dates. They are not versions of any act.
- **History is not date-ordered.** Acts were backfilled in several runs, and a version found later
  is appended at the end of history with a back-dated commit. Therefore `git log --since/--until`,
  `git log --until`, `main@{<date>}` and `git checkout` of an "old" commit do **not** give you the
  law of that day. Take SHAs from `allapotok.json`, and **always give a path filter** to `git log`
  (`-- jogszabalyok/<slug>/`), otherwise index commits and other acts flood the output.
- `git diff <sha_old> <sha_new> -- jogszabalyok/<slug>/szoveg.md` between two SHAs from
  `allapotok.json` is literally the amendment.
- The amending act that caused a change is **not** recorded in the text (the consolidated source
  does not carry it). The commit message lists the njt source URLs of the ingested versions
  (`Forrás (njt):` lines); that URL is the version's own njt page, which names the amending acts.

## Cheapest way in: the API and the MCP server

Do not read whole `szoveg.md` files (most are under 100 KB, the largest is 2.33 MB) into
context. The website exposes section-level (§) endpoints, no key needed:

- MCP (streamable HTTP): `https://gitjog.remenyfarm.hu/api/mcp`
- REST + OpenAPI: `https://gitjog.remenyfarm.hu/api/v1/openapi.json`
- Human-readable guide: `https://gitjog.remenyfarm.hu/llms.txt`

Tools / endpoints (all read-only):

| tool | what it answers |
|---|---|
| `kereses` | "which § regulates X?" and reference resolution ("Ptk. 6:272. §", "2013. évi CXXII. törvény") — returns the § text, ready to cite |
| `szakasz` | one § at a given date; without a § it returns the act's metadata, versions and table of contents |
| `valtozasok` | "what changed since I last looked?" — cursor-based: start with an ISO-8601 timestamp in `felveve_utan`, then pass the response's `kovetkezo_felveve_utan` back **unchanged** (an opaque string, e.g. `2026-10-06T17:43:08Z_2013-evi-v-torveny-ptk_2026-10-01`); pages are oldest-first, gap-free and duplicate-free; returns the affected §§ with old/new text |
| `diff` | full §-level comparison of two versions |

## Working from a clone (git-native recipes)

```bash
# one act only: a blobless partial clone (--filter=blob:none) keeps the FULL history and
# fetches file contents on demand; it is not a shallow clone
git clone --filter=blob:none --sparse https://github.com/godavid/magyar-jog
cd magyar-jog && git sparse-checkout set jogszabalyok/2013-evi-cxxii-torveny-foldforgalmi index

# PRIMARY RECIPE — every version of an act with its commit SHA (dates = entry into force)
jq -c '.[] | [.datum, .sha]' jogszabalyok/2013-evi-cxxii-torveny-foldforgalmi/allapotok.json

# the text in force on a given day
S=jogszabalyok/2013-evi-cxxii-torveny-foldforgalmi
SHA=$(jq -r '[.[] | select(.datum <= "2024-01-01")] | max_by(.datum) | .sha' $S/allapotok.json)
git show "$SHA:$S/szoveg.md"

# what changed between two versions (SHAs from allapotok.json)
git diff <sha_old> <sha_new> -- jogszabalyok/<slug>/szoveg.md

# the amendment history as git sees it (ALWAYS with a path filter; order is not by date)
git log --date=short --format='%ad %h %s' -- jogszabalyok/<slug>/szoveg.md

# which version last touched a given line
git blame -L 120,140 -- jogszabalyok/<slug>/szoveg.md

# what was added to the repository since a timestamp (all acts; log starts 2026-09-21,
# one file per UTC month, keys {felveve, slug, datum, sha})
cat index/felvetel/*.jsonl | jq -c 'select(.felveve > "2026-09-15T00:00:00Z")'
```

Structural units (book, part, chapter, subtitle) and many sections (§) are Markdown headings (`##`–`####`),
but not all: in many acts a § is plain text (e.g. `18. § (1) …`) under the nearest heading, and some acts have no headings at all.
A heading such as `6:272. § [Megbízási szerződés]` (Ptk.) has the anchor `6-272-sz-megbizasi-szerzodes` on the
website (`/jogszabaly/<slug>#<anchor>`). To get a § by number, call `szakasz` with `paragrafus` (e.g. `18. §`);
do not build the anchor yourself.

## Caveats

- The daily run is scheduled at 03:30 UTC, but GitHub usually starts it between 08:00 and 11:00 UTC,
  and it runs for about 25 minutes.
- **Lag:** a version often appears days after its entry into force (228 of 397 versions ingested in
  the first weeks of the log arrived 2–7 days late): njt publishes consolidated texts with a delay,
  and repealed or text-less acts are polled only weekly. Use `felveve` in the ingestion log, not
  the commit date, to know when something entered the repo.
- Which amending act caused a change is not recorded in the text; the njt URL in the commit
  message (`Forrás (njt):`) is the trail.
- Acts without a consolidated text on njt.jog.gov.hu are missing (about 1,250; the `nincs-szoveg`
  layer of `index/enumeralas.json`).
- Dates before 1970 are clamped in git metadata; the real date is in the commit message and `meta.json`.

## Traps (ranked by how often they bite)

1. `git log --since/--until`, `git log @{date}`, `main@{<date>}`: history is not date-ordered and
   `@{date}` reads the local reflog. Both give empty or wrong results. Use `allapotok.json`.
2. `git checkout <old commit>` is **not** the law of that day: acts were ingested later with
   back-dated commits, so the tree at that commit lacks everything backfilled afterwards.
3. The largest `verzio` number is not the latest version: for 282 acts njt version numbers are
   not monotonic with the date. Sort by `datum`.
4. Keys are ASCII (`datum`, `felveve`, `verzio`, `cim`); there are no accented JSON keys.
5. `index/kihagyottak.json` is a backfill snapshot, not the list of missing acts; use the
   `nincs-szoveg` layer of `index/enumeralas.json`.
6. `git show <sha>` on an `Index frissítés` or `Enumerálás-térkép frissítés` commit shows pipeline
   bookkeeping, not a version of an act. Take SHAs only from `allapotok.json`.
7. Reading a whole `szoveg.md` into context (up to 2.33 MB) wastes it; use the `szakasz` and
   `kereses` tools, or `git show <sha>:<path>` with a line range.

## Vocabulary

| term | meaning |
|---|---|
| `datum` | date of entry into force of a version (`YYYY-MM-DD`); the commit date of its commit |
| `felveve` | ISO-8601 time the version entered this repository (ingestion log); never equals the commit date |
| `verzio` | njt's own version number of a version; not monotonic with `datum`, do not sort by it |
| `documentId` | njt identifier of an act (`YYYY-NUMBER-00-00`) |
| `slug` | URL and directory name of an act in this repository (`jogszabalyok/<slug>/`) |
| `megjeloles` | official designation of an act ("2013. évi V. törvény") |
| `aktiv` / `lezart` / `nincs-szoveg` | layers in `index/enumeralas.json`: in force or future version exists / repealed / no consolidated text |

## Maintaining this repository

- `jogszabalyok/`, `index/` and `AGENTS.md` are output of the GitJog pipeline
  (https://github.com/godavid/gitjog). Never hand-edit them: fix the pipeline instead. `AGENTS.md`
  is rewritten from the `AGENTS_MD` template in `packages/pipeline/src/sablonok.ts` on every daily
  run that adds a version, so a hand edit is lost.
- `.github/workflows/napi-delta.yml` is a byte-identical copy of
  `packages/pipeline/adatrepo/napi-delta.yml` in gitjog: update both together.
- The `main` history is never rewritten (no rebase, force-push or `git filter-repo`) and never
  reordered: the indexes and the ingestion log reference commit SHAs.
- Do not push to `main` while the daily run is active (03:30 UTC cron, usually started 08:00–11:00
  UTC, about 25 minutes): the run pushes its version commits first and then builds the index from
  the pushed SHAs; a concurrent push makes it rebase, and a conflicting change stops the run.
- Operations runbook: https://github.com/godavid/gitjog/blob/main/docs/uzemeltetes.md. Human
  overview and licence: [README.md](README.md), [LICENSE](LICENSE), [LICENSE-HU.md](LICENSE-HU.md).
