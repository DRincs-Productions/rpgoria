# TODO

## Ottimizzazioni future (non previste nella prima implementazione)

- [ ] **Simulazione semplificata degli NPC fuori camera nella mappa corrente** (API 121, futura opzione B). Valutare un aggiornamento logico più leggero degli NPC non visibili dalla camera, preservando coerenza di routine NQTR, posizione, tempi/velocità e collisioni quando necessario. La prima implementazione mantiene invece la simulazione completa degli NPC fuori camera (API 121 A). Definire in una futura API criteri di attivazione, sincronizzazione e gestione delle transizioni dentro/fuori camera prima di implementare.

## Comportamenti NPC futuri (da progettare)

- [ ] **Inseguimento di un bersaglio mobile** (API 122): comportamento NPC per inseguire il protagonista o altri personaggi, con aggiornamento del percorso e gestione di ostacoli, velocità e condizioni di arresto. Attivabile dal developer tramite trigger personalizzati.
- [ ] **Avvicinamento a un personaggio o bersaglio** (API 122): comportamento di avvicinamento con distanza di arrivo configurabile e rispetto di movimento e collisioni.
- [ ] **Movimento casuale naturale** (API 122): esplorazione/wandering credibile, con scelta di destinazioni, pause e cambi di direzione non meccanici, compatibile con griglia e movimento libero.

Questi comportamenti sono idee per implementazioni successive, non API confermate né funzionalità da realizzare subito.

## Strategie future configurabili per destinazioni irraggiungibili (API 128)

- [ ] **Impostazione del comportamento in caso di percorso impossibile**: permettere al developer di scegliere una strategia, mantenendo per la prima implementazione la fermata nella posizione attuale nel caso di `rpgoria.routine.continue(id)` impossibile.
- [ ] **Punto accessibile più vicino**: raggiungere la posizione percorribile più vicina alla destinazione irraggiungibile e fermarsi.
- [ ] **Ricerca naturale di una via d'uscita**: raggiungere un punto accessibile e muoversi lungo il perimetro degli ostacoli come se l'NPC cercasse un passaggio, con limiti per evitare loop infiniti.
- [ ] **Teletrasporto solo fuori camera**: dirigersi verso un punto accessibile vicino alla destinazione o scelto casualmente e, quando l'NPC è fuori dal campo visivo del giocatore, teletrasportarlo secondo regole da definire. Evitare teletrasporti osservabili dal giocatore.

Le strategie sono idee future: definire configurazione, ambito, collisioni, sicurezza e compatibilità con routine e mappe prima di implementarle.


## NQTR — Attività automatica dopo salti temporali (fuori ambito Rpgoria, API 155)

- [ ] **Progettare in NQTR un nuovo tipo di attività automatica**: indipendente da luogo/Location/Room, con personaggio `MC`, attivata anche quando un grande salto del tempo di gioco attraversa l'orario previsto (esempio 08:00 → 14:00, attività alle 10:00). Verificare prima come NQTR gestisce attualmente le diverse modalità di attività e i salti temporali; definire poi il contratto in NQTR.
- [ ] **Rispettare il confine NQTR/Rpgoria**: la gestione dell'esecuzione degli eventi temporali è di NQTR. Non aggiungere in Rpgoria `catchUp`, registri di eventi recuperati, deduplicazione/persistenza dedicata né storico delle posizioni. Le API 145–154 di Rpgoria sono superate dall'API 155; API 143–144 rimangono valide per il calcolo dello stato logico NPC dopo un salto temporale.

Questo è un promemoria progettuale per **NQTR** registrato nel repository Rpgoria; non autorizza modifiche al codice di nessuna libreria.
