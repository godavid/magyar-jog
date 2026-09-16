# GitJog — a magyar jogrendszer git-natív verziókövetése

> **Hungarian legislation as a git repository.** 4,300+ acts in consolidated
> Markdown; one commit = one point-in-time version (the commit date is the date of
> entry into force), so `git diff` is literally the amendment. Updated daily.
> Public domain (CC0) — but **not an authentic source of law**, see
> [DISCLAIMER.md](DISCLAIMER.md). Browse it at
> [gitjog.remenyfarm.hu](https://gitjog.remenyfarm.hu).

Ez a repó a magyar törvények **konszolidált szövegét** tartalmazza Markdown
formátumban, úgy, hogy **a git history maga a jogtörténet**:

- **egy commit = egy időállapot** — a commit dátuma a hatálybalépés napja,
- **a `git diff` = a törvénymódosítás** — szó szerint látszik, mit vettek ki és mit tettek be,
- `git log --follow jogszabalyok/2013-evi-v-torveny-ptk/szoveg.md` — a Ptk. teljes módosítás-történet,
- `git blame` — megmutatja, melyik szakasz mikor változott utoljára,
- `git checkout` egy múltbeli commitra — az akkor hatályos állapot.

## Példák

```bash
# Mit módosított a jogalkotó a Ptk.-n 2024-ben?
git log --since=2024-01-01 --until=2025-01-01 -- jogszabalyok/2013-evi-v-torveny-ptk/

# Két időállapot összevetése
git diff 'main@{2023-01-01}' 'main@{2025-01-01}' -- jogszabalyok/2012-evi-c-torveny-btk/szoveg.md
```

## Szerkezet

```
jogszabalyok/<slug>/szoveg.md   # a konszolidált szöveg (HEAD = aktuális állapot)
jogszabalyok/<slug>/meta.json   # azonosító, cím, forrás-URL, időállapot-lista
index/jogszabalyok.json         # az összes jogszabály listája
index/allapotok.json            # időállapot → commit SHA térkép (a weboldal használja)
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

A repót napi automatikus futás (GitHub Actions) tartja karban: az aznap hatályba
lépő új időállapotokat commitolja. A feldolgozó kód nyílt:
[godavid/gitjog](https://github.com/godavid/gitjog). Weboldal:
[gitjog.remenyfarm.hu](https://gitjog.remenyfarm.hu).

## Licenc

A jogszabályszöveg a szerzői jogról szóló 1999. évi LXXVI. törvény 1. § (4)
bekezdése alapján nem áll szerzői jogi védelem alatt (közkincs). A repó saját
hozzáadott rétege — szerkezet, metaadatok, index-fájlok — [CC0 1.0](LICENSE)
alatt áll; magyar magyarázat: [LICENSE-HU.md](LICENSE-HU.md). Röviden: bármire
felhasználható, gépi feldolgozásra is, forrásmegjelölés nélkül.

A feldolgozó kód külön repóban él ([godavid/gitjog](https://github.com/godavid/gitjog)),
MIT licenc alatt.
