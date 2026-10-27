# GitJog — a magyar jogrendszer git-natív verziókövetése

> **Hungarian legislation as a git repository.** 4,300+ acts in consolidated
> Markdown; each point-in-time version of an act is a commit dated on its date of
> entry into force (the SHAs are listed in `jogszabalyok/<slug>/allapotok.json`),
> so `git diff` between two of them is literally the amendment. History is not in
> date order: pick commits from `allapotok.json`, not with `--since` or `@{date}`.
> Updated daily. Public domain (CC0) — but **not an authentic source of law**, see
> [DISCLAIMER.md](DISCLAIMER.md). Browse it at
> [gitjog.remenyfarm.hu](https://gitjog.remenyfarm.hu).

Ez a repó a magyar törvények **konszolidált szövegét** tartalmazza Markdown
formátumban, úgy, hogy **a git history maga a jogtörténet**:

- **minden időállapot egy commit** — a commit dátuma a hatálybalépés napja (1970 előtti
  állapotnál 1970-01-01; a valódi dátum a commit-üzenetben és a `meta.json`-ban áll).
  Egy commit több jogszabály azonos napú állapotát is hordozhatja. Az `Index frissítés`
  és az `Enumerálás-térkép frissítés` commit nem időállapot, és egy jogszabály git
  logjában ismételt (visszajátszott) commitok is lehetnek. Egy jogszabály időállapotainak
  kanonikus commitjai: `jogszabalyok/<slug>/allapotok.json`,
- **a `git diff` két ilyen SHA között = a törvénymódosítás** — szó szerint látszik, mit vettek ki és mit tettek be,
- `git log --follow jogszabalyok/2013-evi-v-torveny-ptk/szoveg.md` — a Ptk. teljes módosítás-történet,
- `git blame` — megmutatja, melyik szakasz mikor változott utoljára,
- egy múltbeli időállapot: `git show <sha>:jogszabalyok/<slug>/szoveg.md`, ahol a `<sha>` a
  `jogszabalyok/<slug>/allapotok.json`-ból jön. Egy régi commit `git checkout`-ja **nem** az
  akkor hatályos joganyagot adja: a jogszabályok később, visszadátumozott commitokkal kerültek be.

## Példák

```bash
# Mely időállapotai léptek hatályba a Ptk.-nak 2024-ben?
# A SHA-t mindig az allapotok.json-ból vedd: a history nem dátum szerint rendezett, ezért
# a `git log --since/--until` itt üres eredményt ad, a `main@{dátum}` pedig a helyi
# reflogot nézi (klónban mindkét oldala a HEAD, a diff üres).
jq -r '.[] | select(.datum >= "2024-01-01" and .datum < "2025-01-01") | "\(.datum) \(.sha)"' \
  jogszabalyok/2013-evi-v-torveny-ptk/allapotok.json

# Két időállapot összevetése: a Btk. 2023-01-01-jén és 2025-01-01-jén hatályos szövege
B=jogszabalyok/2012-evi-c-torveny-btk
REGI=$(jq -r '[.[] | select(.datum <= "2023-01-01")] | max_by(.datum) | .sha' $B/allapotok.json)
UJ=$(jq -r '[.[] | select(.datum <= "2025-01-01")] | max_by(.datum) | .sha' $B/allapotok.json)
git diff "$REGI" "$UJ" -- $B/szoveg.md
```

Agent-receptek és API: [AGENTS.md](AGENTS.md). Ha egy leírás és az adat eltér, az adat
(`allapotok.json`, `meta.json`) a mérvadó.

## Szerkezet

A `jogszabalyok/`, az `index/` és az `AGENTS.md` a [GitJog pipeline](https://github.com/godavid/gitjog)
kimenete (lásd a Karbantartás szakaszt).

```
jogszabalyok/<slug>/szoveg.md        # konszolidált szöveg (HEAD = jellemzően a legutolsó felvett
                                     #   időállapot; biztosan: az allapotok.json utolsó SHA-ja).
                                     #   Nem feltétlenül hatályos jog: lásd enumeralas.json, lezart
jogszabalyok/<slug>/meta.json        # azonosítók, cím, forrás-URL (forras), időállapot-lista
jogszabalyok/<slug>/allapotok.json   # időállapotok: [{datum, verzio, sha}] — innen vedd a SHA-kat
index/jogszabalyok.json              # minden jogszabály: [{slug, documentId, megjeloles, cim, rovidites}]
index/allapotok.json                 # slugonként [{datum, verzio, sha}] (nagy fájl; a weboldal
                                     #   és a napi futás állapota)
index/felvetel/ÉÉÉÉ-HH.jsonl         # felvételi napló, soronként {felveve, slug, datum, sha}
index/enumeralas.json                # documentId → aktiv (hatályos vagy jövőbeli időállapota van) |
                                     #   lezart (hatályát vesztette) | nincs-szoveg (nincs a repóban);
                                     #   a napi futás ütemezése is
index/kihagyottak.json               # a backfill kihagyási listája okkal: [{documentId, slug, ok}];
                                     #   a napi futás nem frissíti, elavulhat (a friss lista: nincs-szoveg)
AGENTS.md                            # agent-útmutató; a gitjog sablonjából minden futás felülírja
CLAUDE.md                            # szimlink az AGENTS.md-re (ne szerkeszd, lásd AGENTS.md)
.github/workflows/napi-delta.yml     # a napi futás; a gitjog packages/pipeline/adatrepo/napi-delta.yml másolata
```

Egyetlen szöveg letöltéséhez nem kell klónozni:

```
https://raw.githubusercontent.com/godavid/magyar-jog/main/jogszabalyok/<slug>/szoveg.md
```

## ⚠️ Nem hiteles jogforrás

Ez a repó **tájékozódási és kutatási célra** készült, automatikus feldolgozással a
[Nemzeti Jogszabálytár](https://njt.jog.gov.hu) publikus felületéből. **Nem hiteles
szöveg** — a hiteles jogforrás a njt.jog.gov.hu és a Magyar Közlöny. Részletek:
[DISCLAIMER.md](DISCLAIMER.md).

## Frissítés

A repót napi automatikus futás (GitHub Actions) tartja karban: minden olyan új
időállapotot commitol, amely legkésőbb aznap hatályba lépett, és még nincs a repóban.
Egy időállapot napokkal a hatálybalépése után is bekerülhet: a hatályát vesztett és a
szöveg nélküli jogszabályokat a futás csak hetente kérdezi le, és az njt is később
teheti közzé az új szöveget. A bekerülés idejét a havi `index/felvetel/*.jsonl` napló
rögzíti, nem a commit dátuma. Hiba vagy gyanús adat esetén a futás `parser-riasztas`
címkés issue-t nyit ebben a repóban. A feldolgozó kód nyílt:
[godavid/gitjog](https://github.com/godavid/gitjog). Weboldal:
[gitjog.remenyfarm.hu](https://gitjog.remenyfarm.hu).

## Karbantartás (embereknek és agenteknek)

- A `jogszabalyok/`, az `index/` és az `AGENTS.md` a gitjog pipeline kimenete: kézzel ne
  szerkeszd, a hibát a [godavid/gitjog](https://github.com/godavid/gitjog) forrásában javítsd.
  Az `index/allapotok.json` és az `index/enumeralas.json` a napi futás állapota is. Az
  `AGENTS.md`-t minden futás felülírja a gitjog `packages/pipeline/src/sablonok.ts`
  `AGENTS_MD` sablonjából.
- A `.github/workflows/napi-delta.yml` a gitjog `packages/pipeline/adatrepo/napi-delta.yml`
  másolata: a kettőt mindig együtt módosítsd.
- A `main` történetét soha ne írd át (rebase, force-push, `git filter-repo`), és ne rendezd
  dátum szerint: az indexek és a felvételi napló commit-SHA-kra hivatkoznak. A később talált
  múltbeli időállapot új commit a history végén (gitjog `docs/uzemeltetes.md`, Ismert korlátok),
  és a korábbi backfill-futások is a meglévő commitok után kerültek be, ezért a history nem
  dátum szerint rendezett.
- A napi futás alatt ne pusholj és ne olvassz be semmit a `main`-re (a cron 03:30 UTC, de a
  GitHub jellemzően csak 08:00 és 11:00 UTC között indítja, és kb. 25 percig fut; ellenőrzés:
  `gh run list -R godavid/magyar-jog -w napi-delta.yml -L 1`). A futás a push előtt írja be a
  commit-SHA-kat az indexbe; ha közben a `main` elmozdul, a `git pull --rebase` után ezek a
  SHA-k a `main`-ről nem érhetők el.
- Üzemeltetés, riasztások, hibaelhárítás:
  [gitjog docs/uzemeltetes.md](https://github.com/godavid/gitjog/blob/main/docs/uzemeltetes.md).

## Licenc

A jogszabályszöveg a szerzői jogról szóló 1999. évi LXXVI. törvény 1. § (4)
bekezdése alapján nem áll szerzői jogi védelem alatt (közkincs). A repó saját
hozzáadott rétege — szerkezet, metaadatok, index-fájlok — [CC0 1.0](LICENSE)
alatt áll; magyar magyarázat: [LICENSE-HU.md](LICENSE-HU.md). Röviden: bármire
felhasználható, gépi feldolgozásra is, forrásmegjelölés nélkül.

A feldolgozó kód külön repóban él ([godavid/gitjog](https://github.com/godavid/gitjog)),
MIT licenc alatt.
