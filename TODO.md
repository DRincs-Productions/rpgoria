# TODO

## Ottimizzazioni future (non previste nella prima implementazione)

- [ ] **Simulazione semplificata degli NPC fuori camera nella mappa corrente** (API 121, futura opzione B). Valutare un aggiornamento logico più leggero degli NPC non visibili dalla camera, preservando coerenza di routine NQTR, posizione, tempi/velocità e collisioni quando necessario. La prima implementazione mantiene invece la simulazione completa degli NPC fuori camera (API 121 A). Definire in una futura API criteri di attivazione, sincronizzazione e gestione delle transizioni dentro/fuori camera prima di implementare.
