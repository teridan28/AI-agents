# cyanospectra – zhrnutie a stav projektu (SK)

## Čo to je
Python knižnica na analýzu a dekonvolúciu *in vivo* absorpčných spektier siníc. Z jedného
spektra celých buniek vyrobí príspevky jednotlivých pigmentov (Chl a, karotenoidy, PC, APC, PE),
fyziologické pomery (PC/Chl, APC/PC, Car/Chl, PBS/Chl), neistoty a interpretáciu stavu kultúry.

## Rýchly štart
```bash
pip install -e .
pytest                      # 31 testov
python examples/demo.py     # kompletná ukážka na syntetických dátach
```
```python
import cyanospectra as cs
spectra = cs.parse_table("spektra.xlsx")        # alebo cs.parse_pasted(text_zo_schranky)
res = cs.analyze({"spectra": spectra})
res["ratios"]    # {vzorka: {"PC/Chl": ..., "APC/PC": ..., "Car/Chl": ..., "PBS/Chl": ...}}
res["qc"]        # nálezy kontroly kvality
```

## Čo je v balíku
| Krok z projektu | Modul | Stav |
|---|---|---|
| Generátor syntetických spektier | `synthetic.py` | hotové, otestované |
| Parsing (xlsx, text, vložené z Excelu) | `parsing.py` | hotové, otestované |
| Kontrola kvality (QC) | `qc.py` | hotové, otestované |
| Preprocessing (rozptyl, vyhladenie, normalizácia, replikáty) | `preprocessing.py` | hotové, otestované |
| Referenčná knižnica pigmentov | `reference.py`, `data/` | mechanizmus hotový, **obsah je zatiaľ placeholder** |
| Dekonvolúcia (Gauss + NNLS), AIC/BIC, neistoty | `deconvolution.py`, `uncertainty.py` | hotové, otestované |
| Benchmark voči syntetickej pravde | `benchmark.py` | hotové, len na syntetických dátach |
| Odvodené parametre a pomery | `derived.py` | hotové |
| Vizualizácia | `viz.py` | hotové |
| Klasifikácia (pravidlá, PLS-DA, logreg, CNN) | `classification.py` | čiastočne (pozri nižšie) |
| Jednotný vstup `analyze()`, JSON, Excel export | `pipeline.py` | hotové |
| Voliteľný AI agent (Claude tool use) | `agent.py` | hotové, nie je overený na živom API |

## Hlavné výsledky
**Benchmark (syntetické dáta, medián relatívnej chyby pomerov):**
- NNLS je robustnejší voči šumu, prekryvu pásiem aj chybe korekcie rozptylu → predvolená metóda.
- Najväčšie riziko je odhad rozptylu iba z NIR okna 710–750 nm (Gauss 47 % chyba, NNLS 6 %).
  Preto sa rozptyl predvolene fituje **spoločne s pásmami** (power-law pozadie): obe metódy ~3,7 %.
- Pri väčšom prekryve pásiem sa obe metódy zhoršujú (Gauss 21 %, NNLS 14 % pri rozšírení ×1,3).

**Klasifikácia stavu kultúry (syntetické dáta, krížová validácia podľa kultúr):**
| model | CV presnosť |
|---|---|
| surové spektrum, PLS-DA | 0,81 ± 0,17 |
| surové spektrum, logistická regresia | 0,96 ± 0,08 |
| príznaky z dekonvolúcie, PLS-DA | 1,00 |
| príznaky z dekonvolúcie, logistická regresia | 1,00 |

⚠️ Toto **ešte nie je odpoveď** na vedeckú otázku projektu (pomáha mechanistická extrakcia príznakov
oproti sieti na surovom spektre?). Triedy som v generátore definoval práve cez pomery pigmentov,
takže príznaky sú zvýhodnené už z konštrukcie. Rozhodujúci test treba spraviť na reálnych
dátach alebo na syntetických dátach, kde sa triedy líšia aj iným spôsobom než len pomermi.

## Obmedzenia (treba ich povedať otvorene)
1. Referenčné spektrá sú parametrický placeholder, nie merania; presnosť NNLS na reálnych dátach od nich závisí.
2. Všetky čísla z benchmarku platia pre syntetické dáta; validácia na reálnych kultúrach chýba.
3. Neistota z bootstrapu zahŕňa iba šum, nie systematickú chybu (nesedí referencia, zlý rozptyl).
4. Prahy pravidlového interpretera a definície tried LL/HL/N_lim sú ilustračné, nekalibrované.
5. Trénovanie 1D CNN (PyTorch) nebolo spustené; otestovaná je len NumPy inferencia.
6. AI agent je otestovaný proti falošnému klientovi, nie proti živému API.
7. Inštalácia nad rámec `pip install -e .`, licencia a CI sú „TBD“; webová vrstva je mimo rozsahu.

## Čo môžete povedať, že je z projektu hotové
> „Je hotová celá analytická knižnica podľa roadmapy: generátor syntetických spektier, parsing,
> QC, preprocessing, dve metódy dekonvolúcie (Gauss a NNLS) s neistotami, odvodené parametre,
> vizualizácia, benchmark voči známej pravde, pravidlový interpreter a klasifikačné modely a jednotný
> vstup `analyze()` s JSON a Excel výstupom, vrátane 31 automatických testov. Správnosť dekonvolúcie
> je overená na syntetických dátach s poznanou pravdou. Zatiaľ nie je hotová validácia na reálnych
> spektrách, doplnenie nameraných referenčných spektier, kalibrácia pravidiel a trénovanie CNN.“

## Čo povedať, že ešte hotové nie je
- reálne namerané referenčné spektrá pigmentov a validácia na reálnych kultúrach,
- vedecké vyhodnotenie „surové spektrum vs. mechanistické príznaky“ na nezávislých dátach,
- natrénovaná a otestovaná 1D CNN,
- licencia, inštalačná dokumentácia, CI, príkladový dataset,
- integrácia do platformy cyano.tools (mimo rozsahu repozitára).
