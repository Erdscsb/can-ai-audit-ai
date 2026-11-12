# Can AI Audit AI? — NIS2 / 2024/2690 pilot kutatási csomag

**Állapot (2026-11): a pilot lefutott, a nyers API-válaszok és metrikák a `results/` mappában vannak.** A számok az előzetes, elemzői címkékkel való egyezést mutatják, nem független jogi validációval igazolt pontosságot. Angol nyelvű összefoglaló az eredményekkel: [README.md](README.md).

## Mi van benne?

- `controlled_cases.csv` — **40** tesztállítás **20** jogszabályi mellékletponthoz, 20 helyes és 20 szándékosan hibás. A tesztadat szövege ANGOL.
- `anchor_reference.csv` — forráshivatkozások és elemzői normatív összefoglalók; a független jogi validáció státusza jelenleg **PENDING**.
- `study.py` — API-tesztfuttató három szolgáltatóhoz; részeredményeket minden hívás után elment.
- `controlled_cases.json` — a tesztkészlet programozáshoz.
- `results/` — a futtatás után a válaszok, metrikák, ábrák. Kezdetben üres.
- `AI_Audit_AI_Pilot_Dataset.xlsx` — áttekinthető munkafüzet a kontrollált adathalmazról és a validációs feladatokról.

## A legfontosabb módszertani különbség

**Kontrollált hibafelismerési kísérlet:** szakértői felülvizsgálatra váró helyes/hibás állítások, modellre elrejtett gold címkékkel. Ugyanazokat vizsgálja Gemini forrás nélkül, Gemini elemzői referencia-összefoglalóval, DeepSeek forrás nélkül. Ez **nem önellenőrzési kísérlet**.

**Természetes AI-generálás és modellazonos ellenőrzés:** külön `generate` lépés OpenAI-modellel létrehozott állításokhoz, majd `run-natural` hívás ugyanezen modellazonosítóval (`self`) és független modellel (`independent`). A `self` itt azonos modellcsalád általi **új kontextusú felülvizsgálat**, nem ugyanannak a gondolatmenetnek vagy beszélgetésnek a belső újraértékelése. A generált állításoknak nincs automatikus gold címkéje. Ezek manuális jogi értékelés nélkül **NEM** szolgálnak pontossági eredményként.

A `*_reference` mód **elemző által írt parafrázist** mutat be, nem hivatalos jogszabályi idézetet. A közléskor ezért a megfelelő kifejezés: **reference-assisted verification**. A `*_source` valódi forrásalapú teszt csak akkor indul, ha a `source_excerpt` mezőkbe hiteles jogszabályrészletek kerültek és a forrás ellenőrzött. Ezt **elég a 20 `anchor_reference.csv` soron** kitölteni: `verbatim_source_excerpt` + `independent_validation=VERIFIED`, majd futtasd `py study.py apply-sources`. Ez a 40 esethez szinkronizálja a szöveget. Javasolt ezt a teljes éles futtatás előtt pótolni.

## Előkészítés — Windows PowerShell

1. Csomagold ki a ZIP-et egy mappába. Legyen telepítve Python 3.10 vagy újabb.
2. A `requests` csomag nem szükséges. Az ábrákhoz opcionálisan: `py -m pip install matplotlib`
3. PowerShell ablakban **csak a saját számítógépeden** add meg a kulcsokat, pl.:

```powershell
$env:OPENAI_API_KEY="IDE_A_SAJAT_OPENAI_API_KULCS"
$env:GEMINI_API_KEY="IDE_A_SAJAT_GEMINI_API_KULCS"
$env:DEEPSEEK_API_KEY="IDE_A_SAJAT_DEEPSEEK_API_KULCS"
```

A kulcsokat **soha ne másold be a ChatGPT-beszélgetésbe**, ne tárold munkafüzetben, és ne töltsd fel GitHubra.

A modellek beállíthatók, például:

```powershell
$env:OPENAI_MODEL="gpt-5-mini"
$env:GEMINI_MODEL="gemini-3.8-flash"
$env:DEEPSEEK_MODEL="deepseek-flash"
```

Ezek csak példák: az adott API-fiókodban elérhető modellazonosítókat ellenőrizd. A DeepSeek régi `deepseek-chat` neve 2026 júliusában megszűnt; a jelenlegi Flash azonosító `deepseek-flash`. A beállítások és az esetleges API-korlátok szolgáltatónként változhatnak.

## Első teszt, legfeljebb néhány hívással

```powershell
py study.py validate
py study.py run-controlled --limit 2
py study.py run-controlled --limit 2 --modes gemini_closed,gemini_reference --execute
py study.py report
```

- `--execute` nélkül a program nem küld fizetős hívást.
- A teszt az alapbeállításokban 2 eset × 2 mód = legfeljebb **4 API-hívás**.
- A `results/controlled_responses.jsonl` minden sikeres válasz után frissül. A program hibákat is naplóz; ismételt futtatás csak a még nem sikeres párokat hívja.
- A statisztikák több ismétlés esetén ismétlésenként készülnek el (nem tesznek úgy, mintha a különböző futások független jogszabályi esetek lennének).

## Teljes kontrollált pilot

```powershell
py study.py run-controlled --modes gemini_closed,gemini_reference,deepseek_closed --execute
py study.py report
```

Ha forrásalapú és nem parafrázisos összehasonlítást szeretnél, előbb `py study.py apply-sources`, utána `py study.py run-controlled --modes gemini_closed,gemini_source,deepseek_closed --execute`.

Ez **120 értékelés** az eredeti 40 állításon. Ha kevés az API-keret, futtathatod csak a két Gemini-feltételt is (80 hívás), és DeepSeek nélkül is megmarad a kontrollált referenciahozzáférés-vizsgálat.

## Második, különálló modellazonos audit track

```powershell
py study.py generate --limit 10 --execute
py study.py run-natural --limit 10 --modes self,independent,reference --execute
```

Ekkor létrejön `natural_candidates.csv`. A **`manual_gold_verdict` mezőket manuálisan töltsd ki** `ACCEPT` vagy `REJECT` értékkel, és dokumentáld a jogi döntést. Utána `py study.py report`. A fenti kb. 10 generálás + 30 ellenőrzés kizárólag **ellenőrzés előtti**, és nem garantáltan hibás AI-generált mintát eredményez; emiatt alacsony lehet a valódi hibaesetek száma.

## Főbb mérőszámok

- **Error detection recall**: hibás állítások közül mekkora arányt utasított el az ellenőr.
- **False acceptance rate**: hibás állítások közül hányat fogadott el helyesnek.
- **False rejection rate**: helyes állítások közül hányat utasított el.
- **Abstention rate**: bizonytalansági döntések aránya.
- **Accuracy**: teljes esethalmazra vonatkozó helyes döntések aránya; az `UNCERTAIN` a nevezőben marad, nem helyes besorolás.

**Lényeges korlátok:** 40 eset/20 forráspont, párhuzamos ellenőrzési módok, elemzői és nem függetlenül hitelesített címkék, manuálisan szerkesztett hibák, a `*_reference` ágba az arany címkézés alapját is jelentő paraphrázis kerül, valamint az egyes modellek verzió- és szolgáltatófüggősége. Az ebből származó százalékok kizárólag pilotjellegűek; statisztikai általánosítást vagy emberi auditor helyettesítését nem támasztják alá.

## Felhasznált szabályozási forrás és az alkalmazhatóság

- A Bizottság (EU) **2024/2690** végrehajtási rendeletének **melléklete**, az alkalmazási körre tekintettel (nem az összes NIS2 érintett ágazat általános kontrolllistája): https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R2690
- ENISA Technical Implementation Guidance (2025; nem kötelező erejű útmutató): https://www.enisa.europa.eu/publications/nis2-technical-implementation-guidance

**Jogi QA:** a 20 hely és a hozzájuk írt állítások előzetesen ellenőrzött, elemzői tesztanyagok. A `anchor_reference.csv` `independent_validation` mezőit egy független szakértőnek célszerű igazolnia a tudományos közlés előtt. A modellválaszok, hivatkozások és vitás referenciacímkék emberi ellenőrzése szükséges.

## Előadás: ajánlott felépítés

1. Motiváció: miért veszélyes a téves AI-audit jóváhagyás.
2. NIS2 / 2024/2690 alkalmazási kör; források.
3. A 20 páros jogszabályi feladat és a hibakategóriák.
4. Vak tesztek, három konfiguráció; gold címkék elkülönítése.
5. Eredmények: detection recall, false acceptance, abstention. **Csak futtatott, ellenőrzött értékekkel.**
6. Egy konkrét, ténylegesen megfigyelt false acceptance részletesen.
7. Természetes generálás és modellazonos audit: külön kutatási ág / kvalitatív eredmény.
8. Korlátok, emberi kontroll, következő kutatási lépések.

### Fontos: CSV és XLSX

A program a `controlled_cases.csv` fájlt olvassa, **nem** az Excel-munkafüzetet. Az XLSX vizuális ellenőrzésre szolgál. Ha javítod az Excelben a jogi hivatkozást, az adatot a CSV-ben is aktualizáld (UTF-8 CSV export), különben az API-futtatás a régi adatot használja.
