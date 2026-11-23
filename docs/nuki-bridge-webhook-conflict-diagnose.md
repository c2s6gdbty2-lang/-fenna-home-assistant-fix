# Diagnose: Nuki Bridge — `ValueError: Handler is already defined!`

**Prioriteit:** P2 · **Modus:** READ-ONLY / DIAGNOSE · **Peilmoment:** ma 23-11-2026 15:21 CET
**Uitgevoerde acties:** uitsluitend statusuitlezing via de Home Assistant MCP (`GetLiveContext`, `GetDateTime`) en statische analyse van de broncode van Home Assistant Core. Er is niets herstart, herladen, gewijzigd of bediend. Er zijn geen scripts of intents uitgevoerd. Fenna en HEMS zijn niet aangeraakt. Er zijn geen credentials gebruikt.

Labels: **[F]** bewezen feit · **[C]** afgeleid uit broncode (bewezen voor de geanalyseerde versie) · **[H]** hypothese · **[?]** onbekend.

## Beperking van deze diagnose

Via de beschikbare koppeling heb ik **geen toegang** tot het HA-logboek, `.storage/core.config_entries`, de interne webhookregistraties, `ha core info` of het lokale netwerk. Deel C (historie) en deel D (netwerk) kon ik daarom niet zelf uitvoeren. Ze zijn uitgewerkt als read-only controlestappen in §6. De codeanalyse gebruikt HA Core **2026.2.3**, de recentste wheel die hier te downloaden was. Jouw Core-versie is nieuwer [?]; de relevante code moet dus nog worden vergeleken (§6, stap 1).

---

## 1. Vastgestelde feiten

| # | Feit | Bron |
|---|---|---|
| F1 | Om 23-11 15:12:35 trad `Error setting up entry 22885D2A for nuki` op, met `ValueError: Handler is already defined!` vanuit `_create_webhook → webhook.async_register` | Jouw logfragment |
| F2 | `22885D2A` is de **titel** van de config entry (het hardware-ID van de Bridge). Het **webhook-ID is niet de titel maar het interne `entry_id`** (een ULID). De Nuki-code registreert `webhook.async_register(hass, DOMAIN, entry.title, entry.entry_id, …)` | [C] `nuki/__init__.py` r. 79–81 |
| F3 | `async_register` gooit deze fout alleen als dat exacte webhook-ID al in de in-memory handlertabel `hass.data["webhook"]` staat. Die tabel is volatiel en is na elke Core-start leeg | [C] `webhook/__init__.py` r. 51–54 |
| F4 | Een `ValueError` uit `async_setup_entry` zet de entry in **`SETUP_ERROR`**. Er volgt **geen** automatische retry, dus de integratie blijft uit tot een herlaad of herstart | [C] `config_entries.py` (generieke `except Exception`) |
| F5 | De Nuki-integratie ruimt de HA-webhook bij een setupfout **niet** op via `async_on_unload`. Alleen bij een `NukiWebhookException` (geen URL, ongeldige credentials, `RequestException`) wordt `async_unregister` aangeroepen | [C] `nuki/__init__.py` r. 92–131 |
| F6 | De bridge-callbackregistratie loopt binnen `asyncio.timeout(10)`, terwijl elke HTTP-call naar de Bridge een timeout van **20 s** heeft (`DEFAULT_TIMEOUT = 20`). De `TimeoutError` die daaruit kan volgen wordt **niet** afgevangen in `_create_webhook` en ook niet in `async_setup_entry` | [C] `nuki/__init__.py` r. 120–131, `const.py` r. 13 |
| F7 | De HA Core-update stond op 21-11 13:28 nog op `on` (update beschikbaar) en staat nu op `off`. Datzelfde geldt voor HA OS. **Core en OS zijn tussen 21-11 en 23-11 bijgewerkt**, dus er was minstens één Core-herstart in die periode | [F] MCP-statusvergelijking |
| F8 | Er zijn geen entiteiten van de Nuki Bridge-integratie aan Assist blootgesteld. Hun status kon ik dus niet zien | [F] MCP |
| F9 | De HA-entiteiten met de naam **"Tuindeurslot"** (lock, Actuator, Batterij, Battery voltage, 2× Firmware, Identificeren, *User code temporary disable time*, *Wrong code limit*) zijn **Matter**-entiteiten, geen Nuki-entiteiten. De Nuki-integratie levert alleen de platforms `binary_sensor`, `lock` en `sensor`, en de namen "Wrong code limit" en "User code temporary disable time" zijn de vertaalsleutels van het Matter-DoorLock-cluster | [C] `nuki/__init__.py` r. 36; `matter/number.py` r. 498–523; `matter/strings.json` |
| F10 | Alle "Tuindeurslot"-entiteiten waren **al op 21-11 13:28 `unavailable`**, dus vóór de Nuki-fout van 23-11. Ze zijn dat nu nog steeds | [F] MCP-snapshots 21-11 en 23-11 |
| F11 | Er bestaat een `person`-entiteit **"Nuki_mqtt"** met status `unknown` | [F] MCP |

## 2. Waarschijnlijke hoofdoorzaak

**[H, hoge waarschijnlijkheid] Hypothese 1: een achtergebleven handler na een eerder mislukte setup in dezelfde Core-sessie.**

Redenering:
1. Na een Core-start is de handlertabel leeg (F3). De fout van 15:12:35 kan daarom **niet de eerste** setuppoging van deze sessie zijn geweest. Er moet eerder in dezelfde sessie een registratie zijn gebeurd die niet is opgeruimd.
2. De code laat de handler achter wanneer de setup na `webhook.async_register` faalt met iets anders dan een `NukiWebhookException` (F5).
3. Het meest voor de hand liggende pad is F6. De callbackregistratie bij de Bridge (`callback/list`, eventueel `callback/remove` en `callback/add`) duurt langer dan 10 s. Dat ligt voor de hand bij een trage of druk bezette hardware-Bridge, of direct na de herstart door de update (F7). Dan valt er een `TimeoutError`, belandt de entry in `SETUP_ERROR` en blijft de handler geregistreerd.
4. Elke volgende setuppoging in dezelfde sessie, zoals een herlaad via de UI, een herlaad door een automatisering of script, of een herontdekking, geeft vervolgens precies `Handler is already defined!`. Omdat `SETUP_ERROR` geen retry kent (F4), blijft de integratie daarna uit.

**Toetsbare voorspelling:** in het logboek van dezelfde Core-sessie staat vóór 15:12:35 een eerdere `Error setting up entry 22885D2A for nuki` met een *andere* traceback, waarschijnlijk `TimeoutError` (of een `KeyError`/JSON-fout uit `callback_list`). Tussen die twee pogingen zit geen `Home Assistant initialized`/`Starting Home Assistant`. Wordt dit gevonden, dan is de hoofdoorzaak bewezen.

## 3. Alternatieve verklaringen

| # | Verklaring | Status | Toelichting |
|---|---|---|---|
| H2 | Dubbele registratie door dezelfde integratie binnen één setup | **Uitgesloten** voor 2026.2.3 [C] | `_create_webhook` roept `async_register` precies één keer aan per setup; config entries hebben een setuplock |
| H3 | Een tweede config entry of een ander component claimt hetzelfde webhook-ID | **Zeer onwaarschijnlijk** [C] | Het ID is de unieke ULID van deze entry. Een tweede Nuki-entry heeft een ander `entry_id` en botst dus niet. Alleen een custom component die precies dit `entry_id` registreert zou kunnen botsen. Nog te controleren (§6, stap 3) |
| H4a | Een **custom component `nuki`** in `/config/custom_components` overschaduwt de core-integratie, met eigen (afwijkend) gedrag | **Niet uitgesloten** [?] | Zie de logregel "We found a custom integration nuki…" |
| H4b | Een regressie in de nieuwe Core-versie (F7), bijvoorbeeld een gewijzigde unload- of setupflow | **Niet uitgesloten** [?] | De codevergelijking met de geïnstalleerde versie ontbreekt nog |
| H5 | De integratie was nog `LOADED` en werd opnieuw opgezet zonder unload | **Onwaarschijnlijk** [C] | Bij een normale reload roept HA eerst `async_unload_entry` aan, en die meldt de webhook als eerste af |
| H6 | Netwerkprobleem als *oorzaak* | **Niet bewezen** | Kan wel de *aanleiding* zijn van de eerste (verborgen) mislukte poging, via de 10 s-timeout. De `ValueError` zelf is puur intern. Er is geen netwerkbewijs beschikbaar |

## 4. Impact

| Onderdeel | Status | Label |
|---|---|---|
| Nuki-integratie in HA | Vermoedelijk `SETUP_ERROR`; geen automatische retry | [C]+[H] |
| Entiteiten van de Nuki-integratie | Vermoedelijk `unavailable`; niet zichtbaar via MCP | [?] |
| Fysiek slot en Bridge | Waarschijnlijk normaal. De Bridge en het slot werken onafhankelijk van HA (Nuki-app, handbediening). Niet geverifieerd | [H] |
| "Tuindeurslot"-entiteiten (Matter) | `unavailable`, maar **los van deze fout**: ze komen uit Matter en waren al vóór 23-11 onbeschikbaar | [F] |
| Voordeur (Nuki Pro Ultra) | Niet geraakt door deze fout, voor zover die niet via dezelfde Bridge-entry loopt | [?] |
| Fenna / HEMS | Geen aanwijzing voor afhankelijkheid van de Nuki-entry. Fenna-veiligheidsredeneringen die een slotstatus gebruiken, krijgen wel `unavailable` | [?] |

**Belangrijke nevenbevinding (F9/F10).** De HA-entiteiten die als "Tuindeurslot" zichtbaar zijn, zijn Matter-entiteiten. Het herstellen van de Nuki Bridge-entry brengt deze dus **niet** terug. Er zijn twee mogelijkheden: (a) het tuindeurslot is (ook) via Matter gekoppeld en die koppeling is een apart probleem, of (b) de "Tuindeurslot"-namen horen bij een ander apparaat dan het Nuki 2.0-slot achter de Bridge. Een Nuki 2.0 ondersteunt bij mijn weten geen Matter [?, te verifiëren in de apparaatinformatie].

## 5. Herstelvoorstel (pas na GO)

**Optie A (minst ingrijpend): één gerichte herlaad is waarschijnlijk niet voldoende.**
Bij een herlaad van een entry in `SETUP_ERROR` roept HA `async_unload_entry` niet aan (`config_entries.py`: bij een state die niet `LOADED` is, wordt de entry alleen op `NOT_LOADED` gezet). De achtergebleven handler blijft dan staan en de herlaad geeft opnieuw dezelfde fout [C]. Daarom adviseer ik dit niet als eerste stap.

**Optie B (aanbevolen): één geplande Core-herstart** (geen host-reboot en geen Bridge-herstart).
- Effect: de in-memory handlertabel wordt geleegd (F3) en de Nuki-entry krijgt een schone eerste setuppoging.
- Risico's: 1–3 minuten zijn alle integraties onbeschikbaar. De Matter-, Zigbee- en cloudintegraties moeten opnieuw verbinden. Fenna-automatiseringen met `homeassistant.start`-triggers lopen opnieuw. De HEMS v2-validatie loopt tot en met 23-11. Plan de herstart dus pas ná afronding van die validatie, en in overleg.
- Restrisico: als de oorzaak van de eerste mislukte poging (vermoedelijk de 10 s-timeout naar de Bridge) blijft bestaan, kan de fout bij de eerstvolgende herlaad terugkomen. Dat is in de code ingebakken (F5/F6), dus een upstream issue/PR bij `home-assistant/core` is de structurele oplossing: de webhook opruimen bij elke setupfout, of `TimeoutError` afvangen.
- Na de herstart **niet** handmatig herladen zolang de setup goed gaat.

**Niet doen:** de integratie verwijderen en opnieuw toevoegen, of de Bridge resetten. Dat levert een nieuw `entry_id` en nieuwe entiteits-ID's op, zonder dat het de oorzaak raakt.

**Apart spoor:** de onbeschikbaarheid van de Matter-"Tuindeurslot"-entiteiten los onderzoeken (Matter Server-logs, Thread/wifi-bereik, batterij).

## 6. Read-only controlestappen en verificatieplan

Deze stappen voer je zelf uit, bijvoorbeeld via *Terminal & SSH*. Ze zijn alle alleen-lezen. **Plak geen tokens in rapporten.**

1. **Versie:** `ha core info`. Noteer de versie en vergelijk `homeassistant/components/nuki/__init__.py` met de hier geanalyseerde 2026.2.3 (F5/F6).
2. **Historie / sessies** (deel C):
   `grep -nE "Starting Home Assistant|Home Assistant initialized|Stopping Home Assistant|nuki|pynuki|Handler is already defined" /config/home-assistant.log /config/home-assistant.log.1`
   Zoek de eerste `Error setting up entry 22885D2A` binnen de sessie van 15:12:35 en de bijbehorende traceback. Controleer ook of de fout eerder voorkwam (eerdere `.log.1`, of herhaling na eerdere updates). Maakt dat het structureel of incidenteel?
3. **Config entries** (deel A), zonder token:
   `jq '.data.entries[] | select(.domain|test("nuki";"i")) | {entry_id,domain,title,source,disabled_by,unique_id,host:.data.host,port:.data.port}' /config/.storage/core.config_entries`
   Verwacht één entry met domein `nuki`. Controleer ook op `nuki_ng` of andere Nuki-varianten.
   `ls /config/custom_components | grep -i nuki` (H4a).
   `grep -rn "<entry_id uit stap 3>" /config --include=*.py --include=*.yaml` (H3).
4. **Wie herlaadde om 15:12?** Kijk in Instellingen → Logboek of in `home-assistant.log` naar `config_entries.reload` of een UI-actie rond 15:12:35, en naar Fenna-scripts in die minuut (bijvoorbeeld `fenna_deploy_reload`).
5. **Netwerk** (deel D, zonder token): `ping -c3 <bridge-ip>` en `nc -zv <bridge-ip> 8080` vanuit de SSH-add-on. Vergelijk het IP met `data.host` uit stap 3 en met de DHCP-lease in de router. Zoek in de logs naar `ConnectTimeout`, `ConnectionError` of `Read timed out` bij `pynuki`. Vraag de Bridge-API niet herhaaldelijk aan, want de Bridge is traag en beperkt in gelijktijdige verbindingen.
6. **Bridge-callbacks:** in de Nuki-app → Bridge → *Callbacks* (alleen bekijken). Verwacht maximaal één callback met het pad `/api/webhook/<entry_id>`. Ontbreekt die of zijn er meerdere, dan is dat relevant voor het herstel.

**Verificatie na herstel (na GO):**
- Instellingen → Integraties → Nuki Bridge staat op *Geladen* en er is geen nieuwe `Handler is already defined!` in het log.
- De Nuki-lock-entiteit heeft een geldige status en de batterij-entiteit een waarde.
- **De webhook werkt end-to-end:** bij handmatige bediening van het slot (door de bewoner zelf, niet via HA) verandert de HA-status binnen enkele seconden, zonder te wachten op het poll-interval.
- 7 dagen observatie, inclusief minstens één (geplande) Core-herstart, zonder herhaling van de fout.
- In de Bridge-callbacklijst staat precies één callback naar deze HA-instantie.
