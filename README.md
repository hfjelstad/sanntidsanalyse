# Sanntidsanalyse

> Strukturelle svakheter i sanntidsinformasjon for norsk jernbane — og et forslag til løsning.

## Innhold

- [**Problemanalyse**](problemanalyse.md) — Hva er galt med dagens sanntidsinformasjon?
- [**Løsningsforslag**](sammenstilling.md) — Kjøretøysentrert modell med sensorfusjon
- [**Detaljert bakgrunn**](sanntidsanalyse_svakheter.md) — Full analyse med alle detaljer

## Illustrasjoner

| Fil | Beskrivelse |
|-----|-------------|
| [sinuskurve_tog_oslo_lillehammer_v2.svg](sinuskurve_tog_oslo_lillehammer_v2.svg) | Togets kontinuerlige bevegelse (virkeligheten) |
| [sinuskurve_servicejourneys_deadrun.svg](sinuskurve_servicejourneys_deadrun.svg) | Oppdeling i ServiceJourneys + deadruns (systemets modell) |
| [virkelighet_vs_system.svg](virkelighet_vs_system.svg) | Virkelighet vs. system — side ved side |
| [arkitektur_forslag.svg](arkitektur_forslag.svg) | Foreslått kjøretøysentrert arkitektur |

## TL;DR

Norsk jernbane bruker en sikkerhetssensor fra 1872 som primærkilde for sanntidsinformasjon. Standarden finnes. Profilene finnes. Kravet finnes. Hardwaren finnes. Det eneste som mangler er erkjennelsen av at inndata er utilstrekkelig — og et krav om å bruke kildene som allerede sitter på toget.
