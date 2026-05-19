# Svakheter i sanntidsinformasjonsflyten for jernbane

## Innledning

Dette dokumentet belyser strukturelle svakheter i hvordan sanntidsinformasjon flyter fra sensor til sluttbruker i norsk jernbane. Analysen tar utgangspunkt i strekningen Oslo S – Lillehammer som eksempel.

## Det fysiske versus det logiske

### Kjøretøyets kontinuerlige bevegelse

Et tog som trafikkerer Oslo S – Lillehammer beveger seg fysisk som en kontinuerlig pendel gjennom nettverket i løpet av en driftsdag (se *sinuskurve_tog_oslo_lillehammer_v2.svg*). Fra infrastrukturens perspektiv er dette ett og samme objekt som passerer sporfelter og akseltellere uten avbrudd.

### ServiceJourney-grensene er logiske, ikke fysiske

Fra et kundeperspektiv — og i rutedata (NeTEx/SIRI) — er den samme bevegelsen delt opp i diskrete **ServiceJourneys**, adskilt av **deadruns**, layover, vending eller førerbytte ved endestasjonene (se *sinuskurve_servicejourneys_deadrun.svg*). Hver halve bølge er én kundereise med eget avgangs-ID.

## Søknadsprosessen: Kundereise bestiller spor, materiell planlegges separat

Tilgang til sporet (ruteleie/kanaler) bestilles gjennom en søknadsprosess som er organisert rundt **kundereiseperspektivet** — dvs. ServiceJourneys med avgangstid, strekning og stoppmønster. Det er dette som utgjør rutetabellen.

Planlegging av **materiell** (hvilke togsett som faktisk skal kjøre) skjer derimot ut fra **tilgjengelighet** — en separat prosess hos togoperatøren. Infrastrukturforvalter (IM) kjenner ikke til hvilke togsett som er tilgjengelige eller hvordan de er allokert.

Konsekvensen av denne todelte prosessen:

```
Søknad om ruteleie       →  IM tildeler sporkapasitet  →  Sensor knyttet til spor
    (ServiceJourney)              (infrastruktur)              (infrastruktur)

Materiellplanlegging     →  Operatør allokerer togsett →  Kjøretøy-ID
    (tilgjengelighet)             (operatør-intern)           (ukjent for IM)
```

Siden sporet er bestilt per kundereise, er det **kundereisen som er koblingsnøkkelen** mellom sensor og informasjon. Men kjøretøyet eksisterer også *mellom* kundereisene — i deadruns, vendinger og opphold — perioder som ikke er synlige i søknaden om sporkapasitet på samme måte.

## Primærsensoren: Infrastrukturbasert deteksjon

Primærsensoren for sanntidsinformasjon på jernbane i Norge er **sporfelter og akseltellere** som er plassert langs sporet — altså i infrastrukturen. Disse sensorene:

- Detekterer at *noe* befinner seg på et sporfelt eller passerer et punkt
- Kjenner ikke til **hvilken ServiceJourney** kjøretøyet utfører
- Kjenner ikke til **hvem som opererer** kjøretøyet
- Har ingen informasjon om **passasjerer, destinasjon eller rutenummer**
- Kjenner ikke til **materiell-ID** — IM vet ikke hvilket togsett som fysisk okkuperer sporfeltet

### Historisk bakgrunn: Sikkerhetssensor ble sanntidskilde

At infrastruktursensoren ble primærkilde for sanntid er ikke et bevisst arkitekturvalg — det er en **historisk arv**.

Sporfelter (track circuits) ble oppfunnet i 1872 av William Robinson med ett eneste formål: **å bevise at en blokkstrekning er fri** slik at signalsystemet trygt kan slippe neste tog inn. Teknologien ble lovfestet i Storbritannia gjennom Regulation of Railways Act 1889 etter Armagh-ulykken, og er grunnlaget for automatisk blokksignalering (automatic block signalling) verden over.

Den opprinnelige designhensikten er entydig: *"A track circuit is an electrical device used to prove the absence of a train on a block of rail tracks to control railway signals"* (Wikipedia: Track circuit). Sensorens jobb er å besvare ett binært sikkerhetsspørsmål: **er blokken fri — ja eller nei?**

Men disse sikkerhetssensorene genererer som biprodukt en observerbar hendelse: *et tog kjørte inn i / ut av en blokk på tidspunkt T*. Da man senere ønsket å tilby sanntidsinformasjon til reisende, var dette den eneste tilgjengelige kilden med nasjonal dekning — sporfelter og akseltellere var allerede installert overalt. Sikkerhetssensoren ble gjenbrukt som sanntidssensor — uten at den var designet for formålet.

Konsekvensene av dette «gjenbruket»:

| Designet for (sikkerhet) | Brukt til (sanntid) |
|---|---|
| Er blokken fri? (ja/nei) | Hvor er toget? (posisjon over tid) |
| Anonym deteksjon (noe er der) | Identifisert bevegelse (tog 123 passerte) |
| Binær tilstand per blokk | Kontinuerlig posisjonssporing |
| Kun relevant i øyeblikket | Historikk og prediksjon |
| Fungerer uavhengig av ruteplan | Krever kobling til ruteplan for å gi mening |

Arkitekturen vi har i dag er altså ikke designet for sanntidsinformasjon — den er **tilpasset i etterkant** fra et sikkerhetssystem som tilfeldigvis produserer passeringshendelser.

## Vendingslister: Plaster på et strukturelt gap

For å koble sammen to påfølgende ServiceJourneys over et deadrun brukes **vendingslister** — manuelt eller semi-automatisk vedlikeholdte tabeller som sier at «tog X (nordgående) går videre som tog Y (sørgående)». 

Disse fungerer **av og til**, men er sårbare for:

- Endringer i materielldisposisjon (innstillinger, togbytte, ekstrainnsats)
- Forsinkelser som gjør at planlagt vending ikke er mulig
- Manglende oppdatering i sanntid når driftssituasjonen endrer seg

Vendingslistene er i praksis en **statisk kobling** som forsøker å dekke et **dynamisk gap** i informasjonsmodellen.

## Identifiserte svakheter

### 1. Tognummer-problemet: Operasjonelt vs. kommersielt

Historisk har det **operasjonelle tognummeret** (brukt i trafikkledelse og over samband) og det **kommersielle tognummeret** (brukt mot kunder i rutetabeller og billettsystemer) vært identiske. Dette skaper en grunnleggende spenning:

**Operasjonelle krav:**
- Tognummeret må være **kort, enkelt og verbalt kommuniserbart** over radio/samband mellom lokfører og togleder
- Av sikkerhetsgrunner kan det **ikke eksistere to tog med samme operasjonelle tognummer** samtidig i nettverket
- Nummeret identifiserer en *bevegelse i infrastrukturen* på et gitt tidspunkt

**Kommersielle krav:**
- Tognummeret identifiserer en *kundereise* (ServiceJourney) i rutetabell og billett
- Kunden forholder seg til «tog 123 kl. 08:00 til Lillehammer»
- Nummeret skal være gjenkjennelig og stabilt over tid

Når ett fysisk kjøretøy vender og starter en ny ServiceJourney, **må** det operasjonelle tognummeret endres (det kan ikke finnes to «tog 123» i nettet). Men fra kundens perspektiv er dette en helt annen avgang med eget nummer. Denne navneendringen ved vending er nettopp det vendingslistene forsøker å holde styr på: *«tog 61 (nordgående) går videre som tog 64 (sørgående)»*.

Problemet for sanntid:

- **Identitetsbrudd ved vending** — det fysiske kjøretøyet er det samme, men identifikatoren endres. Systemet mister kontinuitet.
- **Matching avhenger av navnekobling** — infrastruktursensoren ser et tog med nytt nummer etter vending, og trenger vendingslisten for å vite at det er samme kjøretøy
- **Ved avvik kollapser koblingen** — dersom tog 61 ikke kan gå videre som tog 64 (f.eks. ved togbytte), er det kun operatøren som vet dette — og informasjonen når ikke nødvendigvis sanntidssystemet

Hadde man i stedet sporet **kjøretøy-ID** (f.eks. togsettnummer) som et eget lag *under* tognummeret, ville identiteten vært bevart gjennom vending, og tognummerbytte blitt en tjenestelag-hendelse uten å bryte sanntidskontinuiteten.

### 2. Mapping-problemet: Infrastrukturhendelse → ServiceJourney

For at en sensorhendelse (kjøretøy detektert på sporfelt X) skal bli til nyttig sanntidsinformasjon (tog 123 er 3 minutter forsinket til Hamar), kreves en **koblingsprosess** som matcher:

```
Fysisk observasjon (sporfelt, tidspunkt)
    ↓ matching
Planlagt rute (ServiceJourney, timetable)
    ↓ beregning
Prediksjon (estimert ankomsttid per stasjon)
```

Denne koblingen er sårbar for:

- **Avvik fra plan** — ved forsinkelser, innstillinger eller ekstraavganger kan matchingen feile
- **Deadrun/vending** — i perioden mellom to ServiceJourneys finnes ingen aktiv kundereise å knytte observasjonen til, noe som skaper hull i sanntidsinformasjonen
- **Togbytte/sammenkobling** — fysisk endring i togsettet (splitting/kobling) synliggjøres ikke av infrastruktursensoren

### 3. Informasjonshull ved endepunkter (deadrun-gapet)

Når toget ankommer Lillehammer og vender, skjer følgende fra et informasjonsperspektiv:

1. Siste ServiceJourney (nordgående) avsluttes
2. En **deadrun-periode** inntreffer (stiplet linje i illustrasjon 2)
3. Neste ServiceJourney (sørgående) starter

I dette mellomrommet:
- Det er **ingen aktiv ServiceJourney** å rapportere sanntid for
- Eventuelle forsinkelser i vendeprosessen fanges ikke opp som kunderelevant informasjon før neste ServiceJourney formelt starter
- Kunder som venter på neste avgang fra Lillehammer **har ingen sanntidsinformasjon** om hvorvidt toget ligger an til å gå i rute

### 4. Arving av forsinkelse

En forsinkelse dør ikke når en ServiceJourney avsluttes — den **arves** av neste ServiceJourney som utføres av samme kjøretøy. Dersom tog nordover ankommer Lillehammer 8 minutter forsinket, og vendingstiden er planlagt til 12 minutter, har neste sørgående avgang kun 4 minutters margin før forsinkelsen propagerer videre.

I dagens modell er denne arvingen **usynlig i gapet**:

```
ServiceJourney 456 (nordgående)    deadrun/vending    ServiceJourney 789 (sørgående)
         ankomst +8 min ──────►  [informasjonshull]  ──────► avgang +?? min
                                   ↑
                          Arving skjer her,
                          men er ikke synlig
                          for kunden
```

Problemet forsterkes av:

- **Vendingstiden er bufferen** — men den er ofte dimensjonert for normaldrift, ikke for forsinkelser. Når forsinkelsen spiser opp bufferen, arves restforsinkelsen direkte.
- **Kunden på vendestasjonen vet ingenting** — ServiceJourney 789 viser "i rute" helt til den formelt starter, selv om det fysiske toget allerede er forsinket inn
- **Kaskadeeffekt nedover linja** — en arvet forsinkelse fra Lillehammer forplanter seg til alle etterfølgende stopp (Moelv, Hamar, ..., Oslo S), men prediksjonen for disse stoppene starter først når ServiceJourney 789 er aktivert
- **Uten materiellkjennskap** — sanntidsleverandøren kan ikke beregne arving fordi koblingen mellom ServiceJourney 456 og 789 kun finnes i vendingslisten, som kanskje ikke er oppdatert

I en kjøretøysentrert modell ville arvingen vært triviell: kjøretøy Z er 8 min sent → vendingstid 12 min → estimert forsinkelse på neste avgang: max(0, 8−12) = 0 min, men med redusert margin. Denne beregningen kan gjøres *umiddelbart*, ikke først når neste ServiceJourney formelt starter.

### 5. Sensoren ser kjøretøyet — ikke reisen

Infrastruktursensoren opererer på **nettverkslag** (fysisk plassering i spornett), mens kundeinformasjonen opererer på **tjenestelaget** (ServiceJourney, linje, destinasjon). Denne lagdelingen medfører:

| Infrastrukturlag | Tjenestelag |
|---|---|
| Sporfelt opptatt/ledig | Tog X er ved stasjon Y |
| Akseltelling | Togsammensetning |
| Signalstatus | Forsinkelsesårsak |
| Sporvekselposisjon | Plattforminformasjon |

Oversettelsen mellom lagene krever kontinuerlig vedlikeholdt **topologisk mapping** og **ruteplankobling** — begge er feilkilder.

### 6. Manglende redundans i sensorlaget

Når primærsensoren er infrastrukturbasert og det ikke finnes et komplementært sensorlag (f.eks. GPS fra kjøretøyet), oppstår:

- **Single point of failure** — ved feil i sporfeltdeteksjon forsvinner sanntidsinformasjonen helt
- **Lav oppløsning** — posisjon er kun kjent ved diskrete punkter (sporfelter), ikke kontinuerlig
- **Forsinkelsesmåling kun ved passeringspunkt** — faktisk forsinkelse mellom punkter er ukjent

### 7. Tidsforsinkelse i informasjonskjeden

Informasjonsflyten fra sensor til kunde involverer flere ledd:

```
Sporfeltdetektor → Signalanlegg → Trafikkstyringssentral → 
    → Mapping/matching → Sanntidssystem → SIRI-feed → 
        → Reiseplanlegger/app → Kunde
```

Hvert ledd introduserer latens og mulighet for feil.

### 8. Linjebrudd: Én sinuskurve blir til mange

Når det oppstår brudd på linja (infrastrukturfeil, anleggsarbeid, ras etc.), splittes den opprinnelige sammenhengende sinuskurven — ett tog mellom Oslo S og Lillehammer — i **to separate sinuskurver** for tog (én på hver side av bruddet), pluss **mange nye sinuskurver for buss-for-tog** som betjener bruddstrekningen.

```
Normalsituasjon:
    Oslo S ═══════════════════════════ Lillehammer
    [────── 1 sinuskurve (tog) ──────]

Brudd (f.eks. ved Hamar):
    Oslo S ════════ Hamar       Hamar ════════ Lillehammer
    [── sinuskurve 1 (tog) ──]  [── sinuskurve 2 (tog) ──]
                        🚌  🚌  🚌
    [── mange sinuskurver (buss-for-tog) ──]
```

Dette medfører:

- **Eksplosjonen i antall ServiceJourneys** — der det var én kundereise Oslo S → Lillehammer, finnes det nå potensielt tre: tog A, buss, tog B
- **Ny sensorproblematikk** — bussene har et *annet* sensorregime (GPS-basert, operatør-eiet) enn togene (infrastruktur-basert)
- **Matching-kompleksiteten øker dramatisk** — vendingslistene må nå koble på tvers av transportmidler og sensorsystemer
- **Korrelasjonsproblemet** — at buss X som ankommer Hamar korresponderer med tog Y som går videre nordover, er en logisk kobling som ikke er forankret i noen felles sensor
- **Kundens reise er fragmentert** — reisende må forholde seg til sanntid fra separate systemer med ulik pålitelighet og oppløsning

Bussene har potensielt bedre sensordata, og den er i det minste **knyttet til kjøretøyet** — i motsetning til togenes infrastrukturbaserte deteksjon. Men integrasjonen mellom de ulike sanntidskildene er svak.

I tillegg oppstår et **operativt matchingproblem**:

- Sanntidsleverandøren må nå holde kontroll på **to separate tog** på hver sin side av bruddet — der det tidligere var ett sammenhengende løp
- Matchingen mot vendingslister og ServiceJourney-kobling må bygges opp på nytt for den midlertidige situasjonen
- Sanntidsleverandøren har fortsatt **ingen kjennskap til materiellet** — det er ukjent om det er samme togsett som pendler på delstrekningen, eller om det er byttet
- Ved bruddet oppstår nye deadrun-gaper ved de improviserte endestasjonene (f.eks. Hamar) som ikke eksisterer i den opprinnelige planstrukturen

## Konsekvenser for reisende

1. **Upålitelig sanntid ved vending** — kunder på vendestasjoner (Lillehammer, Oslo S) har minst informasjon, til tross for at de er mest sårbare for forsinkelser
2. **Forsinkelsespropagering er usynlig** — at en nordgående forsinkelse vil forplante seg til påfølgende sørgående avgang er vanskelig å kommunisere i sanntid
3. **Svak prediksjon** — uten kjøretøybasert sensordata mangler grunnlag for god estimering av fremtidig ankomsttid

## Mulige tiltak

| Tiltak | Adresserer svakhet |
|---|---|
| GPS/GNSS fra kjøretøy som sekundærsensor | #3, #4 |
| Vehicle-centric sanntidsmodell (spore kjøretøy, ikke ServiceJourney) | #1, #2 |
| Publisering av deadrun-status til kundesystemer | #2 |
| Kortere informasjonskjede (edge computing ved sensor) | #5 |
| Bedre integrasjon mellom trafikkstyring og kundesystemer | #1, #5 |

## Forslag: Optimal håndtering — kjøretøysentrert sanntidsmodell

Dagens arkitektur forsøker å besvare spørsmålet *«Hvor er ServiceJourney X?»* ved å observere infrastrukturen. Den optimale modellen snur spørsmålet: **«Hva gjør kjøretøy Z akkurat nå, og hvilken ServiceJourney utfører det?»**

### Prinsipp: Spor kjøretøyet, utled tjenesten

```
┌─────────────────────────────────────────────────────────────────┐
│  KJØRETØYLAG (kontinuerlig, alltid aktivt)                      │
│                                                                 │
│  Kjøretøy Z: GPS-posisjon, hastighet, retning, status          │
│  ───────────────────────────────────────────────────────────►   │
│  Uavbrutt gjennom ServiceJourney, deadrun, vending, brudd       │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ kobling (automatisk eller regelbasert)
┌─────────────────────────────────────────────────────────────────┐
│  TJENESTELAG (logisk, hendelsesstyrt)                           │
│                                                                 │
│  ServiceJourney 1 │ deadrun │ ServiceJourney 2 │ deadrun │ ... │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ publisering
┌─────────────────────────────────────────────────────────────────┐
│  KUNDELAG (SIRI/sanntidsvisning)                                │
│                                                                 │
│  "Tog til Lillehammer: Forsinket 4 min"                         │
│  "Neste avgang fra Lillehammer: I rute (toget er på stasjon)"   │
└─────────────────────────────────────────────────────────────────┘
```

### Konkrete arkitekturkrav

**1. Kjøretøyet er primærobjektet**

- Hvert togsett rapporterer kontinuerlig posisjon (GPS/GNSS) uavhengig av om det utfører en ServiceJourney eller ikke
- Sanntidssystemet holder en **levende tilstandsmodell per kjøretøy** — ikke per ServiceJourney
- Deadruns og vendinger er eksplisitte tilstander i modellen, ikke informasjonshull

**2. ServiceJourney-kobling er en utledet egenskap**

- Koblingen «kjøretøy Z utfører nå ServiceJourney 456» skjer i sanntidssystemet basert på:
  - Posisjon + retning + tid → matching mot ruteplan
  - Bekreftet av operatør (fallback: automatisk inferens)
- Når koblingen brytes (vending/deadrun), faller systemet tilbake til kjøretøynivå — ikke til "ukjent"

**3. Vendingslister erstattes av tilstandsoverganger**

```
Kjøretøy Z:
  [ServiceJourney 456, nordgående] 
    → ankomst Lillehammer 
    → tilstand: VENDING (varighet: estimert 12 min)
    → [ServiceJourney 789, sørgående]
```

Kunder som venter på avgang 789 ser: *«Toget er på Lillehammer, vender — estimert avgang i rute»* — i stedet for ingen informasjon.

**4. Brudd-scenarioet håndteres naturlig**

Ved linjebrudd:
- Kjøretøy Z1 (sør for brudd) og Z2 (nord for brudd) rapporterer begge uavhengig
- Bussene rapporterer på samme modell (kjøretøy med GPS → ServiceJourney-kobling)
- **Korrespondansekobling** mellom tog Z1, buss B og tog Z2 er en eksplisitt relasjon i systemet
- Kunden ser hele reisekjeden med sanntid fra ende til ende, uavhengig av transportmiddel

**5. IM og operatør deler nødvendig informasjon**

| Aktør | Bidrar med | Mottar |
|---|---|---|
| Operatør | Kjøretøy-ID, GPS, materiellstatus | Sporkapasitet, signalinformasjon |
| IM | Sporfeltdata, signalstatus, infrastrukturtilstand | Kjøretøyposisjon (for trafikkledelse) |
| Sanntidsleverandør | ServiceJourney-kobling, prediksjon | Begge sensorkilder |

Infrastruktursensoren blir en **sekundær valideringskilde** (bekreftelse av passering) i stedet for primærsensor — og GPS fra kjøretøy gir kontinuerlighet, oppløsning og vedvarende identitet.

### Hva dette løser

| Dagens situasjon | Optimal modell |
|---|---|
| Informasjonshull ved deadrun | Kjøretøyet spores kontinuerlig — vending er en synlig tilstand |
| Vendingslister som statisk plaster | Tilstandsmaskin per kjøretøy erstatter manuell kobling |
| IM kjenner ikke materiell | Operatør deler kjøretøy-ID i sanntid |
| Brudd → eksplosjon i kompleksitet | Alle kjøretøy (tog + buss) på samme modell, korrespondanse er eksplisitt |
| Sensor ser spor, ikke reise | GPS ser kjøretøy — reise utledes |
| Single point of failure (sporfelt) | To uavhengige sensorkilder (GPS + sporfelt) |

### Forutsetninger og barrierer

- Krever at **operatør deler kjøretøydata** med sanntidssystemet (avtale/regulering)
- Krever GPS-utrustning med tilstrekkelig pålitelighet på alle togsett
- Krever endring i **ansvarsmodell** — i dag eier IM sensorlaget; i ny modell deles ansvaret
- Kan fases inn gradvis: starte med GPS som supplement, så la det bli primærkilde etter hvert som tillit bygges

### ERTMS/ETCS som muliggjører — men ikke løsningen i seg selv

Norge er i ferd med å innføre ERTMS (European Rail Traffic Management System) med ETCS (European Train Control System). I ETCS Level 2 og høyere kommuniserer togets ombordcomputer (European Vital Computer, EVC) direkte med en Radio Block Centre (RBC) via kryptert radiolink (GSM-R). Toget blir da en **unikt identifiserbar enhet** i nettverket — med kryptografiske nøkler og kontinuerlig posisjonsrapportering.

Analogien til en MAC-adresse er treffende: toget får en digital identitet og er adresserbart i sanntid. I ETCS Level 3 rapporterer toget selv sin posisjon og integritet — ingen sporfelt/akseltellere trengs lenger for togdeteksjon.

**Hva ERTMS løser for sanntid:**
- Toget er unikt identifisert på radioen — IM *vet* hvilket tog som er hvor
- Posisjon rapporteres kontinuerlig fra toget, ikke kun ved diskrete blokkgrenser
- Ingen anonymitet — matching mellom observasjon og tog er innebygd

**Hva ERTMS *ikke* løser uten videre:**
- ERTMS er et **sikkerhetssystem** — posisjonsdataen brukes til å utstede Movement Authorities, ikke til SIRI/passasjerinformasjon
- Informasjonsflyten fra RBC til kundesystemene er ikke spesifisert i ERTMS-standarden
- Koblingen til ServiceJourney (kundereise) må fortsatt gjøres i et separat lag
- Deadrun/vending-gapet forsvinner ikke automatisk — toget rapporterer posisjon, men det er fortsatt uklart hvilken *kundereise* det utfører mellom to ServiceJourneys

**Praktisk status:**
ERTMS-innføringen i Norge har blitt forsinket gjentatte ganger. Per 2015 var kun den østre Østfoldbanen utstyrt. Fullskala-utrulling lar fortsatt vente på seg — parallelt med at Danmark (planlagt ferdig 2033, opprinnelig 2021) og Sverige (2050) også opplever betydelige forsinkelser. Teknologien lover 20–30% kapasitetsøkning og eliminering av linjesidesignaler, men den faktiske leveransen er langt bak tidsplanen.

**Konklusjon for sanntidsarkitekturen:**
ERTMS gir det tekniske grunnlaget for en kjøretøysentrert modell — toget er identifisert og rapporterer posisjon. Men for å realisere gevinsten for sanntidsinformasjon må det bygges en **bro** mellom ERTMS-laget (sikkerhet) og kundelaget (SIRI/sanntid). Denne broen er ikke en del av ERTMS-standarden og må designes og implementeres separat.

### Sensorfusjon: Du trenger ikke vente på ERTMS

ERTMS-utrullingen er kronisk forsinket — og det er ingen grunn til å gjøre sanntidsinformasjon avhengig av den. Allerede i dag finnes det **flere uavhengige posisjonskilder om bord** på norske togsett:

| Kilde | Tilgjengelighet | Karakteristikk |
|---|---|---|
| **WiFi-router ombord** | De fleste togsett | Har GPS for geofencing/innholdsfiltrering. Rapporterer allerede posisjon til driftssentral. |
| **GPS i sikringsskap** | Nyere materiell | Dedikert posisjonsenhet i teknisk kabinett. Høy pålitelighet. |
| **Odometer** | Alle togsett | Hjulbasert avstandsmåling. Drifter over tid, men gir høyoppløselig relativ posisjon. |
| **ERTMS/EVC** | Kun der utrullet | Kryptografisk identitet + posisjon. Gullstandard — men begrenset dekning i overskuelig fremtid. |
| **Diverse IoT-enheter** | Varierer | Tablets, diagnostikk-bokser, passasjertelling-enheter — mange med egen GPS eller nettverkstilknytning. |
| **Infrastruktursensorer (sporfelt/akselteller)** | Full dekning | Dagens kilde. Anonym, diskret, forsinkelse — men 100% utbygd. |

**Arkitekturprinsippet: sensorfusjon med graceful degradation**

```
┌──────────────────────────────────────────────────────────────────┐
│  KJØRETØY (Vehicle ID: fast, kjent)                              │
│                                                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐  │
│  │WiFi-GPS │ │Sikr.GPS │ │Odometer │ │  ERTMS  │ │  IoT/div │  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬─────┘  │
│       │            │           │            │           │         │
│       └────────────┴─────┬─────┴────────────┴───────────┘         │
│                          ▼                                        │
│               ┌─────────────────────┐                             │
│               │  Fusjon / best av N │                             │
│               │  (vektet, validert) │                             │
│               └──────────┬──────────┘                             │
│                          │                                        │
└──────────────────────────┼────────────────────────────────────────┘
                           ▼
              ┌────────────────────────┐
              │  Posisjonsmelding ut   │
              │  (Vehicle ID + lat/lon │
              │   + tidsstempel        │
              │   + confidence)        │
              └────────────┬───────────┘
                           │
              ┌────────────┼────────────────────┐
              ▼            ▼                    ▼
     ┌─────────────┐ ┌──────────┐    ┌─────────────────┐
     │Sanntidssystem│ │Trafikk-  │    │Infrastruktur-   │
     │(SIRI/kunde) │ │styring   │    │sensorer (valider)│
     └─────────────┘ └──────────┘    └─────────────────┘
```

**Nøkkelegenskaper:**

1. **Redundans** — faller én kilde bort, lever systemet videre på de andre. Adresserer direkte svakhet #6.
2. **Ingen single point of failure** — hverken ERTMS-utrulling eller operatørvilje er en blocker alene.
3. **Confidence-score** — meldingen ut inkluderer et mål på hvor sikker posisjonen er. Tre GPS-er enige = høy confidence. Kun odometer = lav, men brukbar.
4. **Cutoff ved deadrun** — systemet vet alltid hvor kjøretøyet er, men *kundefeeden* kuttes bevisst ved ServiceJourney-grensene. Toget spores internt uavbrutt; utad vises kun aktive kundereiser.
5. **Infrastruktursensorene blir validator, ikke primærkilde** — sporfeltene bekrefter det kjøretøyet rapporterer, heller enn å være eneste sannhet.

**Hvorfor dette kan implementeres i dag:**
- WiFi-routeren er allerede installert og har GPS. Det krever en *dataavtale*, ikke en infrastrukturinvestering.
- Odometer finnes allerede — det trengs kun et grensesnitt.
- ERTMS-feeden kobles på *når den finnes* — modellen er kilde-agnostisk.
- Inkrementell utrulling: start med det du har, legg til kilder etter hvert.

### Ansvarsmodell: operatøren eier assignment

Spørsmålet «hvem eier fusjonslaget?» har et naturlig svar: **operatøren**. De eier hardwaren, de kjenner materiellomløpet, og de vet hvilken avgang kjøretøyet utfører. Å legge mapping-logikken hos en tredjepart (f.eks. Entur) ville gjenskape svakhet #2 — ekstern inferens av noe operatøren allerede vet.

Prinsippet er det samme som i luftfart: flyselskapet *deklarerer* «dette flyet opererer SK4455» — Avinor gjetter ikke.

**Meldingsformat — posisjon + assignment:**

```json
{
  "vehicle_id": "VY-BM74-2148",
  "position": { "lat": 60.793, "lon": 11.068 },
  "confidence": 0.97,
  "timestamp": "2026-05-19T14:32:01Z",
  "assignment": {
    "service_journey": "VYT:ServiceJourney:456",
    "status": "IN_PROGRESS",
    "next_stop": "NSR:Quay:0342"
  }
}
```

Ved vending/deadrun:
```json
{
  "vehicle_id": "VY-BM74-2148",
  "position": { "lat": 61.115, "lon": 10.466 },
  "confidence": 0.95,
  "timestamp": "2026-05-19T14:58:12Z",
  "assignment": {
    "status": "DEADRUN",
    "previous_journey": "VYT:ServiceJourney:456",
    "next_journey": "VYT:ServiceJourney:789",
    "next_departure": "2026-05-19T15:05:00Z"
  }
}
```

**Rollefordeling:**

| Aktør | Ansvar | Prinsipp |
|---|---|---|
| **Operatør** | Sensorfusjon ombord + assignment-deklarasjon | «Jeg vet hva kjøretøyet mitt gjør — jeg sier fra» |
| **Nasjonal plattform (Entur)** | Aggregering + distribusjon til kunder (SIRI) | Ren mottaker — ingen inferens, ingen blackbox |
| **Bane NOR** | Validering mot infrastruktursensorer | Bekrefter at kjøretøyet er der det sier det er |
| **Jernbanedirektoratet** | Definerer format + kravstiller via trafikkavtaler | «Dere *skal* levere denne feeden» |

**Ingen blackbox:**
- Operatøren vet *hva* og *hvorfor* — de deklarerer eksplisitt.
- Plattformen gjør ingen magi — den videresender strukturert data.
- Bane NOR validerer — de kan flagge avvik mellom egenrapportert posisjon og infrastruktursensorer.
- Kunden ser resultatet: sanntidsposisjon med klar kobling til sin avgang.

**Cutoff-logikk er transparent:**
- Under `IN_PROGRESS`: kunden ser toget i sanntid.
- Under `DEADRUN`: kunden ser *«Toget er på Lillehammer, neste avgang 15:05»* — ikke et sort hull.
- Operatøren styrer overgangen — det er de som vet når vendeprosedyren er ferdig og toget er klart for ny ServiceJourney.

**Regulatorisk forankring:**
Kravet kan hjemles i trafikkavtalene mellom Jernbanedirektoratet og operatørene. Tilsvarende hvordan AIS er pålagt i sjøfart via SOLAS-konvensjonen, kan posisjons- og assignment-rapportering gjøres til et kontraktsvilkår for å operere på det norske jernbanenettet.

---

*Illustrasjoner:*
- `sinuskurve_tog_oslo_lillehammer_v2.svg` — Togets kontinuerlige bevegelse i nettverket
- `sinuskurve_servicejourneys_deadrun.svg` — Dekomponering i ServiceJourneys og deadruns
