# Fenna Energy Efficiency Audit v1.0 — eerste oplevering

**Status:** inventarisatie + methodiek. Geen implementatie (wacht op expliciete GO).
**Peilmoment:** zaterdag 21-11-2026, 13:28–13:30 CET (momentopname Home Assistant).
**Modus:** volledig READONLY. Er is niets gewijzigd aan Home Assistant, Fenna, HEMS v2, Solarbank, Zaptec of WinterClimate. Er zijn geen scripts, intents of diensten aangeroepen, alleen statusuitlezing (`GetLiveContext`, `GetDateTime`).
**HEMS v2-validatie:** loopt tot 23-11-2026 en is niet aangeraakt.

Iedere bewering in dit rapport heeft een label:

| Label | Betekenis |
|---|---|
| **[M]** Meting | Direct uitgelezen waarde uit Home Assistant op het peilmoment |
| **[C]** Configuratie | Ingestelde parameter (input_number/input_text), dus geen meting |
| **[A]** Afleiding | Rekenkundig uit [M]/[C] afgeleid, zonder extra aannames |
| **[S]** Modelschatting | Afhankelijk van een model of scenario; aannames staan erbij |
| **[?]** Aanname / te verifiëren | Niet aangetoond |

---

## 0. Managementsamenvatting

1. **Totale EEI: ONBEKEND.** Geen van de vijf deelscores is op dit moment met voldoende onderbouwing te berekenen. De beschikbare koppeling geeft alleen een momentopname van de aan Assist blootgestelde entiteiten. Er is geen toegang tot recorder-historie of long-term statistics, en gas, PV-vermogen, batterij-SOC/-energiestromen, Zaptec en tarieven zijn niet zichtbaar.
2. **De datakwaliteit is op dit moment de grootste bevinding.** De HomeWizard P1 is volledig `unavailable`, de HEMS P1-node is `stale` en de net-meting draait op terugvaltier 2 (Anker SOLIX P1). De dagtellers *HomeNet Dag Import/Export* staan op 0 kWh terwijl er 884 W wordt geïmporteerd. De PV-bron staat op `TRUSTED` terwijl de eigen evidence-diagnose `INSUFFICIENT_EVIDENCE` meldt.
3. **HEMS-sturing is feitelijk niet actief.** De schakelaars *energiepilot batterij* en *energiepilot Tiguan* staan uit [M], en klimaatautonomie is gepauzeerd met een handmatige override sinds 15-11 [M]. Een gerealiseerde HEMS-besparing is daarmee per definitie niet aantoonbaar. Dat past bij de lopende validatie.
4. **De batterijconfiguratie is consistent met 9,3 kWh nominaal**: 1,6 + 2,688 + 5,0 = 9,288 kWh [A]. De bruikbare capaciteit per slot staat echter op 0,0 kWh [C] en de round-trip-efficiëntie van 90 % is een ingestelde aanname [C], geen meting.
5. **Methodiek:** de EEI is zo ontworpen dat ze alleen normeert tegen (a) natuurkundige grenzen, (b) eigen aantoonbaar beste prestaties onder gelijke omstandigheden en (c) een transparante referentiestrategie die op de eigen meetdata wordt herspeeld. Er worden geen externe benchmarks gebruikt tenzij die met bron worden aangeleverd.
6. **Volgende stap:** een read-only data-export (zie §6). Daarmee zijn naar verwachting alle deelscores te berekenen behalve een deel van de gasscheiding.

---

## 1. Fase 1 — Inventarisatie

### 1.1 Toegangskanaal en beperkingen

| Kanaal | Beschikbaar | Opmerking |
|---|---|---|
| HA MCP `GetLiveContext` | Ja | Alleen huidige toestand van entiteiten die aan Assist zijn blootgesteld |
| HA recorder / history / statistics | **Nee** | Geen tool beschikbaar; zonder historie zijn graaddagen, basislast, RTE en trends onmogelijk |
| HEMS v2-rapportages / baselines / prognoses (bestanden) | **Nee** | Niet in deze repository (bevat alleen een Home Connect-fix) en niet via MCP zichtbaar |
| Scripts/intents (bijv. `FennaCarplayEnergie`) | Bewust niet gebruikt | Deze schrijven naar decision-/policy-logs en zijn dus niet READONLY |

Entiteiten blootstellen aan Assist zou een configuratiewijziging zijn en valt daarom buiten deze opdracht.

### 1.2 Inventaris per domein (peilmoment 21-11 13:28)

**Elektriciteit / net**

| Entiteit | Waarde | Label | Beoordeling |
|---|---|---|---|
| P1 Meter (HomeWizard): vermogen, import/export T1/T2, DSMR, meter-ID, spanning | alle `unavailable` | M | **Bron uitgevallen.** Past bij de gemelde P1-storing en de metervervanging |
| HEMS P1 Node | `stale` | M | Primaire HEMS-bron niet actueel |
| Anker SOLIX P1 Meter: grid usage / export | 884 W / 0 W, MQTT-tijd 13:28:13 | M | Actueel; nu de actieve bron |
| HEMS grid last switch diagnostic | `TIER_3_CALCULATED -> TIER_2_ANKER_P1; FAILBACK` | M | HEMS draaide eerder op berekende waarden (tier 3) |
| HEMS grid last-known value | 884 W | M | Consistent met Anker P1 |
| HomeNet Dag Import / Dag Export | 0 kWh / 0 kWh | M | **Inconsistent:** om 13:28 kan de dagimport bij doorlopende afname niet 0 zijn. Teller gereset of gekoppeld aan de uitgevallen P1 |
| Fenna woningverbruik baseline | 500 W | C | Parameter, geen gemeten baseline |

**Gas**

| Entiteit | Status |
|---|---|
| Geen enkele gasentiteit blootgesteld | **ONBEKEND.** Gas komt via DSMR doorgaans binnen op dezelfde P1-poort als de (uitgevallen) HomeWizard [?]. Dan heeft de gasreeks waarschijnlijk dezelfde gap als de stroomreeks. Of de Anker P1 gas doorgeeft, is te verifiëren. |

**PV (13 panelen, Enphase)**

| Entiteit | Waarde | Label | Beoordeling |
|---|---|---|---|
| Actief PV-vermogen | niet blootgesteld | — | **ONBEKEND** |
| HEMS PV evidence last diagnostic | `LIMITED_EVIDENCE \| INSUFFICIENT_EVIDENCE \| HOLD \| pv.enphase.site_ac` | M | Bewijs onvoldoende |
| HEMS PV stability diagnostic | `state=TRUSTED; confidence=70.0; fallback=452` | M | **Tegenstrijdig met de regel hierboven** |
| Successful validations / consecutive / overlap samples | 0 / 0 / 0 | M | Nooit gevalideerd tegen een referentie |
| Official Enphase app reference | 0 W | C | Referentiewaarde niet ingevuld |
| Source change count / fallback count | 672 / 452 | M | Veel bronwisselingen, dus een gefragmenteerde reeks |
| Valid measurement count | 30.022 | M | Er zijn wel veel metingen, alleen niet gevalideerd |
| Physical maximum | 10.000 W | C | Plausibiliteitsgrens; ruim boven het te verwachten piekvermogen van 13 panelen [?] |

**Thuisbatterij (Anker Solarbank 2 E1600 AC + uitbreidingen)**

| Entiteit | Waarde | Label |
|---|---|---|
| Centrale unit E1600 AC | 1,6 kWh | C |
| Slot 1 BP2700 | 2,688 kWh nominaal, bruikbaar **0,0** | C |
| Slot 2 BP5000 | 5,0 kWh nominaal, bruikbaar **0,0** | C |
| **Totaal nominaal** | **9,288 kWh** (≈ 9,3 kWh) | A |
| Bruikbare capaciteitsfactor | 1,0 | C |
| HEMS minimale reserve / Fenna minimale strategie-reserve | **10 % / 15 %** | C, twee verschillende reservewaarden |
| Maximale target-SOC | 90 % | C |
| Strategisch venster | 90 − 15 = 75 % → **6,97 kWh**; met 10 %-reserve: 80 % → 7,43 kWh | A |
| Round-trip-efficiëntie | 90 % | **C. Aanname, niet gemeten** |
| Max. netlaadvermogen / minimale economische marge / hysterese | 1200 W / 0,03 €/kWh / 0,01 €/kWh | C |
| Energiepilot batterij | `off` | M |
| Backup plan (Solarbank) | `on` | M |
| Temperaturen (unit / exp1 / exp2) | 23 / 12 / 15 °C | M. Expansiemodules in een koele berging; capaciteit en efficiëntie kunnen bij lage temperatuur afnemen [?] |
| SOC, laad-/ontlaadvermogen, kWh-tellers | **niet blootgesteld** | ONBEKEND |

**EV (Tiguan eHybrid, Zaptec)**

| Entiteit | Waarde | Label |
|---|---|---|
| Zaptec-entiteiten | niet blootgesteld; Fenna-context meldt `zaptec … degraded` | M (indirect) |
| VW-integratie (T-Roc, Tiguan) | alle `unavailable`; Fenna: `integration=degraded` | M |
| Energiepilot Tiguan | `off` | M |
| Laatste Zaptec-waarde in outcome log | `zap=on/0.0` (21-08-2026) | M (historisch, 1 punt) |

**Klimaat / comfort**

| Entiteit | Waarde | Label |
|---|---|---|
| Fenna WinterClimate binnentemperatuur | 18,71 °C | M |
| Woonkamer (SwitchBot Hub 2, west, Fenna Klimaat) | 19,0 / 19,1 / 19,0 °C, RV 59 % | M |
| Huiskamer (climate) | 19,2 °C, RV **68 %**, mode `heat`, **setpoint 9,4 °C** | M |
| Aqara W100 (ruimte onbekend) | 16,9 °C, RV 72 % | M |
| Klimaatautonomie gepauzeerd | `on` | M |
| Actuele beslissing | `manual_override`, decision-ID `fenna_climate_20261115T051513…` | M |
| Comfortband (CarPlay-config) | 19,0–25,0 °C; RV max 60 % | C |
| Buitentemperatuur | **geen bron blootgesteld** (geen weather-entiteit; autosensoren `unavailable`) | ONBEKEND |
| Nest | geen Nest-thermostaatentiteit herkenbaar; alleen Nest Hub-speakerconfig | ONBEKEND |

**Tarieven**

| Bron | Status |
|---|---|
| Dynamisch tarief, belastingen, terugleververgoeding | Niet blootgesteld; Fenna-context meldt `tariff=unknown`, `price=unknown` → **ONBEKEND** |

**Fenna/HEMS-gedrag en betrouwbaarheid**

| Entiteit | Waarde | Label |
|---|---|---|
| Fenna goal snapshot | `data_reliability 0.88 — Datakwaliteit is degraded` | M |
| Decision timing sample count 24 h | 0 | M |
| Orchestrator metrics | alles 0 | M |
| Backup: laatste poging / laatste succes | 21-11 04:58 / **20-11 05:04** | M. De poging van vandaag is niet geslaagd |
| HA Core / OS update | beschikbaar | M |

### 1.3 Betrouwbaarheidsoordeel per databron

| Bron | Oordeel | Bruikbaar voor | Reden |
|---|---|---|---|
| Anker SOLIX P1 (net, realtime) | **Matig–goed** (actueel) | Netvermogen na de failover | Actueel en consistent met HEMS last-known. Kalibratie tegen de nieuwe meter onbekend |
| HomeWizard P1 | **Onbruikbaar sinds de storing** | Historie vóór de storing [?] | Volledig `unavailable` |
| HomeNet dagtellers | **Onbetrouwbaar** | Niets | 0 kWh bij lopende import |
| PV Enphase (via HEMS) | **Onzeker** | Pas na validatie | Trust-state en evidence spreken elkaar tegen; 0 validaties; referentie 0 W |
| Batterij | **Onbekend** | Alleen configuratie | Geen SOC- of energiereeksen zichtbaar |
| Gas | **Onbekend** | — | Niet zichtbaar; waarschijnlijk gap door de P1-storing |
| EV / Zaptec | **Gedegradeerd** | — | Integraties `degraded`/`unavailable` |
| Binnentemperatuur (woonkamer) | **Goed** | Comfortanalyse | Drie onafhankelijke sensoren binnen 0,2 °C; WinterClimate-signaal 0,3–0,5 °C lager |
| Binnen-RV | **Matig** | Comfort met correctie | Huiskamer 68 % tegen woonkamer 59 %. Óf verschillende ruimtes, óf een sensorafwijking van circa 9 procentpunt |
| Buitentemperatuur | **Afwezig** | — | Nodig voor graaddagen; voorstel: KNMI-daggegevens van het dichtstbijzijnde station (publieke bron) |
| Tarieven | **Afwezig** | — | Nodig voor alle financiële scores |

### 1.4 Meetreeksbreuken waarmee de analyse rekening moet houden

1. **Metervervanging:** de cumulatieve tellers van de oude en nieuwe meter mogen niet op elkaar worden opgeteld. Methode: per meter-ID (DSMR-identificatie) aparte segmenten maken en alleen deltaʼs binnen een segment gebruiken.
2. **Tierwisselingen grid (HomeWizard → berekend → Anker P1):** gebruik per interval precies één bron, met de actieve tier als sleutel. Intervallen van tier 3 (berekend) worden gemarkeerd en tellen niet als meting.
3. **PV-bronwisselingen (672×) en fallbacks (452×):** neem alleen intervallen met een gevalideerde bron mee. De energiebalans `PV + import − export ± batterij − verbruik` moet binnen de tolerantie van 100 W [C] sluiten, anders wordt het interval uitgesloten.
4. **Dubbeltellingen:** de Anker P1 Meter en HomeWizard P1 meten hetzelfde aansluitpunt en mogen nooit samen worden opgeteld.
5. **Gasgap:** dagen zonder geldige gasdelta worden niet geïnterpoleerd. Ze tellen als ontbrekend in de datakwaliteitsscore.

---

## 2. Fase 2 — Energieanalyse: wat nu berekenbaar is

### 2.1 Gas en verwarming

| Analyse | Resultaat | Reden |
|---|---|---|
| m³ per graaddag | **ONBEKEND** | Geen gas- en geen buitentemperatuurreeks |
| Relatie binnen/buiten/comfort/gas | **ONBEKEND** | Idem |
| Ruimteverwarming tegenover tapwater/koken | **ONBEKEND** | Idem |
| WinterClimate-strategievergelijking | **ONBEKEND** | Bovendien staat klimaatautonomie sinds 15-11 gepauzeerd met een override, dus die periode is niet vergelijkbaar |
| Resterende besparingsruimte | **ONBEKEND** | — |

Observatie [M], niet interpreteerbaar zonder historie: de huiskamerthermostaat staat in `heat` met setpoint 9,4 °C bij 19,2 °C binnen. Dat wijst erop dat de ketel op dit moment niet vraagt. Of dit een bewuste WinterClimate-/override-strategie, een eco-stand of een fout is, valt niet vast te stellen. [?]

### 2.2 Elektriciteit

| Analyse | Resultaat |
|---|---|
| Basislast, dag/nacht | **ONBEKEND.** Eén momentopname van 884 W netimport (za 13:28, zon boven de horizon, PV onbekend) zegt niets over de basislast |
| Scheiding huishouden / EV / batterijverlies | **ONBEKEND** |
| Zelfconsumptie / zelfvoorziening | **ONBEKEND** |
| Gemeten RTE | **ONBEKEND.** Alleen de ingestelde aanname van 90 % |
| Onverklaarde pieken | **ONBEKEND** |

### 2.3 Batterij: wat de configuratie wel vastlegt (economische grenzen) [A]/[S]

De bruto waarde van één volledige cyclus is:

```
W_cyclus = E_venster × p_ontlaad − (E_venster / η) × p_laad − E_venster × c_slijtage
```

Een cyclus is rendabel als `p_ontlaad − p_laad/η > c_slijtage + marge`.
Met η = 0,90 [C] moet de ontlaadprijs dus minstens 11,1 % boven de laadprijs liggen. Daarbovenop komen slijtage en de ingestelde marge van 0,03 €/kWh.

**Scenario-illustratie [S].** Dit zijn rekenvoorbeelden, geen gemeten tarieven. E_venster = 6,97 kWh, η = 0,90, slijtage voorlopig 0:

| Laadprijs | Ontlaadprijs | W per cyclus | Bovengrens bij 1 cyclus/dag (365) |
|---|---|---|---|
| 0,20 €/kWh | 0,25 €/kWh | 6,97 × (0,25 − 0,222) = **€0,19** | €71/jaar |
| 0,20 €/kWh | 0,30 €/kWh | 6,97 × (0,30 − 0,222) = **€0,54** | €198/jaar |
| 0,15 €/kWh | 0,35 €/kWh | 6,97 × (0,35 − 0,167) = **€1,28** | €467/jaar |

Kanttekeningen: dit zijn bovengrenzen. Niet elke dag heeft zoʼn spreiding, en het netlaadvermogen van 1200 W [C] betekent dat een volledig venster minstens ~6,5 uur netladen kost (6,97 / 0,9 / 1,2). Slijtage verlaagt het resultaat verder. **Conclusie [S]:** bij een dagelijkse spreiding van ≲ 0,05 €/kWh is netgestuurde arbitrage met deze batterij economisch nauwelijks zinvol. De waarde zit dan vrijwel geheel in PV-zelfconsumptie. Dit wordt hard zodra de echte uurprijzen zijn aangeleverd.

### 2.4 HEMS

| Analyse | Resultaat |
|---|---|
| Werkelijk gedrag tegenover referentiestrategie | **ONBEKEND** (geen historie) |
| Gerealiseerde besparing | **≈ 0 per definitie zolang de pilots uit staan** [A], *mits* de batterij niet via een eigen Anker-modus tariefgestuurd draait [?] |
| Hypothetische besparing | **ONBEKEND.** Zie de bovengrenzen in §2.3 |
| Toegevoegde waarde dynamische sturing | **ONBEKEND** |
| Handmatige interventies | 1 actieve override (klimaat, sinds 15-11) [M]; counters "overrides vandaag" = 0 [M] |

---

## 3. Fase 3 — Energy Efficiency Index (EEI): methodiek

### 3.1 Uitgangspunten

1. **Geen verzonnen benchmarks.** Elke referentie is één van:
   - **R-fys:** een natuurkundige of technische grens die uit de eigen data volgt, bijv. de maximaal haalbare zelfconsumptie bij de gemeten PV- en verbruiksprofielen;
   - **R-eigen:** de eigen best aangetoonde prestatie onder vergelijkbare omstandigheden, bijv. de beste 10 % van de weken bij een vergelijkbaar comfortniveau;
   - **R-strat:** een transparante referentiestrategie die deterministisch op de eigen meetdata wordt herspeeld;
   - **R-ext:** alleen als de gebruiker een externe bron aanlevert (bijv. energielabel, fabrieksdatasheet, CBS-cijfer), met bronvermelding.
2. **Score = gerealiseerd / haalbaar**, begrensd op [0, 1], maal het aantal punten. "Haalbaar" is R-fys of R-eigen, nooit een wensbeeld.
3. **Reproduceerbaar:** de EEI wordt per kalendermaand berekend over een vaste periode, met een vastgelegde dataversie (export-hash) en de parameterwaarden van dat moment.
4. **ONBEKEND-regel:** een deelscore wordt alleen berekend als de datadekking ≥ 90 % van de benodigde intervallen is én de onzekerheidsmarge ≤ 25 % van de maximale punten van die deelscore is. Anders wordt ze ONBEKEND. De totale EEI wordt alleen gerapporteerd als alle vijf deelscores bekend zijn. Anders volgt alleen "x van 100 punten berekenbaar". De drempels van 90 % en 25 % zijn een methodische keuze ter goedkeuring, geen benchmark.
5. **Drie afzonderlijke rapportages** naast de EEI: *energie-efficiëntie* (kWh/m³), *financiële efficiëntie* (€) en *datakwaliteit*.

### 3.2 Deelscore 1 — Gas en verwarming (30 punten)

**Model (stooklijn per dag):**
```
G_dag = a + b · HDD(T_b),   HDD(T_b) = max(0, T_b − T_buiten,dagmiddel)
```
- `a` = niet-temperatuurafhankelijk gas (tapwater/koken) [S], `b` = m³ per graaddag.
- **Basistemperatuur T_b wordt uit de data bepaald**, niet op 18 °C vastgezet. Er wordt gescand over 10–20 °C in stappen van 0,5 °C en gekozen voor de T_b met de laagste residuele fout (change-point-regressie). Ter controle wordt ook de binnen-gecorrigeerde variant `HDD = max(0, T_binnen,gemeten − ΔT_winst − T_buiten)` gerapporteerd, zodat een hogere comforttemperatuur niet als "inefficiënt" wordt bestraft maar apart zichtbaar is.
- Scheiding ruimteverwarming/overig: `a` uit zomerdagen (HDD = 0) en uit de intercept. Bij onvoldoende zomerdata wordt de scheiding ONBEKEND.

**Score:**
```
S1 = 30 × [ 0,7 × clamp(b_ref,eigen / b_actueel) + 0,3 × clamp(a_ref,eigen / a_actueel) ]
```
- `b_ref,eigen` = laagste rollende 4-weekse `b` uit de beschikbare historie, *alleen* in perioden met gelijk comfort (binnen ≥ 19,0 °C tijdens bewoonde uren; ondergrens uit de eigen config).
- WinterClimate-strategieën worden vergeleken via `b` per strategieperiode, met gelijkwaardige buitentemperatuurklassen (bins van 2 °C) en windkracht als covariaat als KNMI-data beschikbaar is.
- De gewichten 0,7/0,3 zijn voorlopig. Ze worden vervangen door het gemeten aandeel ruimteverwarming in het totale gasverbruik (zie §3.8).

**Databronnen:** gasmeter (DSMR, per meter-ID gesegmenteerd), KNMI-daggemiddelde temperatuur (en eventueel wind/zon), binnentemperatuursensoren, WinterClimate-strategielog.
**Onzekerheid:** standaardfout van `b` uit de regressie (95 %-BI) doorgerekend naar S1. Plus de onzekerheid door de gasgap rond de metervervanging.
**Besparingsruimte:** `(b_actueel − b_ref,eigen) × HDD_normjaar` in m³. HDD_normjaar is het gemiddelde uit de KNMI-historie van hetzelfde station.

### 3.3 Deelscore 2 — Elektrisch basisverbruik (20 punten)

**Huishoudelijk verbruik (afgeleid per 15 minuten):**
```
L_huis = Import − Export + PV − (Batt_laad − Batt_ontlaad) − EV_laad
```
Een interval telt alleen mee als de balans sluit binnen de geconfigureerde 100 W-tolerantie.

| Component | Formule | Punten |
|---|---|---|
| Basislast | `P_base` = P10 van L_huis tussen 01:00–05:00 per nacht; maandmediaan. Referentie R-eigen: laagste maandwaarde van de afgelopen 12 maanden. `10 × clamp(P_base,ref / P_base,actueel)` | 10 |
| Onverklaarde pieken | Fractie van de kWh boven `P_base + 3·σ` die niet aan een bekend apparaat (Siemens-apparaten, airco, EV, batterij) is toe te wijzen. `5 × (1 − f_onverklaard)` | 5 |
| Trend | Jaar-op-jaar dag- en nachtverbruik, graaddag- en bewoningsneutraal. `5 × clamp(E_vorigjaar / E_ditjaar)` | 5 |

**Let op:** de geconfigureerde baseline van 500 W [C] wordt *niet* als referentie gebruikt. Die wordt juist getoetst tegen de gemeten `P_base`.
**Onzekerheid:** bootstrap over nachten (95 %-BI van de mediaan).

### 3.4 Deelscore 3 — PV en batterij (20 punten)

| Component | Formule | Referentie | Punten |
|---|---|---|---|
| Zelfconsumptie | `SC = (PV − Export)/PV` tegenover `SC_max` = simulatie met ideale zelfconsumptiebatterij (venster 6,97 kWh, gemeten η) op hetzelfde 15-min-profiel | R-fys | 8 |
| Zelfvoorziening | `SS = (L_totaal − Import)/L_totaal` tegenover `SS_max` uit dezelfde simulatie | R-fys | 4 |
| Round-trip-efficiëntie | `η_RTE = ΣE_ontlaad,AC / ΣE_laad,AC` over cycli met SOC_begin ≈ SOC_eind (±2 %). `4 × clamp(η_gemeten / η_datasheet)` | R-ext (Anker-datasheet, aan te leveren) | 4 |
| Stand-by-verlies | Energieverlies per dag bij een stilstaande batterij (geen laden/ontladen, ΔSOC). `4 × clamp(1 − verlies/verlies_max)`; verlies_max = R-eigen slechtste maand | R-eigen | 4 |

Inclusief temperatuurcorrectie: RTE en bruikbare capaciteit worden apart gerapporteerd voor modules < 15 °C (BP2700/BP5000 staan nu op 12–15 °C).

### 3.5 Deelscore 4 — HEMS en tariefoptimalisatie (20 punten)

Drie kostenreeksen worden op **identieke gemeten** verbruiks- en PV-profielen en **werkelijke** tarieven berekend:

| Reeks | Definitie |
|---|---|
| `K_ref` | **Referentiestrategie R-strat (transparant):** batterij laadt alleen uit PV-overschot en ontlaadt bij netafname tot de 15 %-reserve; geen netladen; EV laadt direct bij aansluiten met vol vermogen |
| `K_werkelijk` | Gemeten importkosten − exportopbrengst + vaste kosten per tijdvak (met energiebelasting, btw en terugleververgoeding volgens contract) |
| `K_optimaal` | Perfect-foresight-optimalisatie (LP) met dezelfde fysieke grenzen (1200 W netladen, 90 % max SOC, 15 % reserve, gemeten η) plus slijtagekosten per kWh doorzet |

```
Gerealiseerd (gemeten)   V_real = K_ref − K_werkelijk
Hypothetisch potentieel  V_pot  = K_ref − K_optimaal
Resterend                V_rest = K_werkelijk − K_optimaal
S4 = 20 × clamp(V_real / V_pot) × (1 − p_interventie)
```
- `p_interventie` = aandeel beslissingen dat handmatig werd overruled of teruggedraaid (uit het Fenna decision/override-log). Een strategie die alleen werkt doordat de bewoner ingrijpt, scoort lager.
- **Economische drempel:** als `V_pot` kleiner is dan de eigen onzekerheidsmarge (2σ) of kleiner dan `c_slijtage × extra doorzet`, wordt S4 niet gescoord maar gerapporteerd als **"optimalisatie economisch niet zinvol"**. Een kleine potentie levert dan geen strafpunten op.
- Slijtagekosten: `c_slijtage = aanschafprijs / (garantiecycli × E_venster × eind-SoH-factor)`. Gegevens uit de aankoopfactuur en garantievoorwaarden, aan te leveren.
- De validatieperiode van HEMS v2 tot 23-11 wordt apart gerapporteerd en niet vermengd met de periode daarna.

### 3.6 Deelscore 5 — Comfort en betrouwbaarheid (10 punten)

| Component | Formule | Punten |
|---|---|---|
| Thermisch comfort | Fractie bewoonde uren met T_woonkamer ∈ [19,0; 25,0] °C (eigen config) | 3 |
| Vochtcomfort | Fractie uren met RV ≤ 60 % (eigen knowledge base `hum_max=60`) | 2 |
| Technische betrouwbaarheid | `1 − (uren met stale/unavailable bron of fallback-tier)/totaal`, gewogen over grid, PV, batterij en klimaat | 3 |
| Interventielast | `1 − min(1, handmatige overrides per week / max_waargenomen)` | 2 |

"Bewoonde uren" volgen uit aanwezigheid of beweging. Zonder betrouwbaar aanwezigheidssignaal: 07:00–23:00 [A, aanname vast te leggen].

### 3.7 Afzonderlijke rapportages

| Rapport | Inhoud |
|---|---|
| **Energie-efficiëntie** | m³/graaddag, `P_base` (W), kWh/jaar per stroom, SC/SS %, η_RTE |
| **Financiële efficiëntie** | `K_werkelijk`, `V_real`, `V_pot`, `V_rest` in €/maand en €/jaar; break-even-spreiding batterij; kosten per gereden EV-kWh |
| **Datakwaliteit (DQ, 0–100 %)** | `DQ = dekking × consistentie × actualiteit` per bron. Dekking = geldige intervallen/verwachte intervallen; consistentie = fractie intervallen met sluitende balans; actualiteit = 1 − fractie stale |

### 3.8 Wegingen

De voorgestelde wegingen (30/20/20/20/10) worden voorlopig aangehouden. Er is nog geen data om ze aantoonbaar te weerleggen. **Voorgestelde toetsregel voor herweging** (na de eerste volledige berekening):
- De gewichten van deelscores 1–4 worden evenredig gemaakt aan het aandeel van elk domein in de **beïnvloedbare jaarlijkse energiekosten** (€), met 10 punten vast voor comfort en betrouwbaarheid.
- Concreet signaal dat herweging waarschijnlijk nodig is [S]: als `V_pot` voor HEMS kleiner blijkt dan ~10 % van de totale beïnvloedbare kosten (bijv. bij een geringe prijsspreiding, zie §2.3), is 20 punten voor HEMS onevenredig en verschuift gewicht naar gas.

---

## 4. Fase 4 — Eerste resultaten

### 4.1 Scoretabel

| Deelscore | Max | Score | Datadekking | Betrouwbaarheid | Reden |
|---|---|---|---|---|---|
| 1 Gas en verwarming | 30 | **ONBEKEND** | 0 % | — | Geen gas- en buitentemperatuurreeks |
| 2 Elektrisch basisverbruik | 20 | **ONBEKEND** | 0 % historie | — | Geen historie; P1-breuk |
| 3 PV en batterij | 20 | **ONBEKEND** | 0 % | — | PV niet gevalideerd; batterij-SOC niet zichtbaar |
| 4 HEMS en tarief | 20 | **ONBEKEND** | 0 % | — | Geen tarieven; pilots uit |
| 5 Comfort en betrouwbaarheid | 10 | **ONBEKEND** | momentopname | — | Comfort op het peilmoment binnen de band (19,0–19,2 °C, RV 59 %), maar één punt is geen score |
| **EEI totaal** | 100 | **ONBEKEND** | 0 van 100 punten berekenbaar | | |

**Datakwaliteit (momentopname, kwalitatief):**

| Bron | Actueel bruikbaar |
|---|---|
| Net (Anker P1) | ✅ |
| Net (HomeWizard P1) | ❌ |
| Dagtellers | ❌ |
| PV | ⚠️ niet gevalideerd |
| Batterij | ❌ niet zichtbaar |
| Gas | ❌ |
| EV | ⚠️ degraded |
| Tarief | ❌ |
| Binnenklimaat | ✅ |
| Buitentemperatuur | ❌ |

Er kan geen numerieke DQ-score worden gegeven zonder historie.

### 4.2 Historische trends en vergelijkingen

**ONBEKEND.** Er is geen historie beschikbaar via het huidige kanaal.

### 4.3 Top 5 bevindingen

Geen van onderstaande bevindingen is een *aangetoonde energie-inefficiëntie in kWh, m³ of €*. Daarvoor ontbreekt historie. Het zijn aantoonbare **structurele zwakheden** die de efficiëntie of de meetbaarheid ervan raken.

| # | Bevinding | Bewijs | Impact | Betrouwbaarheid |
|---|---|---|---|---|
| 1 | **Primaire meetketen gebroken:** HomeWizard P1 `unavailable`, HEMS P1 `stale`, dagtellers 0 kWh bij lopende import | [M] | HEMS en rapportages rekenen op fallback-data; dag- en maandrapportages over deze periode zijn onbetrouwbaar | Hoog (feit) / impact onbekend |
| 2 | **PV-trust zonder validatiebewijs:** `TRUSTED` tegenover `INSUFFICIENT_EVIDENCE`, 0 validaties, referentie 0 W, 672 bronwisselingen | [M]/[C] | Zelfconsumptie, prognoses en batterijbeslissingen kunnen op een ongevalideerde PV-reeks steunen | Hoog (feit) / impact onbekend |
| 3 | **Batterijparameters onvolledig of inconsistent:** bruikbare capaciteit 0,0 kWh per slot, twee reservewaarden (10 % en 15 %), RTE 90 % aangenomen | [C] | Planner rekent mogelijk met het verkeerde venster (6,97 tegenover 7,43 kWh, een verschil van 0,46 kWh per cyclus) en een niet-gemeten efficiëntie | Hoog (feit) / impact klein tot middel [S] |
| 4 | **Tarief niet beschikbaar in de beslislaag** (`tariff=unknown`, `price=unknown`) | [M] | Dynamische tariefsturing kan nu geen waarde leveren, ook niet bij ingeschakelde pilots | Hoog (feit) / impact begrensd door §2.3 |
| 5 | **Klimaatautonomie langdurig overruled** (handmatige override sinds 15-11, autonomie gepauzeerd) en thermostaat-setpoint 9,4 °C | [M] | WinterClimate-strategieën zijn over deze periode niet vergelijkbaar; de bijdrage van automatisering aan de gasbesparing is niet te isoleren | Hoog (feit) / interpretatie [?] |

Nevenbevindingen ter verificatie [?]:
- *Voordeur Door sensor* = `on` op het peilmoment. Als dat "open" betekent tijdens het stookseizoen, gaat er warmte verloren. Mogelijk sensorlogica of een momentopname.
- RV-afwijking huiskamer (68 %) tegenover woonkamer (59 %).
- Automatische back-up van 21-11 niet geslaagd. Dat raakt de reproduceerbaarheid van de recorder-historie.

### 4.4 Realistisch resterend besparingspotentieel

| Domein | kWh | m³ | € | Status |
|---|---|---|---|---|
| Gas/verwarming | — | ONBEKEND | ONBEKEND | Wacht op gas- en KNMI-data |
| Basislast | ONBEKEND | — | ONBEKEND | Wacht op historie |
| PV-zelfconsumptie | ONBEKEND | — | ONBEKEND | Wacht op PV-validatie |
| Batterij-arbitrage (HEMS) | — | — | **Bovengrens [S]: €70–470/jaar** bij 0,05–0,20 €/kWh dagelijkse spreiding, vóór slijtage en zonder rekening met dagen zonder spreiding | Scenario, geen meting |
| EV-slim laden | ONBEKEND | — | ONBEKEND | Wacht op Zaptec-data |

**Wanneer is verdere optimalisatie economisch nauwelijks zinvol?** Er zijn drie vooraf vastgelegde stopcriteria:
1. `V_rest < 2σ(V)`: het resterende verschil met het optimum valt binnen de meetonzekerheid.
2. `V_rest < c_slijtage × extra doorzet` die nodig is om het te realiseren.
3. Een prijsspreiding onder het break-even-punt `p_laad × (1/η − 1) + c_slijtage + marge`. Bij η = 0,9 en p_laad = 0,20 €/kWh is dat ≥ 0,022 €/kWh + slijtage + 0,03 €/kWh marge [A].

---

## 5. Onderscheid meting, model en aanname: overzicht

| Categorie | Voorbeelden in dit rapport |
|---|---|
| **Meting** | 884 W netimport (Anker P1), binnentemperaturen 18,7–19,2 °C, RV 59/68/72 %, P1 `unavailable`, PV-diagnoses, pilotschakelaars uit, override sinds 15-11 |
| **Configuratie (geen meting)** | 9,288 kWh nominaal, RTE 90 %, reserves 10/15 %, max SOC 90 %, baseline 500 W, comfortband 19–25 °C, RV max 60 % |
| **Afleiding** | 6,97/7,43 kWh venster, break-even-formules, ≥ 6,5 uur netladen voor een vol venster |
| **Modelschatting** | Arbitrage-bovengrenzen in §2.3 en §4.4 |
| **Aanname / te verifiëren** | Gas via dezelfde P1 en dus dezelfde gap, de betekenis van setpoint 9,4 °C, de deurstatus, koude-effect op de modules, het piekvermogen van de 13 panelen |

---

## 6. Datagaten en benodigde read-only aanlevering

Zonder wijzigingen aan de systemen. Exporttaken voert de gebruiker zelf uit, of ze gebeuren pas na een GO.

| # | Gegevens | Periode | Mogelijke bron | Nodig voor |
|---|---|---|---|---|
| D1 | Elektriciteit import/export (15 min of uur), per meter-ID | ≥ 12 mnd, incl. de storing | HA Energy-statistieken (CSV-export), netbeheerder-portaal, HomeWizard-app | S2, S3, S4 |
| D2 | Gas (uur of dag) | ≥ 12 mnd, liefst ≥ 2 stookseizoenen | HA-statistieken, netbeheerder, jaarafrekening | S1 |
| D3 | PV-productie (15 min) + Enphase-app-totalen ter validatie | ≥ 12 mnd | Enphase Enlighten-export | S3, PV-validatie |
| D4 | Batterij SOC, laad-/ontlaad-kWh (AC), moduletemperaturen | ≥ 3 mnd | Anker-app-export, HA-statistieken | S3, RTE |
| D5 | Zaptec-sessies (kWh, start/eind) | ≥ 12 mnd | Zaptec-portaal-export | S2-scheiding, S4 |
| D6 | Uurprijzen + contract (opslagen, energiebelasting, btw, terugleververgoeding/-kosten) | idem | Leverancier / prijsbron die HEMS gebruikt | S4, financieel |
| D7 | Buitentemperatuur, wind, zonnestraling per dag | idem | KNMI-daggegevens van het dichtstbijzijnde station (station of postcodegebied opgeven) | S1 |
| D8 | WinterClimate-strategielog + Fenna override-/decisionlog | idem | HA-logboek / Fenna-logs | S1, S4, S5 |
| D9 | HEMS v2-rapportages, baselines, prognoses | alle | Locatie opgeven (bestanden of repo) | Hergebruik, R-strat |
| D10 | Batterij-aanschafprijs, garantiecycli, datasheet-RTE | — | Factuur / Anker-datasheet | c_slijtage, S3 |
| D11 | Datum en tijd van de metervervanging + oude en nieuwe meter-ID | — | Netbeheerder / eigen notitie | Segmentatie |

**Verwachte berekenbaarheid na aanlevering:** D1 + D3 + D4 + D6 → S3 en S4. Daarbij komen D2 + D7 → S1 en D8 → S5. Alleen bij minder dan één stookseizoen aan gasdata blijft de scheiding ruimteverwarming/overig ONBEKEND.

---

## 7. Voorstel vervolg (na expliciete GO, en na 23-11)

1. Data-aanlevering D1–D11, read-only.
2. Opschoning en segmentatie: meter-ID-segmenten, tierkeuze per interval, balanscontrole.
3. Eerste volledige EEI-berekening plus de drie afzonderlijke rapportages, met BI per conclusie.
4. Toetsing van de wegingen volgens §3.8.
5. Pas daarna een eventueel voorstel voor wijzigingen in HEMS of Fenna, met de verwachte waarde boven de stopcriteria van §4.4.
