# Zonnecollector Digital Twin

Een interactieve, in-browser simulatie van de ECO-boiler zonnecollector-regeling, gebouwd om te experimenteren met PID-parameters zonder de echte installatie te moeten aanpassen. Gestart als hulpmiddel om samen met mijn kleinzoon Leon (14, physica/wiskunde) te spelen met wat "regeltechniek" en "optimaliseren" in de praktijk betekent.

**Live demo:** zet GitHub Pages aan voor deze repository (Settings → Pages → Deploy from branch → `main` / `/root`) en de simulator draait meteen op `https://<gebruikersnaam>.github.io/<repository>/`.

## Voor Leon: hoe lees je dit document

Dit README legt stap voor stap uit *waarom* de regeling doet wat ze doet — niet als een handleiding die je van boven naar onder moet doorworstelen, maar als een verhaal met een logische volgorde:

1. Eerst het **fysisch model**: welke twee dingen simuleren we eigenlijk, en met welke simpele natuurkunde?
2. Dan **dode tijd**: een subtiel maar cruciaal begrip uit de regeltechniek, dat verklaart waarom de pomp soms bleef pendelen ongeacht hoe we de regelaar instelden.
3. Dan de **regellogica zelf**: van "de pomp staat stil" tot "de pomp regelt zichzelf strak rond het doel", fase per fase, in de volgorde waarin ze ook echt na elkaar gebeuren op een zonnige ochtend.
4. Dan **P, I en D** apart uitgelegd — de kern van een PID-regelaar, met concrete cijfers die we zelf met deze simulator geverifieerd hebben.
5. Tot slot twee **echte bugs** die we onderweg vonden, als voorbeeld van hoe je in de praktijk fouten opspoort — dat is minstens even leerrijk als de regeltechniek zelf.

Je hoeft geen regel code te kunnen lezen om dit te volgen. Overal waar een formule staat, leggen we ook in woorden uit wat ze betekent.

## Wat dit is — en niet is

Dit is een **kwalitatief correct, niet kwantitatief exact** model. Het doel is intuïtie opbouwen over hoe P, I en D elk apart bijdragen, en waarom bepaalde instellingen tot pendelen of net tot een strakke vergrendeling leiden — niet om te voorspellen wat de echte installatie op de graad nauwkeurig zal doen. De grootteordes (tijdconstantes, temperatuurstijgingen) zijn zo gekozen dat ze *aanvoelen* zoals de echte data die we via de Photon-sketch verzamelden, maar zijn niet gekalibreerd op werkelijke metingen.

## Het fysisch model

De simulator simuleert eigenlijk maar **twee temperaturen**: de collector (Tsol, op het dak) en de onderste boilerlaag (BotH). Alles daartussen — de leidingen, de pomp, de warmtewisselaar — wordt vereenvoudigd tot een paar simpele regels.

### De collector warmt op en koelt af — Newton's afkoelingswet

De collector trekt elke minuut een stukje dichter naar de ingestelde "zon-intensiteit" (een soort evenwichtstemperatuur — hoger betekent niet meer instraling in W/m², maar wél een hogere temperatuur waar de collector uiteindelijk naartoe zou gaan als er niets anders gebeurde):

```
Tsol_nieuw = Tsol + HEAT_GAIN × (zon-intensiteit − Tsol)
```

Dit is exact dezelfde wiskundige vorm als Newton's afkoelingswet, die je wellicht al kent (of binnenkort tegenkomt) uit de fysica: hoe groter het verschil met de omgeving, hoe sneller de temperatuur verandert; hoe kleiner het verschil, hoe trager. Dat geeft een **exponentiële toenadering** naar de eindtemperatuur — nooit een rechte lijn, en nooit een abrupte sprong. `HEAT_GAIN = 0,12` bepaalt hoe snel die toenadering gaat (een soort tijdconstante).

Zodra de pomp draait, onttrekt ze warmte evenredig met hoe hard ze draait (PWM) én hoe groot het verschil is met de boiler:

```
onttrokken warmte = FLOW_COOL × PWM × max(Tsol − BotH, 0)
```

Die warmte verdwijnt bij de collector (`Tsol -= onttrokken warmte`) en een deel ervan (`BOIL_TRANSFER = 0,35`, want de boilerlaag is kleiner en niet alles komt aan) verwarmt de boiler. De boiler verliest daarnaast ook nog traag warmte aan de omgeving/aan warmwaterverbruik (`AMBIENT_LOSS`).

### Het "hete-plug"-effect — waarom de sensor liegt vlak na een herstart

Dit is het stukje fysica dat wéken speurwerk kostte om te snappen. Zolang de pomp stilstaat, warmt het **stilstaande water vlak bij de sensor** langzaam mee met de collector — een soort geïsoleerde "prop" die niet wegstroomt. Zodra de pomp herstart, komt die opgewarmde prop eerst voorbij de sensor, vóór het echte, koelere water uit de rest van het circuit er is. Het gevolg: de sensor toont eventjes een te hoge temperatuur, die na 1-2 minuten weer wegzakt naar de werkelijke waarde.

```
piekgrootte = min(stilstandsduur × 2,5 , 35)   [in °C, dooft uit over 2 simulatie-stappen]
```

Dit is precies wat we in de echte data zagen: dT die in één minuut van 20°C naar 36°C sprong, vlak na een herstart — geen meetfout, maar een fysisch, voorspelbaar verschijnsel.

Zie de constanten `HEAT_GAIN`, `FLOW_COOL`, `BOIL_TRANSFER`, `AMBIENT_LOSS` en `spikeMagnitudeFor()` bovenaan het script in `ECO_Digital_Twin_3aug26.html` — dat zijn de knoppen om het model zelf bij te stellen als het gedrag niet aanvoelt zoals de echte installatie.

## Dode tijd (transportvertraging) — waarom een goede regelaar toch kan blijven pendelen

Dit is misschien het belangrijkste regeltechnische inzicht van heel dit project, dus laten we het rustig opbouwen.

**Het probleem, met een analogie:** stel je speelt een computerspel waarbij je een balletje op een lijn moet houden, maar het beeld hinkt een halve seconde achter op je bewegingen. Je stuurt bij op basis van informatie die al voorbijgestreefd is — tegen de tijd dat je reageert, is de werkelijke situatie alweer veranderd. Hoe harder je dan probeert bij te sturen, hoe groter de kans dat je overcorrigeert, en het balletje blijft heen en weer slingeren — niet omdat je slecht stuurt, maar omdat je *vertraagde* informatie gebruikt.

Dat is precies wat er gebeurde op de echte installatie: op 31 juli gaven **vier volledig verschillende Kp/Ki/Kd-combinaties** allemaal hetzelfde ~9-11-minuten-slingerpatroon. Als de regelaar zelf het probleem was, zou een andere instelling het patroon moeten veranderen. Dat het niet veranderde, is de vingerafdruk van een **dode tijd** (transportvertraging) ergens in het systeem, niet van een verkeerd afgestelde PID.

**Waar komt die vertraging vandaan?** Bij een lage pompsnelheid beweegt het water traag door de leiding. Een verandering in de collector moet dus letterlijk fysiek "reizen" tot bij de sensor vóór de regelaar ze kan zien — hoe trager het debiet, hoe langer die reis duurt.

**Hoe simuleren we dat?** Niet met een letterlijke leidingvertraging (dat zou een "geheugen" van voorbije waarden vereisen — technisch een stuk complexer), maar met een vereenvoudigde **exponentiële naijling**:

```
gevoelde_temp_nieuw = gevoelde_temp + vertragingsfactor × (echte_temp − gevoelde_temp)
vertragingsfactor = clamp(PWM / 60, 0.05, 1)
```

Bij hoge PWM ligt de vertragingsfactor dicht bij 1: de gevoelde temperatuur volgt de echte bijna onmiddellijk. Bij lage PWM (en dus een hoge dode tijd) zakt de factor tot een absolute ondergrens van 0,05 — traag, maar nooit helemaal bevroren (zie de bugfix-uitleg verderop, dat "nooit helemaal bevroren" bleek zelf een addertje onder het gras). Zet `PWM-bodem` laag in de simulator en de vertraging wordt groot genoeg om zelfstandig te slingeren, ongeacht Kp/Ki/Kd; zet ze hoog (zoals op 17 juli, PWM-bodem 60) en de vertraging wordt kort genoeg om weg te regelen.

## De regellogica: van stilstand tot strakke regeling

De simulator draait een JavaScript-vertaling van exact dezelfde `solarPump()`-logica als de echte Photon-sketch (versie PID20, 3 augustus 2026). In plaats van de fasen zomaar op te sommen, lopen we het verhaal van een zonnige ochtend na — in de volgorde waarin het ook echt gebeurt.

### 1. WACHT — de pomp staat stil, en er gebeurt meer dan je zou denken

Zodra de zon opkomt, warmt de collector op terwijl de pomp nog stilstaat. Er lopen dan **twee onafhankelijke tellers** tegelijk, die elk een andere start kunnen triggeren:

- **dT_gefilterd** (het verschil tussen collector en boiler, door een filter gehaald — zie verderop) — zodra die boven 0°C komt, mág de pomp starten.
- De **rauwe Tsol-gradiënt**: hoe snel de collectortemperatuur zelf van minuut tot minuut stijgt, los van dT. Blijft die gradiënt 3 minuten na elkaar boven 0,5°C/min, dan triggert dat een **vroegere** start — zie VROEGSTART hieronder.

Wat er het eerst gebeurt, bepaalt via welke poort de pomp start.

### 2a. OPWARMEN — de gewone weg

Als dT_gefilterd positief wordt zonder dat de gradiënt-drempel gehaald werd, start de pomp via **OPWARMEN**: een open-lus fase (geen PID) waarin de PWM elke minuut met 4 tot 15 klimt — hoe steiler dT nog stijgt, hoe groter de stap:

```
stap = 4 + (15 − 4) × clamp(gradiënt / 2,0 , 0, 1)
```

Zodra dT_gefilterd 3 minuten na elkaar minder dan 0,3°C verandert (of na een vangnet van 10 minuten), wordt het circuit als "opgewarmd" beschouwd, en neemt REGIME (de PID) over.

*Waarom niet meteen de PID gebruiken?* Omdat de PID net na een herstart tegen de hete-plug-piek zou aanbotsen, en die piek verkeerd zou interpreteren als een structureel hoge fout — met een veel te hoge PWM-uitschieter tot gevolg. OPWARMEN laat het systeem eerst rustig tot rust komen.

### 2b. VROEGSTART — de snelle weg (nieuw sinds 2-3 augustus)

Op een snel zonnige ochtend hoeft de regeling niet te wachten tot dT zelf positief is: de collector zelf warmt dan al zichtbaar snel op. VROEGSTART start de pomp preventief zodra die gradiënt 3 minuten na elkaar boven 0,5°C/min blijft — nog vóór dT positief wordt.

De kracht waarmee de pomp dan start, is **lineair evenredig** met hoe steil die gradiënt op het moment van triggeren was:

```
frac = clamp((gradiënt − 0,5) / (3,0 − 0,5) , 0, 1)
PWM  = 30 + frac × (100 − 30)
```

Bij precies 0,5°C/min start de pomp dus zachtjes op PWM=30; bij 3,0°C/min of sneller op PWM=100. Dit is een **lineaire interpolatie** — een wiskundig begrip dat je uit de wiskundeles kent: een rechte lijn tussen twee punten, hier tussen (0,5 → 30) en (3,0 → 100).

De duur van VROEGSTART ligt niet vast: net als bij OPWARMEN wacht ze tot dT_gefilterd zelf 3 minuten na elkaar stabiliseert (met 40 minuten als vangnet), vóór REGIME overneemt. Een mooi neveneffect: **VROEGSTART dient meteen als een natuurlijke meting van de dode tijd** — hoe lang het duurt vóór het systeem op een vaste, gekende PWM stabiliseert, vertelt iets over hoe traag het systeem op dat moment reageert.

### 3. REGIME — de PID neemt het over

Dit is het hart van de regeling: een klassieke PID-formule die de PWM continu bijstuurt om dT_gefilterd op DT_TARGET = 2,5°C te houden.

```
fout = dT_gefilterd − 2,5
PWM  = 60 + Kp×fout + Ki×∫fout + Kd×(dfout/dt)
```

We leggen P, I en D hieronder apart uit — dat verdient zijn eigen sectie.

### 4. AFBOUW en STOP — hoe de pomp weer stilvalt

Zakt de PWM tot op de bodem (60) én blijft dT_gefilterd onder 0°C, dan loopt een teller op (**AFBOUW**). Blijft dat 5 minuten aanhouden zonder herstel, dan stopt de pomp definitief (**STOP**) — een laatste redmiddel tegen zinloos blijven draaien als de zon écht weg is.

### Twee directe overrides, los van dit hele verhaal

- **THERMOSIFON**: als dT_gefilterd toevallig positief is (bv. door een sensorstoring) terwijl de collector zelf nog kouder dan 22°C is, weigert de pomp te starten — anders zou je warm boilerwater richting een koude collector pompen en het net omgekeerde bereiken van wat je wil.
- **OVERVERHIT**: zodra de collector 90°C of meer bereikt, forceert de logica de pomp op volle kracht (PWM=255), ongeacht alle andere logica — een noodrem.

## P, I en D — elk apart, met echte, zelf-geverifieerde cijfers

Een PID-regelaar combineert drie manieren om op een fout te reageren. De analogie die vaak gebruikt wordt: stel je probeert een douchekraan op de juiste temperatuur te zetten.

- **P (Proportioneel)** = hoe hard je de kraan *nu meteen* draait, evenredig met hoe ver de temperatuur nog van goed zit. Grote fout → grote correctie; kleine fout → kleine correctie. Nadeel: P alleen bereikt het doel nooit helemaal — hoe dichter je komt, hoe kleiner je correctie, tot je uiteindelijk vlak bij het doel "blijft hangen" zonder het ooit exact te raken.
- **I (Integrerend)** = een geheugen van hoelang en hoe erg het al mis zat. Zelfs een kleine, aanhoudende afwijking bouwt zich langzaam op tot een correctie die P alleen nooit zou geven — en duwt zo die laatste, hardnekkige afwijking alsnog weg.
- **D (Differentiërend)** = reageert niet op de fout zelf, maar op *hoe snel* ze verandert. Een soort demper die al ingrijpt vóór de fout zelf groot wordt — nuttig om een plotse piek (zoals de hete-plug-transiënt) sneller af te vlakken.

We hebben dit met de simulator zelf getest en geverifieerd (constante zon-intensiteit, PWM-bodem=60, doel=2,5°C):

**Zuiver proportioneel (Ki=0, Kd=0):**

| Kp | gemiddelde dT_gefilterd |
|----|--------------------------|
| 1  | 4,16°C |
| 4  | 3,35°C |
| 8  | 3,14°C |
| 15 | 2,96°C |

Je ziet: hoe hoger Kp, hoe dichter bij het doel van 2,5°C — maar zelfs bij Kp=15 blijft er een kleine, blijvende afwijking over. Dat is precies de beperking van P alleen.

**Ki erbij, op Kp=4 vast (Kd=0):**

| Ki  | gemiddelde dT_gefilterd |
|-----|--------------------------|
| 0   | 3,35°C |
| 0,3 | 2,88°C |
| 1,0 | 2,63°C |

Met dezelfde Kp duwt een hogere Ki die resterende afwijking verder weg richting het doel — exact de rol die I hoort te spelen.

Probeer dit gerust zelf na met de sliders: zet Ki en Kd op 0, speel met Kp, en kijk of dT_gefilterd in de buurt van deze tabel uitkomt.

## Twee echte bugs die we vonden — en wat we ervan leerden

Naast de regeltechniek zelf is er nog een tweede, minstens even waardevolle les in dit project: **hoe je fouten opspoort in een systeem dat er op het eerste gezicht goed uitziet.**

### Bug 1: de "derivative kick" — een D-term die vergeet dat de tijd stilstond

De D-term rekent met `fout − vorige_fout`. Maar wat als je tijdens OPWARMEN (of VROEGSTART) helemaal geen PID gebruikt, en dus die "vorige fout" nooit bijwerkt? Bij de eerste stap ín REGIME vergelijkt de D-term dan plots met een fout van **meerdere minuten geleden**, in plaats van de vorige minuut — een kunstmatig grote sprong, die nog groeit naarmate Kd hoger staat. Het omgekeerde van wat een D-term hoort te doen (dempen, niet versterken).

De fix is klein — `pidPrevError` gewoon laten meelopen, óók tijdens de open-lus-fasen — maar de **manier waarop we hem vonden** is de eigenlijke les: eerst gevonden in déze simulator (makkelijker te doorzoeken dan de echte hardware), en pas daarna ontdekt dat **exact dezelfde bug ook in de echte Photon-sketch zat**, ongemerkt, tot de fix er expliciet naar overgeport werd. Twee stukken code die "hetzelfde" doen, delen niet automatisch dezelfde bugfixes — je moet dat zelf bewaken.

### Bug 2: de teller die zichzelf leegmaakte

Bij het bouwen van VROEGSTART moest een teller bijhouden hoeveel minuten na elkaar de Tsol-gradiënt al boven de drempel zat, *terwijl de pomp stilstaat*. De eerste versie reset die teller per ongeluk **elke keer opnieuw naar 0** zolang de pomp uit bleef — dus juist tijdens de minuten die hij hoorde te tellen. Het gevolg: de teller kon nooit voorbij 1 geraken, en VROEGSTART zou in de praktijk nooit triggeren.

Dit is een klassiek voorbeeld van een **off-by-logic-fout**: de code "leest" op het eerste gezicht logisch, maar de volgorde waarin twee stukjes logica elkaar raken (de teller ophogen vs. de teller resetten) zit net verkeerd. Zulke fouten vind je zelden door de code gewoon te lezen — pas door de logica stap voor stap na te bootsen (of, zoals hier, door het uit te testen op de echte installatie en de resultaten kritisch te controleren) kom je ze op het spoor.

## Het circuitschema (rechtsboven)

Een schematische, levende weergave van collector, boiler en pomp, opgebouwd uit:

- **Collector**: bovenaan, met een zon-icoontje en een kleur die met Tsol meeloopt. Toont de Tsol-waarde op de uitgang (rechts).
- **Boiler**: de helft zo breed als de collector en er precies onder gecentreerd. Zes lagen, van boven naar onder: **TopH, TopL, MidH, MidL, BotH, BotL**. Enkel BotH wordt effectief gesimuleerd (en toont dus een live temperatuur); de vijf andere zijn illustratieve labels.
- **Leidingen**: symmetrisch getekend — de hete leiding daalt recht naar beneden vanaf de collector-uitgang (rechts) en knikt pas helemaal onderaan naar de boiler; de retourleiding stijgt op dezelfde manier recht omhoog vanaf de boiler (links) en knikt pas helemaal bovenaan naar de collector-ingang. De streepjes bewegen mee, sneller bij hogere PWM, stil bij PWM=0.
- **Pomp**: op de retourleiding, met een live PWM-cijfer ernaast.
- **Voor/na de spiraal**: enkel de temperatuur ná de spiraal (afgekoeld, blauw) wordt apart getoond naast de BotH-laag — de temperatuur vóór de spiraal is gewoon Tsol zelf en staat al bovenaan bij de collector, dus die werd niet nodeloos verdubbeld. Formule: `BotH + (Tsol − BotH) × (1 − e^(−PWM/60))` — hoe lager de PWM, hoe meer warmte het water per doorgang afgeeft (langere contacttijd in de spiraal), dus hoe dichter de uittemperatuur bij BotH ligt i.p.v. bij Tsol.

## Info-knopjes bij de systeemparameters

Elke PID-parameter (en de PWM-bodem, en de opstartfasen) heeft een klein ⓘ-icoontje naast het label. Hover toont een korte teaser, klikken opent een uitlegvenster met wat de term doet en wat er gebeurt bij verhogen/verlagen — met concrete, zelf geverifieerde voorbeeldcijfers uit tests van deze exacte regellogica. De simulatie pauzeert automatisch zolang het venster open staat, en herneemt nadien de vorige pauzestatus. Sluiten kan via het kruisje, door buiten het venster te klikken, of met Escape.

## Bediening

- **Tsol — Zon-intensiteit**: de evenwichtstemperatuur waar de collector geleidelijk naartoe trekt — niet Tsol zelf (die volgt met vertraging, zoals in het echie).
- **BotH — boilertemperatuur nu**: sleep om de boilertemperatuur onmiddellijk te wijzigen (bv. na warmwaterverbruik). De schuifknop volgt nadien ook automatisch de live gesimuleerde waarde, behalve terwijl je hem zelf vasthoudt.
- **Kp / Ki / Kd**: dezelfde drie PID-knoppen als in de echte sketch, elk met een ⓘ-info-knop.
- **PWM-bodem**: hoe laag de pomp minimaal mag draaien tijdens REGIME — de knop om de dode-tijd-hypothese zelf te testen.
- **Snelheid**: hoeveel gesimuleerde minuten er per reële seconde verstrijken (verandert niets aan het regelgedrag zelf — enkel aan hoe snel je het ziet gebeuren, want elke stap blijft intern exact 1 gesimuleerde minuut).
- **☁ Wolk voorbij**: laat de zon-intensiteit 5 gesimuleerde minuten dalen en herstelt dan vanzelf.
- **🌤 Wisselende bewolking**: schakelt een voortdurende, wisselvallige zonkracht in (trage golving + willekeurige "gril"), i.p.v. een constante waarde — handig om te zien hoe de regeling omgaat met écht grillig weer, in plaats van steeds dezelfde geteste situatie.
- **↺ Reset**: begint opnieuw vanaf koude start. Alle sliders starten standaard op hun meest linkse (laagste) waarde, zodat je zelf van nul kan opbouwen.

## Lessen uit de echte-hardware-tests

Een aantal harde lessen uit het heen-en-weer traject op de echte Photon-installatie, die de simulator mee vormgaven:

- **Een vaste "kortsluiting" (bv. `Tsun>75°C → PWM=180`) kan een goed werkende PID volledig ondermijnen.** Op 29 juli bleek zo'n vaste override, losgekoppeld van de PID, een griezelig regelmatige zaagtand van 11 minuten te veroorzaken — telkens crashend exact op het moment dat de drempel overschreden werd. Les: laat de PID een heel bereik consequent zelf regelen, i.p.v. hem op een drempel te laten "overrulen".
- **Verschillende PID-instellingen die toch hetzelfde patroon geven, wijzen op een niet-PID-oorzaak.** Zie de sectie "Dode tijd" hierboven.
- **Bij twijfel: isoleer één variabele per test.** Zowel op de hardware als in de simulator bleek keer op keer dat het tegelijk wijzigen van meerdere parameters het achteraf onmogelijk maakt om te weten wélke wijziging welk effect had.
- **Een bug gevonden in de simulator moet je expliciet terugporten naar de echte code** — ze delen dezelfde logica, maar niet automatisch dezelfde bugfixes (zie "Twee echte bugs" hierboven).

## Openstaande vragen / Roadmap

- [x] Dode tijd / transportvertraging simuleren, als vereenvoudigde flow-afhankelijke naijling
- [x] OPWARMEN: gradiënt-gebaseerde open-lus opstartfase i.p.v. een vaste ramp
- [x] VROEGSTART: preventieve start op basis van de Tsol-gradiënt, met lineaire PWM-formule en stabiliteits-gebaseerde duur
- [ ] De dode-tijd-benadering vervangen door een echte, vaste-lengte transportvertraging (delay-buffer) i.p.v. een exponentiële naijling, voor wie het verschil tussen de twee zelf wil voelen
- [ ] Nachtblokkering (07u-21u) simuleren met een eigen kloklijn, los van de zon-intensiteit-slider
- [ ] Een "vergelijk twee instellingen naast elkaar"-modus (bv. huidige Kp/Ki/Kd naast een voorstel)
- [ ] Kalibratie van de fysica-constanten op een echte, geëxporteerde dag data (curve fitting) i.p.v. "aanvoelt goed"
- [ ] Exporteren van een simulatiesessie als CSV, in hetzelfde formaat als de echte Google Sheets-log, om ze naast elkaar te kunnen leggen
- [ ] Mobiele lay-out verder verfijnen (grafieken en schema worden vrij klein op smalle schermen)
- [ ] Eventueel ook live temperaturen simuleren voor de vijf niet-gemodelleerde boilerlagen (TopH/TopL/MidH/MidL/BotL), i.p.v. enkel BotH
- [ ] **Actief onderzoek (3aug26):** de live-test van PID20 toonde bij het verlaten van REGIME opnieuw enorme PWM-schommelingen — nog te analyseren of dit dezelfde ~9-11-minuten-oscillatie is die de aanleiding was voor VROEGSTART, dan wel een nieuw effect van de VROEGSTART→REGIME-overgang zelf

## Versiegeschiedenis (samengevat)

- **v1**: eerste simulator — sliders voor Tsun/BotH/Kp/Ki/Kd, temperatuur- en PWM-grafiek (incl. dT-lijn), consolevenster met uitleg per fase.
- **v2**: dT-lijn en bijhorende rechter-as uit de temperatuurgrafiek verwijderd; linker-as vast op 0-100°C; header versmald en een eerste circuitschema toegevoegd (collector, boiler met 6 lagen, pomp).
- **v3**: boiler verkleind (helft van de collectorbreedte) en gecentreerd; voor/na-spiraal-temperaturen gesplitst in rood/blauw links-rechts van de BotH-laag; BotH-temperatuur zelf zichtbaar gemaakt; zon-icoon en temperatuur-gekleurde rechthoeken toegevoegd voor collector én BotH.
- **v4**: D-term-bug gefixt (kunstmatige piek bij de opstart→REGIME-overgang); ⓘ-info-knoppen met uitlegvensters toegevoegd bij Kp/Ki/Kd.
- **v5**: BotH-slider volgt nu ook live de gesimuleerde waarde; het overbodige dubbele Tsol-label verplaatst/samengevoegd bovenaan bij de collector; leidingen links/rechts symmetrisch getekend (pomp mee verschoven); alle sliders starten voortaan op hun laagste waarde.
- **v6 (31jul26)**: dode tijd (transportvertraging) toegevoegd als flow-afhankelijke naijling tussen de werkelijke en de gevoelde collectortemperatuur — direct geïnspireerd door de ontdekking op de echte hardware dat vier verschillende PID-instellingen allemaal hetzelfde slingerpatroon gaven.
- **v7 (3aug26)**: de vaste opstart-ramp vervangen door **OPWARMEN** (gradiënt-gebaseerde open-lus fase) en **VROEGSTART** toegevoegd (preventieve start op basis van de Tsol-gradiënt, met een lineaire PWM-formule en een stabiliteits-gebaseerde duur); nieuw faselabel **WACHT**; info-modals bijgewerkt met opnieuw geverifieerde cijfers voor het nieuwe doel van 2,5°C; nieuwe modal die OPWARMEN/VROEGSTART uitlegt; README volledig herschreven, systematisch en didactisch opgebouwd voor Leon.

## Herkomst

Gebouwd na een aantal weken heen-en-weer testen met de echte Particle Photon-installatie (zie de sketch-versiehistoriek: PID9 t/m PID20, 25 juli – 3 augustus 2026) — de regellogica hier is bewust 1-op-1 dezelfde als daar, zodat een inzicht uit de simulator ook echt vertaalbaar is naar de sketch.

Belangrijkste mijlpalen sinds de eerste versie:
- **31jul26 (PID16/17):** derivative-kick-bug gevonden (eerst in de simulator, dan bevestigd in de echte sketch) en gecorrigeerd; dode tijd toegevoegd aan het model
- **1aug26 (PID18):** de vaste opstart-ramp vervangen door OPWARMEN, een gradiënt-gebaseerde open-lus fase die zelf beoordeelt wanneer het circuit "opgewarmd" is; trickle-start bij dT>0 i.p.v. te wachten op een hogere drempel
- **2aug26 (PID19):** VROEGSTART toegevoegd — een preventieve starttrigger op basis van de rauwe Tsol-gradiënt, nog vóór dT zelf positief wordt, met een vaste PWM en duur als eerste, eenvoudige versie
- **3aug26 (PID20):** VROEGSTART verfijnd met een lineaire PWM-formule (naar de trigger-gradiënt) en een stabiliteits-gebaseerde duur (i.p.v. een vaste timer) — dezelfde aanpak als OPWARMEN al gebruikte
