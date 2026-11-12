# TODO

## Ottimizzazioni future (non previste nella prima implementazione)

- [ ] **Simulazione semplificata degli NPC fuori camera nella mappa corrente** (API 121, futura opzione B). Valutare un aggiornamento logico più leggero degli NPC non visibili dalla camera, preservando coerenza di routine NQTR, posizione, tempi/velocità e collisioni quando necessario. La prima implementazione mantiene invece la simulazione completa degli NPC fuori camera (API 121 A). Definire in una futura API criteri di attivazione, sincronizzazione e gestione delle transizioni dentro/fuori camera prima di implementare.

## Comportamenti NPC futuri (da progettare)

- [ ] **Inseguimento di un bersaglio mobile** (API 122): comportamento NPC per inseguire il protagonista o altri personaggi, con aggiornamento del percorso e gestione di ostacoli, velocità e condizioni di arresto. Attivabile dal developer tramite trigger personalizzati.
- [ ] **Avvicinamento a un personaggio o bersaglio** (API 122): comportamento di avvicinamento con distanza di arrivo configurabile e rispetto di movimento e collisioni.
- [ ] **Movimento casuale naturale** (API 122): esplorazione/wandering credibile, con scelta di destinazioni, pause e cambi di direzione non meccanici, compatibile con griglia e movimento libero.

Questi comportamenti sono idee per implementazioni successive, non API confermate né funzionalità da realizzare subito.
