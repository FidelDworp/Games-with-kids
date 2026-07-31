# Zonnecollector Digital Twin

Een interactieve, in-browser simulatie van de ECO-boiler zonnecollector-regeling, gebouwd om te experimenteren met PID-parameters zonder de echte installatie te moeten aanpassen. Gestart als hulpmiddel om samen met mijn kleinzoon Leon (14, physica/wiskunde) te spelen met wat "regeltechniek" en "optimaliseren" in de praktijk betekent.

**Live demo:** zet GitHub Pages aan voor deze repository (Settings → Pages → Deploy from branch → `main` / `/root`) en de simulator draait meteen op `https://<gebruikersnaam>.github.io/<repository>/`.

## Wat dit is — en niet is

Dit is een **kwalitatief correct, niet kwantitatief exact** model. Het doel is intuïtie opbouwen over hoe P, I en D elk apart bijdragen, en waarom bepaalde instellingen tot pendelen of net tot een strakke vergrendeling leiden — niet om te voorspellen wat de echte installatie op de graad nauwkeurig zal doen. De grootteordes (tijdconstantes, temperatuurstijgingen) zijn zo gekozen dat ze *aanvoelen* zoals de echte data die we via de Photon-sketch verzamelden, maar zijn niet gekalibreerd op werkelijke metingen.

## De regellogica

De simulator draait een JavaScript-vertaling van exact dezelfde `solarPump()`-logica als de Photon-sketch, met dezelfde vijf fasen:

```
[OPSTART]     dT_gefilterd > 2.0°C (start) → open-lus ramp, losgekoppeld van de PID:
              PWM = 20 + 10×t_min, begrensd op [20, 60], voor t < 3 min
[REGIME]      PID rond DT_TARGET = 1.8°C, PWM = 15 + Kp·fout + Ki·∫fout + Kd·d(fout)/dt,
              begrensd op [15, 255], ramp-limiet max 150 PWM/min
[AFBOUW]      PWM ≤ 17 én dT_gefilterd ≤ 0.0°C → teller loopt op, pomp blijft draaien
[STOP]        teller ≥ 5 min zonder herstel → pomp uit (laatste redmiddel)
[THERMOSIFON] dT_gefilterd > 2.0°C maar Tsol < 22°C → pomp uit (directe override)
[OVERVERHIT]  Tsol ≥ 90°C → PWM = 255 (overschrijft alles)
```

`dT_gefilterd` is een EMA-filter (α=0,3/min) op de ruwe dT, precies zoals in de echte sketch — dat dempt de "hete-plug"-transiënt vóór de PID hem ziet.

**Belangrijk:** `pidPrevError` (nodig voor de D-term) blijft ook tíjdens de OPSTART-ramp elke minuut meelopen. Zonder dat zou de D-term bij de eerste REGIME-stap na een herstart een kunstmatige piek zien — een vergelijking met een fout van 3 minuten geleden in plaats van de vorige minuut — die groeit mét Kd, in plaats van te dempen zoals een D-term hoort te doen. Deze fout zat in de allereerste versie en is intussen gefixt.

## Het fysisch model

Twee gekoppelde thermische massa's, plus een klein extra effect voor de sensor-eigenaardigheid die ons wekenlang bezighield:

- **Collector (Tsol)** warmt op richting de ingestelde "Tsol — Zon-intensiteit" (een drijvende evenwichtstemperatuur, geen instraling in W/m²) en koelt af naarmate de pomp warmte onttrekt, evenredig met PWM × (Tsol − BotH). De collector-rechthoek in het schema kleurt live mee, van blauw (≤30°C) tot felrood (≥100°C).
- **Boiler-onderlaag (BotH)** wint een fractie van die onttrokken warmte, en verliest traag warmte aan omgeving/verbruik. Kleurt op dezelfde blauw-rode schaal mee.
- **"Hete-plug"-effect**: zolang de pomp stilstaat, bouwt zich een fictieve "opstuwing" op die evenredig is met de stilstandsduur. Bij het herstarten van de pomp geeft dit een korte (~2 minuten), afnemende piek bovenop de werkelijke collectortemperatuur — exact het verschijnsel dat we in de echte data zagen (dT die in één minuut van 20°C naar 36°C sprong, vlak na een herstart).
- **Temperatuur na de spiraal** (in de BotH-laag): puur illustratief — hoe lager de PWM, hoe meer warmte het water per doorgang afgeeft (langere contacttijd in de spiraal), dus hoe dichter de uittemperatuur bij BotH ligt i.p.v. bij Tsol. Formule: `BotH + (Tsol − BotH) × (1 − e^(−PWM/60))`.

Zie de constanten `HEAT_GAIN`, `FLOW_COOL`, `BOIL_TRANSFER`, `AMBIENT_LOSS` en `spikeMagnitudeFor()` bovenaan het script in `index.html` — dat zijn de knoppen om het model zelf bij te stellen als het gedrag niet aanvoelt zoals de echte installatie.

## Het circuitschema (rechtsboven)

Een schematische, levende weergave van collector, boiler en pomp, opgebouwd uit:

- **Collector**: bovenaan, met een zon-icoontje en een kleur die met Tsol meeloopt. Toont de Tsol-waarde op de uitgang (rechts).
- **Boiler**: de helft zo breed als de collector en er precies onder gecentreerd. Zes lagen, van boven naar onder: **TopH, TopL, MidH, MidL, BotH, BotL**. Enkel BotH wordt effectief gesimuleerd (en toont dus een live temperatuur); de vijf andere zijn illustratieve labels.
- **Leidingen**: symmetrisch getekend — de hete leiding daalt recht naar beneden vanaf de collector-uitgang (rechts) en knikt pas helemaal onderaan naar de boiler; de retourleiding stijgt op dezelfde manier recht omhoog vanaf de boiler (links) en knikt pas helemaal bovenaan naar de collector-ingang. De streepjes bewegen mee, sneller bij hogere PWM, stil bij PWM=0.
- **Pomp**: op de retourleiding, met een live PWM-cijfer ernaast.
- **Voor/na de spiraal**: enkel de temperatuur ná de spiraal (afgekoeld, blauw) wordt apart getoond naast de BotH-laag — de temperatuur vóór de spiraal is gewoon Tsol zelf en staat al bovenaan bij de collector, dus die werd niet nodeloos verdubbeld.

## Info-knopjes bij Kp/Ki/Kd

Elke PID-parameter heeft een klein ⓘ-icoontje naast het label. Hover toont een korte teaser, klikken opent een uitlegvenster met wat de term doet en wat er gebeurt bij verhogen/verlagen — met concrete, zelf geverifieerde voorbeeldcijfers uit tests van deze exacte regellogica. De simulatie pauzeert automatisch zolang het venster open staat, en herneemt nadien de vorige pauzestatus. Sluiten kan via het kruisje, door buiten het venster te klikken, of met Escape.

## Bediening

- **Tsol — Zon-intensiteit**: de evenwichtstemperatuur waar de collector geleidelijk naartoe trekt — niet Tsol zelf (die volgt met vertraging, zoals in het echie).
- **BotH — boilertemperatuur nu**: sleep om de boilertemperatuur onmiddellijk te wijzigen (bv. na warmwaterverbruik). De schuifknop volgt nadien ook automatisch de live gesimuleerde waarde, behalve terwijl je hem zelf vasthoudt.
- **Kp / Ki / Kd**: dezelfde drie PID-knoppen als in de echte sketch, elk met een ⓘ-info-knop.
- **Snelheid**: hoeveel gesimuleerde minuten er per reële seconde verstrijken (verandert niets aan het regelgedrag zelf — enkel aan hoe snel je het ziet gebeuren, want elke stap blijft intern exact 1 gesimuleerde minuut).
- **☁ Wolk voorbij**: laat de zon-intensiteit 5 gesimuleerde minuten dalen en herstelt dan vanzelf.
- **↺ Reset**: begint opnieuw vanaf koude start. Alle sliders starten standaard op hun meest linkse (laagste) waarde, zodat je zelf van nul kan opbouwen.

## Lessen uit de echte-hardware-tests (28-31 juli)

Deze simulator ontstond parallel aan een intensief traject van dag-tot-dag PID-tuning op de echte Photon-installatie. Een aantal harde lessen daaruit, die ook de latere versies van de simulator hebben beïnvloed:

- **Een vaste "kortsluiting" (bv. `Tsun>75°C → PWM=180`) kan een goed werkende PID volledig ondermijnen.** Op 29 juli bleek zo'n vaste override, losgekoppeld van de PID, een griezelig regelmatige zaagtand van 11 minuten te veroorzaken — telkens crashend exact op het moment dat de drempel overschreden werd. Les: laat de PID een heel bereik consequent zelf regelen, i.p.v. hem op een drempel te laten "overrulen".
- **Een D-term-bug (de "derivative kick") kan ontstaan bij een fase-overgang.** Als `pidPrevError` niet ook tíjdens een open-lus-fase (zoals OPSTART) blijft meelopen, vergelijkt de D-term bij de eerste PID-stap erna met een veel oudere fout dan de vorige minuut — dat geeft een kunstmatige piek die groeit mét Kd, het omgekeerde van wat een D-term hoort te doen. Deze bug is eerst in de simulator gevonden en gefixt (30-31 juli), en pas nadien ontdekt dat ze **ook in de echte Photon-sketch zat**, nooit gefixt vóór 31 juli. Les: een bug gevonden in de simulator moet je expliciet terugporten naar de echte code — ze delen dezelfde logica, maar niet automatisch dezelfde bugfixes.
- **Verschillende PID-instellingen die toch hetzelfde patroon geven, wijzen op een niet-PID-oorzaak.** Op 31 juli gaven vier duidelijk verschillende Kp/Ki/Kd-combinaties (3/0,15/0 · 4/0,15/1,2 · 6/0,5/1,2 · 8/0,6/1,2) allemaal hetzelfde ~9-11-minuten-slingerpatroon. Dat is de vingerafdruk van een **dode tijd** (transportvertraging) in het systeem, niet van een verkeerd afgestelde regelaar — zie de volgende sectie.
- **Bij twijfel: isoleer één variabele per test.** Zowel op de hardware als in de simulator bleek keer op keer dat het tegelijk wijzigen van meerdere parameters (zoals op 28 juli, toen `PWM_MIN`, `DT_TARGET` en de hele STOP-logica in één keer veranderden) het achteraf onmogelijk maakt om te weten wélke wijziging welk effect had.

## Dode tijd (transportvertraging) — nieuw in het model

Sinds de ontdekking hierboven simuleert het model ook een **flow-afhankelijke vertraging** tussen de werkelijke collectortemperatuur en wat de sensor "voelt": bij een lage PWM (traag debiet) beweegt het water trager door de leiding, dus duurt het langer vooraleer een verandering in de collector zich laat voelen bij de sensor; bij een hoge PWM is die vertraging kort. Dit is bewust een **vereenvoudigde, exponentieel-naijlende benadering**, geen letterlijke leidingvertraging (die zou een "geheugen" van voorbije waarden vereisen) — maar ze toont wel hetzelfde kwalitatieve gedrag: zet `PWM_MIN` laag en de vertraging wordt groot genoeg om een zelfstandige slingering te veroorzaken, ongeacht hoe je Kp/Ki/Kd instelt; zet `PWM_MIN` hoog (zoals op 17 juli) en de vertraging wordt kort genoeg om weg te regelen.

## Openstaande vragen / Roadmap

- [x] Dode tijd / transportvertraging simuleren (31 juli — zie hierboven), als vereenvoudigde flow-afhankelijke naijling
- [ ] De dode-tijd-benadering vervangen door een echte, vaste-lengte transportvertraging (delay-buffer) i.p.v. een exponentiële naijling, voor wie het verschil tussen de twee zelf wil voelen
- [ ] Nachtblokkering (07u-21u) simuleren met een eigen kloklijn, los van de zon-intensiteit-slider
- [ ] Een "vergelijk twee instellingen naast elkaar"-modus (bv. huidige Kp/Ki/Kd naast een voorstel)
- [ ] Kalibratie van de fysica-constanten op een echte, geëxporteerde dag data (curve fitting) i.p.v. "aanvoelt goed"
- [ ] Exporteren van een simulatiesessie als CSV, in hetzelfde formaat als de echte Google Sheets-log, om ze naast elkaar te kunnen leggen
- [ ] Mobiele lay-out verder verfijnen (grafieken en schema worden vrij klein op smalle schermen)
- [ ] Eventueel ook live temperaturen simuleren voor de vijf niet-gemodelleerde boilerlagen (TopH/TopL/MidH/MidL/BotL), i.p.v. enkel BotH

## Versiegeschiedenis (samengevat)

- **v1**: eerste simulator — sliders voor Tsun/BotH/Kp/Ki/Kd, temperatuur- en PWM-grafiek (incl. dT-lijn), consolevenster met uitleg per fase.
- **v2**: dT-lijn en bijhorende rechter-as uit de temperatuurgrafiek verwijderd; linker-as vast op 0-100°C; header versmald en een eerste circuitschema toegevoegd (collector, boiler met 6 lagen, pomp).
- **v3**: boiler verkleind (helft van de collectorbreedte) en gecentreerd; voor/na-spiraal-temperaturen gesplitst in rood/blauw links-rechts van de BotH-laag; BotH-temperatuur zelf zichtbaar gemaakt; zon-icoon en temperatuur-gekleurde rechthoeken toegevoegd voor collector én BotH.
- **v4**: D-term-bug gefixt (kunstmatige piek bij de OPSTART→REGIME-overgang); ⓘ-info-knoppen met uitlegvensters toegevoegd bij Kp/Ki/Kd.
- **v5**: BotH-slider volgt nu ook live de gesimuleerde waarde; het overbodige dubbele Tsol-label verplaatst/samengevoegd bovenaan bij de collector; leidingen links/rechts symmetrisch getekend (pomp mee verschoven); alle sliders starten voortaan op hun laagste waarde.
- **v6**: dode tijd (transportvertraging) toegevoegd als flow-afhankelijke naijling tussen de werkelijke en de gevoelde collectortemperatuur — direct geïnspireerd door de ontdekking op de echte hardware dat vier verschillende PID-instellingen allemaal hetzelfde slingerpatroon gaven.

## Herkomst

Gebouwd na een aantal weken heen-en-weer testen met de echte Particle Photon-installatie (zie de sketch-versiehistoriek: PID9 t/m PID15, 25-30 juli 2026) — de regellogica hier is bewust 1-op-1 dezelfde als daar, zodat een inzicht uit de simulator ook echt vertaalbaar is naar de sketch.
