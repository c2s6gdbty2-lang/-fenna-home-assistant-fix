# Audit HEMS-energiesensoren en statistische eenheden — 25-11-2026

**Aanleiding:** HA-reparatiemelding 25-11 13:32: eenheid van `sensor.hems_pv_envoy_vandaag` gewijzigd naar **MWh**; de historische statistieken hebben **geen** eenheid.
**Modus:** uitsluitend READ-ONLY. Er is niets gewijzigd: geen configuratie, statistieken, herstarts, reloads, aansturing of wijziging aan `advice_only`.

**Bronnen**
- Repository `fenna-home-assistant` op `main` @ `0bcf7a3` (= merge van PR #55), alleen gelezen.
- Broncode van Home Assistant Core 2026.2.3: `utility_meter/sensor.py`, `enphase_envoy/sensor.py`, `sensor/__init__.py`.
- Live status via de Assist-MCP.

**Beperking:** `sensor.hems_pv_envoy_vandaag`, `sensor.hems_adapter_pv_today`, `sensor.hems_pv_today` en de Envoy-sensoren zijn **niet** aan de MCP blootgesteld. Ik heb geen toegang tot recorder-historie, statistiek-metadata, `.storage` of het energiedashboard. Live waarden zijn daarom niet geverifieerd; zie §6 voor de read-only controlestappen.

Labels: **[F]** feit (config of code) · **[C]** afgeleid uit broncode · **[H]** hypothese · **[?]** onbekend.

---

## 1. Sensorcontrole

### Keten (uit config en code)

```
sensor.envoy_482335000893_lifetime_energy_production   (Enphase Envoy, core-integratie)
   native Wh → weergegeven/state in MWh, state_class total_increasing      [C]
        │  UI utility_meter-helper (PR #55, 24-11 ~20:22), daily, periodically_resetting: false
        ▼
sensor.hems_pv_envoy_vandaag      eenheid = die van de bron (MWh), nadat de bron één keer is veranderd   [C]
        │  template, rekent MWh ×1000 / Wh ÷1000 / kWh ×1 om
        ▼
sensor.hems_adapter_pv_today      kWh, total_increasing, vast in YAML          [F]
        ▼
sensor.hems_pv_today              kWh, total_increasing, vast in YAML          [F]
```

| Controle | Bevinding | Label |
|---|---|---|
| Ruwe bron | Envoy `lifetime_production`: `native_unit=Wh`, `suggested_unit=MWh`, `suggested_display_precision=3`, `TOTAL_INCREASING`, `ENERGY` (`enphase_envoy/sensor.py` r. 243–250). De state staat in MWh **zonder afronding**; afronding geldt alleen voor de weergave (`sensor/__init__.py`, conversie zonder `round`) | C |
| Utility meter | Neemt `unit_of_measurement` en `device_class` letterlijk over van de bron, maar **pas bij de eerste geldige state-wijziging van de bron** (`utility_meter/sensor.py` r. 426–430, 510–527). Vóór die eerste wijziging is `native_unit_of_measurement = None` | C |
| Waarden zonder eenheid | Een utility meter kan alleen > 0 worden ná `start()`. Die zet tegelijk de eenheid (r. 511–520). **Alle waarden die zonder eenheid zijn vastgelegd, moeten dus 0 zijn** (of `unknown`) | C |
| Template-conversie | `hems_adapter_pv_today`: MWh ×1000, Wh ÷1000, kWh ×1, `round(3)`. De availability accepteert alleen Wh/kWh/MWh, of 0 zonder eenheid. **Correct** | F |
| Unittest | `test_pv_today_from_local_utility_meter` dekt MWh, Wh, kWh, "0 zonder eenheid" (geldig), "1.5 zonder eenheid" (ongeldig) en `unavailable`. Lokaal gedraaid: **13 skipped** (testafhankelijkheden niet aanwezig), dus niet uitgevoerd | F |
| Live waarde, `state_class`, `device_class` van `hems_pv_envoy_vandaag` | **Niet geverifieerd** (niet blootgesteld) | ? |

**Conclusie §1.** De historische numerieke waarden zonder eenheid vertegenwoordigen **0**, in welke eenheid dan ook. Er is dus geen schaalprobleem met factor 1.000 in de historie. Home Assistant rapporteert nu MWh, omdat de utility meter de eenheid van de Envoy-lifetimeteller overneemt, en Home Assistant toont die teller standaard in MWh.

## 2. Oorzaakanalyse

| Kandidaat | Oordeel | Bewijs |
|---|---|---|
| **Utility-meter-initialisatie (PR #55)** | **Hoofdoorzaak** [C/F] | De helper is op 24-11 ~20:22 aangemaakt. Tot de eerste waardewijziging van de Envoy-lifetimeteller stond hij op 0 zonder eenheid, en de recorder legde statistieken vast met eenheid `None`. Bij de eerste wijziging werd de eenheid MWh. De recorder ziet daarna een andere eenheid dan in de metadata en maakt de reparatiemelding `units_changed`. PR #55 voorzag de toestand "0 zonder eenheid" wel (de test hierboven), maar niet het neveneffect op de statistieken |
| Templateconfiguratie | Geen oorzaak [F] | Alle templates hebben een vaste kWh-eenheid. De wijziging zit in de UI-helper, niet in YAML |
| Enphase-integratie | Levert de eenheid (MWh) als standaard; **geen fout** [C] | MWh is de `suggested_unit` van de core-integratie |
| HA-eenheidsconversie | Geen fout [C] | De conversie is verliesvrij (geen afronding van de state) |
| Statistiek-metadata | **Symptoom, geen oorzaak** | Metadata-eenheid `None` tegenover state-eenheid MWh |
| Enphase-synchronisatieproblemen | **Mogelijk verband met het tijdstip, niet met de oorzaak** [H] | Met PV-opbrengst vanaf zonsopgang (~07:35) zou de eerste wijziging rond de ochtend te verwachten zijn, niet rond 13:3x. Een late eerste wijziging past bij een lokale Envoy-lifetimeteller die tot ~13:30 stilstond of geen updates leverde. Dat is niet bewezen: het vereist de historie van die teller (`last_changed`). Ook zonder synchronisatieproblemen was de melding gekomen, alleen eerder |

**Gevolg van een eventueel late eerste update** [C]: door `periodically_resetting: false` en een cumulatieve bron wordt de opbrengst tot dat moment bij de eerste update in één keer geboekt. Er gaat dan geen dagenergie verloren, maar de verdeling over de dag klopt voor 25-11 tot ~13:30 niet. `hems_pv_today` stond in die periode op 0 kWh [H].

## 3. Integrale controle overige HEMS-energiesensoren

| Sensor | Eenheid / class | Bron | Bevinding |
|---|---|---|---|
| `hems_adapter_pv_today`, `hems_pv_today` | kWh, total_increasing | utility meter (omgerekend) | Consistent; conversie correct |
| `hems_battery_energy`, `_reserve_capacity`, `_usable_capacity`, `_available_above_reserve` | Wh, `energy_storage`, measurement | template | Consistent (Wh als voorraad, geen tellers); geen ×1000-fouten in de code |
| `hems_batterij_geladen_energie` / `_ontladen_energie` | kWh (integratie, prefix k, tijd h) | Anker W | Correct opgezet (W·h·k = kWh). Live niet geverifieerd |
| `homenet_dag_import` / `_export` (YAML utility meter) | kWh, eenheid van de bron | **`sensor.p1_meter_energie_import/export` (HomeWizard)** | Eenheid consistent. **Nevenbevinding:** deze dagtellers hangen aan de HomeWizard. Dat verklaart de 0 kWh van 21–24-11 (HomeWizard `unavailable`) en het herstel op 25-11 |
| `homenet_dagverbruik` | kWh, `total` | `homenet_dag_import` | Consistent |
| Fenna Energiebalans (`custom_components/fenna_energy_balance`) | W → kWh intern | `sensor.envoy_…_current_power_production` (vermogen), met schaling naar eenheid (`UNIT_W`) | Geraakt door `hems_pv_envoy_vandaag`: **nee** |
| HEMS-kern (`fenna_hems/core/engine.py`) | verwacht `pv.today` in **kWh**, grenzen 0–500 | — | Consistent met kWh. De component wordt volgens de uitroldocumentatie niet geladen |
| PV-correctiefactor (`fenna_battery_strategy`) | W tegenover W | `hems_adapter_pv_enphase_site_candidate` tegenover Forecast.Solar `power_production_now` | Geraakt door `hems_pv_envoy_vandaag`: **nee** |

**Dag-, maand- en jaartotalen** en **consistentie tussen het native energiedashboard, de Fenna Energiebalans en HEMS v2 Energy Intelligence:** **niet geverifieerd.** Daarvoor zijn statistieken en dashboarddata nodig; zie §6. In de code is geen schaalfout met factor 1.000 gevonden.

## 4. Impactanalyse

| Domein | Impact | Ernst |
|---|---|---|
| Historische energieregistratie | Alleen de eigen statistiek van `hems_pv_envoy_vandaag` (~17 uur aan nulwaarden zonder eenheid). Die statistiek zit voor zover bekend niet in het energiedashboard; de zonnebron daar is volgens PR #55 ongewijzigd [F, live niet geverifieerd]. Tot de reparatie keuze is gemaakt, compileert de recorder geen nieuwe statistieken voor deze sensor [C, standaardgedrag bij `units_changed`, te bevestigen] | Laag |
| PV-prognoses en correctiefactor | Geen. De factor gebruikt vermogen (W), niet de dagteller | Geen |
| Economische optimalisatie | Geen directe invloed. Hooguit was `hems_pv_today` op 25-11 tot ~13:30 0 kWh in de beslis- en dashboardcontext [H] | Laag |
| Schaduwstrategie en toekomstige autonomie | Geen. De beslispoorten gebruiken de vermogensprovider. Bovendien is PV-autonomie geblokkeerd (TD-PV-1), staat `advice_only` aan en valt 25-11 binnen de Observe-periode. **Advies:** markeer 25-11 00:00 tot ~13:3x als onvolledige registratie voor `hems_pv_today` in de Observe-evaluatie | Laag |

## 5. Bevindingen en herstel

### B1 — Statistiek zonder eenheid door initialisatie van de utility meter (oorzaak van de melding)
- **Feiten:** §1 en §2. Utility meter, eenheid pas bij de eerste bronwijziging; de waarden zonder eenheid zijn per constructie 0.
- **Getroffen:** alleen de statistiek van `sensor.hems_pv_envoy_vandaag`.
- **Ernst:** laag.
- **Keuze in het reparatievenster:** **optie 1, "Update the unit of the historic statistic values from '' to 'MWh', without converting"** (NL: de eenheid van de historische waarden bijwerken zonder om te rekenen).
  - Waarom: de oude waarden zijn 0, en 0 zonder eenheid is exact 0 MWh. Bijwerken zonder conversie is dus inhoudelijk juist en behoudt alle historie.
  - Waarom niet optie 2 ("alle oude statistieken verwijderen"): dat levert niets op en wist onnodig data.
  - **Voorwaarde (read-only, vooraf):** controleer in de Geschiedenis van `sensor.hems_pv_envoy_vandaag` (24-11 20:00 – 25-11 13:35) dat er zonder eenheid alleen 0 of `unknown` voorkomt. Staat daar onverwacht een waarde > 0 zonder eenheid, kies dan geen van beide en meld het eerst.
- **Toekomst** (pas na GO): bij nieuwe utility-meter-helpers het reparatievenster na de eerste update meteen met optie 1 afhandelen, of de helper pas aanmaken als de bron actief verandert. Voor de templates is geen wijziging nodig.

### B2 — Mogelijke stilstand van de Envoy-lifetimeteller tot ~13:30
- **Status:** hypothese.
- **Te controleren:** `last_changed` en de historie van `sensor.envoy_482335000893_lifetime_energy_production` op 25-11.
- **Impact bij bevestiging:** de verdeling van `hems_pv_today` over de dag klopt niet; het dagtotaal wel.
- **Herstel:** geen. Alleen markeren in de Observe-evaluatie.

### B3 — HomeNet-dagtellers zijn afhankelijk van de HomeWizard
- **Feit:** de bron is `sensor.p1_meter_energie_import/export`.
- **Impact:** tijdens de HomeWizard-uitval (21–24-11) stonden ze op 0 in plaats van op `unavailable`, wat misleidend is.
- **Open controle:** heeft de herkoppeling van de HomeWizard of de nieuwe meter een sprong in de dagteller veroorzaakt? 25-11 stond om 12:04 al op 12,35 kWh, wat hoog is. Te controleren in de historie van `sensor.p1_meter_energie_import` rond het herstelmoment.
- **Ernst:** middel voor de registratie, geen invloed op beslissingen.

## 6. Read-only controlestappen (zonder wijzigingen)

1. **Geschiedenis:** `sensor.hems_pv_envoy_vandaag` en `sensor.envoy_482335000893_lifetime_energy_production`, 24-11 20:00 – 25-11 14:00. Controleer dat er zonder eenheid alleen 0 voorkomt (voorwaarde B1) en bepaal wanneer de bron voor het eerst veranderde (B2).
2. **Ontwikkelhulpmiddelen → Statistieken:** `sensor.hems_pv_envoy_vandaag` (eenheid in de metadata), `sensor.hems_pv_today` en `sensor.hems_adapter_pv_today` (verwacht kWh). Zoek ook naar andere energiesensoren met een melding.
3. **Instellingen → Dashboards → Energie:** welke entiteit is de zonnebron? Is dat `hems_pv_envoy_vandaag`, controleer dan het dagtotaal 25-11 tegen de Enphase-app.
4. **Dagtotaal 25-11** (na zonsondergang): `hems_pv_today` (kWh) = `hems_pv_envoy_vandaag` × 1000 (MWh→kWh) ≈ Enphase-app (verschil < 1 %).
5. **Geschiedenis van `sensor.p1_meter_energie_import`** rond het herstel van de HomeWizard (B3).

## 7. Beoordeling

| Onderdeel | Oordeel | Toelichting |
|---|---|---|
| 1 Sensorcontrole | **WARN** | Keten, conversies en eenheden zijn uit config en code vastgesteld en correct. Live waarden en attributen zijn niet geverifieerd |
| 2 Oorzaakanalyse | **PASS** (oorzaak) / **WARN** (tijdstip) | Initialisatie van de utility meter staat vast in de broncode. Het verband met Enphase-synchronisatie is een hypothese |
| 3 Integrale controle | **WARN** | Geen eenheids- of ×1000-fouten in de configuratie. Dag-, maand- en jaartotalen en de consistentie tussen dashboards zijn niet verifieerbaar. Nevenbevinding B3 |
| 4 Impactanalyse | **PASS** | Geen invloed op prognoses, correctiefactor, optimalisatie of autonomie. Alleen een cosmetische, historische statistiek |
| 5 Herstelprocedure | **PASS**, onder voorwaarde | Optie 1 (eenheid bijwerken zonder conversie), na controle §6 stap 1. Niet uitgevoerd; wacht op autorisatie |
