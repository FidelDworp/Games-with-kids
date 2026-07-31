# Zonnecollector Digital Twin

Een interactieve, in-browser simulatie van de ECO-boiler zonnecollector-regeling, gebouwd om te experimenteren met PID-parameters zonder de echte installatie te moeten aanpassen. Gestart als hulpmiddel om samen met mijn kleinzoon Leon (14, physica/wiskunde) te spelen met wat "regeltechniek" en "optimaliseren" in de praktijk betekent.

**Live demo:** zet GitHub Pages aan voor deze repository (Settings → Pages → Deploy from branch → `main` / `/root`) en de simulator draait meteen op `https://<gebruikersnaam>.github.io/<repository>/`.

## Wat dit is — en niet is

Dit is een **kwalitatief correct, niet kwantitatief exact** model. Het doel is intuïtie opbouwen over hoe P, I en D elk apart bijdragen, en waarom bepaalde instellingen tot pendelen of net tot een strakke vergrendeling leiden — niet om te voorspellen wat de echte installatie op de graad nauwkeurig zal doen. De grootteordes (tijdconstantes, temperatuurstijgingen) zijn zo gekozen dat ze *aanvoelen* zoals de echte data die we via de Photon-sketch verzamelden, maar zijn niet gekalibreerd op werkelijke metingen.

## De regellogica

De simulator draait een JavaScript-vertaling van exact dezelfde `solarPump()`-logica als de Photon-sketch (versie 30jul26), met dezelfde vijf fasen:

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

## Het fysisch model

Twee gekoppelde thermische massa's, plus een klein extra effect voor de sensor-eigenaardigheid die ons wekenlang bezighield:

- **Collector (Tsol)** warmt op richting de ingestelde "zon-intensiteit" (een drijvende evenwichtstemperatuur, geen instraling in W/m²) en koelt af naarmate de pomp warmte onttrekt, evenredig met PWM × (Tsol − BotH).
- **Boiler-onderlaag (BotH)** wint een fractie van die onttrokken warmte, en verliest traag warmte aan omgeving/verbruik.
- **"Hete-plug"-effect**: zolang de pomp stilstaat, bouwt zich een fictieve "opstuwing" op die evenredig is met de stilstandsduur. Bij het herstarten van de pomp geeft dit een korte (~2 minuten), afnemende piek bovenop de werkelijke collectortemperatuur — exact het verschijnsel dat we in de echte data zagen (dT die in één minuut van 20°C naar 36°C sprong, vlak na een herstart).

Zie de constanten `HEAT_GAIN`, `FLOW_COOL`, `BOIL_TRANSFER`, `AMBIENT_LOSS` en `spikeMagnitudeFor()` bovenaan het script in `index.html` — dat zijn de knoppen om het model zelf bij te stellen als het gedrag niet aanvoelt zoals de echte installatie.

## Bediening

- **Zon-intensiteit**: de "kracht van de zon" op dit moment — hoger = de collector trekt naar een hogere evenwichtstemperatuur.
- **BotH**: sleep om de boilertemperatuur onmiddellijk te wijzigen (bv. na warmwaterverbruik).
- **Kp / Ki / Kd**: dezelfde drie PID-knoppen als in de echte sketch.
- **Snelheid**: hoeveel gesimuleerde minuten er per reële seconde verstrijken.
- **☁ Wolk voorbij**: laat de zon-intensiteit 5 gesimuleerde minuten dalen en herstelt dan vanzelf — handig om te zien hoe de regeling op een voorbijgaande dip reageert.
- **↺ Reset**: begint opnieuw vanaf koude start.

## Openstaande vragen / Roadmap

Dingen die we nog kunnen toevoegen naarmate het project vordert:

- [ ] Nachtblokkering (07u-21u) simuleren met een eigen kloklijn, los van de zon-intensiteit-slider
- [ ] Een "vergelijk twee instellingen naast elkaar"-modus (bv. huidige Kp/Ki/Kd naast een voorstel)
- [ ] Kalibratie van de fysica-constanten op een echte, geëxporteerde dag data (curve fitting) i.p.v. "aanvoelt goed"
- [ ] Exporteren van een simulatiesessie als CSV, in hetzelfde formaat als de echte Google Sheets-log, om ze naast elkaar te kunnen leggen
- [ ] Mobiele lay-out verder verfijnen (grafieken worden nu vrij klein op smalle schermen)

## Herkomst

Gebouwd na een aantal weken heen-en-weer testen met de echte Particle Photon-installatie (zie de sketch-versiehistoriek: PID9 t/m PID15, 25-30 juli 2026) — de regellogica hier is bewust 1-op-1 dezelfde als daar, zodat een inzicht uit de simulator ook echt vertaalbaar is naar de sketch.
