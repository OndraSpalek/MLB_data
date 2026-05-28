# MLB_data — datový repozitář pro BE6PLAYER MLB Monte Carlo Simulator

Tento repozitář slouží jako zdroj dat pro aplikaci `baseball_model_v23.html` (Monte Carlo simulátor MLB zápasů). Aplikace si při startu automaticky stáhne všechny soubory přes GitHub raw URL a načte je místo manuálního drag-and-drop uploadu.

## 📁 Soubory v repozitáři

Všechny soubory leží v rootu repozitáře a načítají se z větve `main`:

| Soubor | Obsah | Povinné |
|---|---|---|
| `fangraphs.xlsx` | Statistiky nadhazovačů, bullpenu, pálky a park factors | ✅ ano |
| `schedule.xlsx` | Rozpis sezóny (zápasy, datumy, týmy, stadiony) | ✅ ano |
| `priors.xlsx` | Preseason projekce (Steamer/ZiPS) — Bayesovský prior | ⛔ ne, volitelné |

URL formát:

```
https://raw.githubusercontent.com/OndraSpalek/MLB_data/main/<soubor>
```

Aplikace má fallback chain proxy (přímo → corsproxy.io → allorigins.win → codetabs.com), takže funguje i když GitHub vrátí dočasnou CORS chybu. Když fetch selže úplně, padne to na localStorage cache nebo manual upload.

---

## 📊 fangraphs.xlsx — struktura listů

Všechna data se exportují z FanGraphs (sezónní leaderboards). Workbook musí obsahovat těchto **8 listů** (parser hledá podle názvu, případně regex matchu):

### Starting pitcher (nebo Pitcher)
Řádek = jeden startující nadhazovač. Povinné sloupce:

| Sloupec | Význam |
|---|---|
| Name | Jméno nadhazovače (přesně jak v FanGraphs) |
| Team | Zkratka týmu (ATL, NYY, LAD, …) |
| ERA | Earned Run Average — pouze info, není prediktor |
| FIP | Fielding Independent Pitching |
| xFIP | Expected FIP (normalizovaný HR/FB) |
| SIERA | Skill-Interactive ERA — hlavní prediktor |
| BABIP | Batting Average on Balls in Play |
| K-BB% | Strikeout minus walk rate (procento, např. 18.5) |
| IP | Innings pitched (váha pro Bayesovský shrinkage) |

### Batting — pálka týmu (overall)
Fallback pro případy, kdy chybí ruka nadhazovače. Řádek = tým.

| Sloupec | Význam |
|---|---|
| Team (nebo Tm) | Zkratka týmu |
| wRC+ | Weighted Runs Created Plus (100 = ligový průměr) |
| PA | Plate Appearances (váha pro shrinkage) |

### Batting vs LHP — pálka proti levákům
Stejné sloupce jako Batting, ale data jen z plate appearances proti levorukým nadhazovačům.

### Batting vs RHP — pálka proti pravákům
Stejné sloupce jako Batting, ale data jen z plate appearances proti pravorukým nadhazovačům.

### Pitcher hand — ruka nadhazovače
Mapuje jméno nadhazovače na jeho házecí ruku.

| Sloupec | Význam |
|---|---|
| Name | Jméno nadhazovače (musí se shodovat s listem Starting pitcher) |
| Pitcher hand (nebo Hand) | LH (levák) nebo RH (pravák) |

### Relieving pitcher — bullpen
Agregovaná statistika bullpenu pro každý tým. Řádek = tým.

| Sloupec | Význam |
|---|---|
| Team | Zkratka týmu |
| ERA, FIP, xFIP, SIERA, BABIP, K-BB%, IP | Stejné jako u startérů, ale agregované za celý bullpen |

### Park factors — faktor hřiště
Multiplikativní úprava očekávaných runs podle stadionu.

| Sloupec | Význam |
|---|---|
| Team | Plný název týmu (např. „Atlanta Braves" — pozor, nikoliv zkratka) |
| Basic (5yr) | Park factor (100 = neutrální, >100 = více runs než průměr) |

### pitcher_last14days — rolling form *(volitelné, nové v v23)*
FIP každého startéra za posledních 14 dní z FanGraphs. Pokud list chybí, model použije jen sezónní SIERA.

| Sloupec | Význam |
|---|---|
| Name | Jméno nadhazovače (musí se shodovat s listem Starting pitcher) |
| FIP | FIP za posledních 14 dní |

---

## 📅 schedule.xlsx — rozpis sezóny

Jeden list, řádek = jeden zápas. Parser zkouší více aliasů názvů sloupců (česky/anglicky).

| Sloupec | Aliasy | Význam |
|---|---|---|
| Datum | Date | ISO YYYY-MM-DD nebo DD.MM.YYYY |
| Čas (ET) | Time (ET), Time | Začátek zápasu v Eastern Time |
| Host (tým) | Away, Away Team | Plný název hostujícího týmu |
| Zkr. hosta | Away Abbr | Zkratka (např. NYY) |
| Domácí (tým) | Home, Home Team | Plný název domácího týmu |
| Zkr. domácích | Home Abbr | Zkratka (např. BOS) |
| Stadion | Venue, Park | Název stadionu |
| Status | Stav | Plánováno / Hraje se / Hotovo |
| Skóre hosta | Away Score | Pro odehrané zápasy |
| Skóre domácích | Home Score | Pro odehrané zápasy |

---

## 🎯 priors.xlsx — preseason projekce (volitelné)

Pokud existuje, model je použije jako Bayesovský prior místo ligového průměru. Řádek = jeden nadhazovač.

| Sloupec | Význam |
|---|---|
| pitcher_name (nebo Name) | Jméno (musí se shodovat s listem Starting pitcher) |
| SIERA_proj | Projektované SIERA (Steamer/ZiPS) |
| KBB_proj | Projektované K-BB% (procento) |
| xFIP_proj | Projektované xFIP |
| FIP_proj | Projektované FIP |

Pokud `priors.xlsx` v repu není (404), aplikace pokračuje bez priors — jako prior se použije ligový průměr (`LEAGUE_AVG_SP` pro startéry, `LEAGUE_AVG_RP` pro bullpen).

---

## 🔄 Workflow aktualizace dat

1. Export z FanGraphs — leaderboards → uložit jako XLSX se všemi 8 listy
2. Push do repa: `git add fangraphs.xlsx && git commit -m "Update <datum>" && git push`
3. Otevři aplikaci — při startu automaticky stáhne nejnovější verzi (cache busting přes `?t=Date.now()`)

> GitHub raw má ~5 minut Fastly CDN cache, takže pokud po pushi okamžitě otevřeš app, můžeš ještě vidět starou verzi. Cache busting v URL to obchází.

---

## 🗂️ Záložky aplikace

Model v23 obsahuje 5 záložek:

**Výběr zápasu** — manuální simulace jednoho matchupu. Výběr týmů, nadhazovačů (s auto-doplněním z MLB Stats API), weather, park factor. Výsledek: kalibrovaná pravděpodobnost + férový kurz + edge %.

**Denní predikce** — automatizovaná pipeline pro všechny zápasy vybraného dne. Model pro každý zápas sám stáhne startující nadhazovače z MLB Stats API, spáruje data z FanGraphs a spustí simulaci. Export XLSX celého přehledu na jeden klik.

**Moje tipy** — stejná pipeline jako Denní predikce, ale s výběrem tipu (domácí/hostující) a zadáním kurzu sázkovky. Snapshot stavu modelu se uloží do localStorage. Přeskočení zápasu (TBD nadhazovač, nezájem) tlačítkem ⏭️.

**Vyhodnocení modelu** — historická analýza tipů. Metriky: Hit Rate (po bucketech pravděpodobnosti), Brier Score, Log Loss, ROI/Yield, peněžní bilance (2u flat stake). Reliability diagram (Chart.js). Tabulka bilance po týmech s Δ kalibrace pro detekci systematického biasu modelu. Auto-doplnění výsledků z MLB Stats API.

**Přehled sázek** — agregovaný sázkovník plněný automaticky z Denní predikce (auto-přenos po každém uložení tipů).

---

## 🧬 Jak simulace funguje

Pipeline od vstupních dat až po finální win probability:

### 1. Bayesovský shrinkage (regrese k prior)
Pozorovaná statistika nadhazovače se „stahuje" k prior podle počtu odpitchovaných innings (IP). Čím méně IP, tím větší váha prior:

```
regressed_stat = (IP × observed + K × prior) / (IP + K)
```

K je shrinkage konstanta (K=50 pro SIERA/xFIP/FIP/BABIP, K=50 pro K-BB%). Prior je buď `priors.xlsx` (pokud existuje), nebo ligový průměr SP/RP.

Stejně se regresuje i wRC+ pálky podle PA s K=200 a prior 100 (ligový průměr).

### 2. v23: Rolling form korekce
Pokud je pitcher nalezen v listu `pitcher_last14days`:

```
SIERA_adj = SIERA_sezónní × 0,75 + FIP_14d × 0,25
```

Trigger penalizace krize/únavy: pokud `FIP_14d > SIERA_sezónní + 1,50`, přidá se k λ_start +0,25. Pokud pitcher v listu chybí, model použije jen sezónní SIERA.

### 3. True Talent Score
Z regresnutých (a případně rolling-form upravených) hodnot se počítá vážené skóre nadhazovače:

```
score = SIERA × 0,40 + kbbERA × 0,30 + xFIP × 0,20 + FIP × 0,10
```

Kde `kbbERA` je transformace K-BB% na ERA-ekvivalent:
```
kbbERA = 6,25 − 0,15 × K-BB%
```
Lineární mapování: K-BB% = 5 % → 5,50 · 15 % → 4,00 · 25 % → 2,50.

### 4. BABIP korekce
Pokud je BABIP nadhazovače extrémní, skóre se upraví:

- BABIP > 0,310 → score × 0,95 (pravděpodobně měl smůlu, regreduje směrem k lepšímu)
- BABIP < 0,270 → score × 1,05 (pravděpodobně měl štěstí, regreduje směrem k horšímu)

### 5. Lambda startéra (očekávané runs allowed)
```
λ_start = score × (4,50 / 4,20)
```
Normalizace na ligový průměr 4,50 R/G (`LEAGUE_AVG`) vůči referenčnímu skóre 4,20 (`LEAGUE_REF`).

### 6. Bullpen blend
Stejný výpočet pro bullpen, pak vážená kombinace:
```
λ_team = λ_start × 0,60 + λ_bull × 0,40
```
(60 % startér, 40 % bullpen — odráží typické rozdělení innings.)

### 7. Platoon split soupeřovy pálky
Vybere se wRC+ podle ruky startujícího nadhazovače:

- Startér je LH → použije se Batting vs LHP daného soupeře
- Startér je RH → použije se Batting vs RHP
- Ruka neznámá → fallback na Batting (overall)
- Tým není v datech → ligový průměr wRC+ = 100

Split se zamrazí na startéra a drží se pro celý zápas, i po výměně do bullpenu.

```
λ_team = λ_team × (wRC+ / 100)
```

### 8. Park Factor
```
λ_team = λ_team × (PF / 100)
```
PF > 100 → více runs (např. Coors Field Colorado), PF < 100 → méně runs (pitcher-friendly parky).

### 9. Home Field Advantage
Domácí tým dostane bonus na svojí ofenzivní lambdě:
```
λ_home = λ_home + 0,15
```

### 10. Negative Binomial Monte Carlo
Pro každý zápas se spustí simulace (výchozí 10 000, slider 1 000–100 000). V každé:
```
r = λ / (dispersion − 1)
p = 1 / dispersion
γ ~ Gamma(r, (1−p)/p)
runs ~ Poisson(γ)
```
NegBin místo čistého Poissonu protože reálná variance v MLB skóre je vyšší než průměr (dispersion ≈ 1,3).

Surová pravděpodobnost vítězství = počet simulací, kde tým A nasázel víc runs než tým B, děleno počtem simulací.

### 11. Logit shrinkage + cap
Surová P_raw se kalibruje proti přemrštěné jistotě:
```
logit = log(P / (1 − P))
P_shrunk = 1 / (1 + exp(−logit × 0,80))
```
Faktor k = 0,80 táhne extrémní pravděpodobnosti zpátky k 50 %. Pak se aplikuje kontextový cap podle situace (early-season, divizní zápas, TBD startér, postseason) — typicky 65–72 %.

### 12. Weather adjustment (volitelné)
Pokud je zapnuto, data přicházejí z OpenWeatherMap API (48h předpověď) nebo MLB Stats API live feed. Ruční přepsání je vždy možné.

Aditivní korekce Win Probability:

| Podmínka | Efekt |
|---|---|
| Vedro >90 °F | Favorit −3 % (slabší bullpen má větší volatilitu) |
| Chlad <50 °F | Domácí +3 % (aklimatizace) |
| Vítr ven >15 mph | Favorit −2 % (vyšší variance) |
| Vítr dovnitř >15 mph | Tým s lepším pitchingem +2 % |
| Vlhkost >70 % | Tým s vyšším wRC+ +1 % |

Korekce jsou aditivní k P_shrunk a normalizují se tak, aby P_A + P_B = 100 %.

### 13. Férový kurz
```
fair_odds = 1 / P_final
```
To je cílová hodnota proti sázkové kanceláři — pokud je nabízený kurz vyšší než fair_odds, sázka má kladnou očekávanou hodnotu.

---

## 🧪 Ligové průměry a konstanty

Hardcoded hodnoty v modelu:

| Konstanta | Hodnota | Význam |
|---|---|---|
| LEAGUE_AVG | 4,50 | Ligový průměr R/G |
| LEAGUE_REF | 4,20 | Referenční True Talent skóre |
| W_SIERA | 0,40 | Váha SIERA |
| W_KBB | 0,30 | Váha K-BB% (kbbERA) |
| W_XFIP | 0,20 | Váha xFIP |
| W_FIP | 0,10 | Váha FIP |
| W_STARTER | 0,60 | Váha startéra v týmové lambdě |
| W_BULLPEN | 0,40 | Váha bullpenu |
| BABIP_HIGH | 0,310 | Práh „smůla" |
| BABIP_LOW | 0,270 | Práh „štěstí" |
| BABIP_ADJ | 0,05 | Velikost BABIP korekce |
| HFA_BONUS | 0,15 | Home field advantage v λ |
| dispersion | 1,3 | NegBin nadrozptyl |
| kShrinkage | 0,80 | Logit shrinkage faktor |

Ligové priory (použité když `priors.xlsx` chybí):

| | SIERA | xFIP | FIP | ERA | BABIP | K-BB% |
|---|---|---|---|---|---|---|
| SP prior | 4,00 | 4,10 | 4,00 | 4,20 | 0,295 | 15,0 % |
| RP prior | 3,85 | 3,95 | 3,85 | 3,90 | 0,295 | 17,0 % |

Nastavení K hodnot, dispersion a kShrinkage je upravitelné v collapsible panelu „Pokročilá nastavení" přímo v aplikaci.

---

## ⚠️ Limity modelu

- **Bez injury reportu** — model neví, kdo je na IL ani kdo má omezené innings
- **Bez konkrétního lineup** — pracuje jen s týmovou agregací wRC+, neřeší den-co-den nasazené pálkaře
- **Bez ručních úprav úlevářů** — bullpen je celkový průměr, neumí rozlišit closer vs mop-up
- **Park factors stagnují** — Basic (5yr) je 5letý průměr, neodráží sezónní změny
- **Rolling form pouze FIP** — sheet `pitcher_last14days` obsahuje jen FIP, ne plný statline

Výstupy slouží jako podkladový analytický nástroj, ne jako záruka výsledku.

