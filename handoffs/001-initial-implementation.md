# Rpgoria — Handoff 001: prima implementazione

Stato: specifica iniziale consolidata dalla conversazione progettuale (2026-10-30). Questo documento è un handoff per Codex, **non** autorizza a reinterpretare le decisioni confermate. Implementare Rpgoria per prima; Scrolloria è futura e non rientra in questo milestone.

## 0. Repos e ordine di lavoro
- Rpgoria: https://github.com/DRincs-Productions/rpgoria — implementazione principale.
- Documentazione: https://github.com/DRincs-Productions/rpgoria-wiki — successiva, non modificare in questa fase senza richiesta.
- Scrolloria: https://github.com/DRincs-Productions/scrolloria — futuro consumer di Rpgoria, non implementare ora.
- Scrolloria wiki: https://github.com/DRincs-Productions/scrolloria-wiki — non modificare ora.
- Nel workspace Codex saranno disponibili anche i sorgenti di Pixi'VN e NQTR. **Leggerli prima di definire firme, tipi, semantiche e hook**. Non duplicare funzioni già fornite dalle dipendenze.

## 1. Principi e integrazioni
- Package npm previsto: `@drincs/rpgoria` (verificare configurazione reale del repo). TypeScript, stretta integrazione con Pixi'VN e NQTR; NQTR peer obbligatoria.
- Un unico package, subpath adapter `/tiled`, `/ldtk`, `/paper-maker`, futuro `/three`. Usare parser/librerie esistenti, non riscrivere i formati.
- NQTR è la fonte di verità per Map → Location → Room, Room corrente, appartenenza dei personaggi alle Room e routine. **Non implementare un secondo sistema di collocazione dei personaggi**.
- La Room NQTR è una zona logica; la mappa grafica è associata all'ID Room.
- Rpgoria non crea né distrugge automaticamente le view/sprite dei personaggi o gli oggetti grafici dell'utente. **Eccezione confermata API 87 revisionata:** `rpgoria.layers.create(id, options)` crea il Container del layer RPG e lo aggiunge a Pixi'VN tramite `canvas.layers.add(id, container)`. La gestione pubblica successiva dei layer (lookup, rimozione, gerarchia) è esclusivamente di Pixi'VN.
- Separare stato spaziale, logica di movimento e rendering. Scrolloria in futuro consumerà internamente Rpgoria, ma avrà API pubbliche proprie; non piegare l'API Rpgoria a Scrolloria.

## 2. API pubbliche confermate
```ts
import {
  rpgoria,
  movement,
  RegisteredCharacterMapElements,
  CharacterMapStoredClass,
} from "@drincs/rpgoria";

await rpgoria.init({
  movement: { mode: "free", updateMode: "auto" },
});

movement.setDirection("liam", "right");
movement.moveTo("liam", { x: 300, y: 200 });

const inRoom = rpgoria.characters.getByRoom(roomId);
const current = rpgoria.characters.current;
```
- `rpgoria`: singleton condiviso creato dalla libreria, senza `new` da parte del consumer; aree `maps`, `layers`, `routines`, `characters`.
- `movement`: manager singleton esportato **separatamente** (non `rpgoria.movement`); metodi diretti con ID personaggio, non controller per personaggio.
- `rpgoria.init({ movement: { mode: "grid" | "free", updateMode: "auto" | "manual" } })`; default updateMode `auto`, ticker Pixi'VN condiviso. Modalità grid/free scelta all'init, non cambiabile in partita. Ripetute chiamate a init devono seguire il comportamento reale di `Game.init()` di Pixi'VN: verificare sorgente, non assumere.
- Tutte le API che accettano un ID personaggio devono utilizzare il **tipo TypeScript ufficiale Pixi'VN**, dopo averne identificato l'export reale (nessun alias inventato come contratto definitivo).
- `movement.setDirection(characterId, "up"|"down"|"left"|"right"|"stop")`: direzione persistente fino al successivo comando; stop non modifica facing. In grid termina lo step corrente; in free arresta al prossimo update.
- `RegisteredCharacterMapElements` esportato direttamente, stile `RegisteredCharacters` di Pixi'VN, con `add(...elements)`, `get(id)`, `values()`, `remove(...)`. Le **firme esatte**, duplicati, ID inesistenti e rimozione devono replicare Pixi'VN dove applicabile: ispezionare sorgente. `add(liamElement, emmaElement)` è variadico, NON array obbligatorio, NON coppie `{characterId,element}`.
- `CharacterMapStoredClass` **sostituisce** il precedente nome `BaseCharacterMapElement` ed è una classe da **estendere obbligatoriamente** per ogni elemento registrabile. Non consentire la registrazione di implementazioni arbitrarie della sola interfaccia. Il costruttore della classe richiede l'ID del personaggio Pixi'VN; espone `readonly characterId`. **API 22 confermata:** Rpgoria NON impone proprietà o metodi per accedere alla view (`displayObject`, `view`, `getDisplayObject()` non fanno parte del contratto obbligatorio). Lo sviluppatore sceglie autonomamente se e come associare/recuperare la grafica PixiJS e in quale layer inserirla.
- `rpgoria.characters.getByRoom(roomId): readonly CharacterMapElement[]`; `rpgoria.characters.current: readonly CharacterMapElement[]` (getter, non funzione). NQTR determina chi appartiene alla Room; Rpgoria restituisce solo i relativi elementi **registrati**, escludendo senza errore quelli assenti. `current` legge automaticamente la Room corrente NQTR a ogni accesso, senza `routines.refresh()`. Non aggiunge elementi ai layer.
- Il registro è persistente attraverso i cambi Room; la collocazione è responsabilità NQTR. Non reintrodurre `rpgoria.routines.getMapElements/currentMapElements`: nomi superati dalla decisione `rpgoria.characters`.
- `rpgoria.routines.refresh()` rimane un comando esplicito per rivalutare le routine dopo l'avanzamento del tempo di gioco, non per leggere la Room corrente.
- Il carattere `readonly` riguarda l'array restituito, **non** gli elementi contenuti. La lista non è un'interfaccia per modificare appartenenza Room.

## 3. CharacterMapStoredClass e stato spaziale
- Nessuna classe `RpgCharacter`; nessuna augmentation di `CharacterInterface` Pixi'VN e nessuna proprietà RPG sugli oggetti personaggio narrativi.
- Ogni personaggio Pixi'VN ha al massimo un `CharacterMapElement` registrato attivo, individuato dal proprio ID.
- La classe base obbligatoria custodisce l'ID e il contratto spaziale, **non** richiede un riferimento PixiJS. Il developer la estende e decide liberamente come ottenere/associare la view (Sprite/Container/AnimatedSprite o altro) e come sincronizzare visual/facing/animazioni con essa. Rpgoria gestisce stato spaziale e movimento senza imporre un'API grafica. **API 23 confermata:** CharacterMapStoredClass fornisce metodi sovrascrivibili invocati da Rpgoria per notificare aggiornamenti spaziali (es. posizione e facing) così che la sottoclasse possa sincronizzare la propria view. **API 24 confermata:** durante ogni frame di movimento Rpgoria invoca un metodo sovrascrivibile di aggiornamento visivo sulla sottoclasse, passando lo stato interpolato (in grid e free). In auto usa il ticker condiviso; in manual gli update esterni. Rpgoria calcola l'interpolazione, lo sviluppatore applica il risultato alla propria view. **API 25 confermata:** metodo sovrascrivibile `protected onVisualUpdate(state: VisualState): void`. `VisualState` contiene posizione visiva interpolata, facing, velocità e stato del movimento. **API 26 confermata:** `VisualState` include `x: number`, `y: number`, `facing: Direction`, `velocityX: number`, `velocityY: number`, `isMoving: boolean`. Questi dati sono indipendenti dalla rappresentazione (immagini alternate, Sprite 2D, 3D, ecc.). L'eventuale identificazione del modello/azione di movimento per scegliere animazioni (walk/run/custom) resta da decidere. Lo sviluppatore applica lo stato alla propria view. Gli eventi pubblici di movimento rimangono disponibili per logica di gioco, ma non sono il meccanismo richiesto per la sincronizzazione della view.
- Contratto spaziale discusso: `x,y,visualX,visualY,velocityX,velocityY,facing,solid,footprint,movementModel`. Consolidare tipi concreti durante implementazione senza mutare la semantica. `solid` default true; footprint indipendente dalle dimensioni sprite. `movementModel` compatibile Motion.js con modalità walk/run/custom.
- Il developer crea il visual e lo inserisce esplicitamente nel layer; Rpgoria non lo auto-instanzia, auto-inserisce o distrugge. Il developer controlla anche cleanup e cambio Room.

## 4. Movimento
- Coordinate `{x,y,unit?:"pixel"|"cell"}`: grid default cell, free default pixel. Celle rettangolari; origine al centro con offset appropriato.
- Grid: 4 direzioni cardinali, passi completi, tasto tenuto ripete, rilascio termina step, buffer di una direzione per 150ms. Posizione logica cambia a fine step, visual interpolato. Prenotazione footprint swept durante step, arbitrazione deterministica, vietati attraversamenti/scambi fra solidi. Teleport libera prenotazioni; errori riportano allo stato iniziale step.
- Free: smoothing visivo con ritardo contenuto, movimento continuo, stop al successivo update.
- Parametri temporali/velocità attraverso `MovementModel` compatibile Motion.js, non vecchi valori numerici hardcoded. Motion.js easing/spring/keyframes per interpolazione ma senza overshoot fisico o violazione collisioni.
- `moveTo`: pathfinding, nearest reachable come fallback (`reached_nearest`), ripianificazione per ostacoli dinamici, attesa massima 3s prima del fallback, nuovo comando sostituisce precedente, tolleranza arrivo free 1px.
```ts
type MoveToCancelReason = "explicit" | "replaced" | "manual_input" | "teleport" | "unregistered";
type MoveToResult =
  | { status: "reached" }
  | { status: "reached_nearest" }
  | { status: "unreachable" }
  | { status: "cancelled"; reason?: MoveToCancelReason };
```
- Errori tecnici rigettano Promise. `cancelMoveTo(): Promise<void>` idempotente, risolve quando fermo; grid completa step salvo teleport, free prossimo update. Prima ragione di cancel prevale salvo override teleport. Input manuale valido annulla moveTo anche se bloccato o mentre si attende collision data.
- Collision data async condivisi e cache per Room attiva; errori ritentabili, ostacoli dinamici inclusi nel pathfinding, risultati async obsoleti ignorati.
- Movimento non cambia mai Room; teleport può cambiare Room secondo NQTR. Solo personaggi Room attiva vengono simulati. Al cambio Room cancellare movimenti e prenotazioni; non distruggere visual del developer.

## 5. Eventi e handler
- Eventi: `movementStart`, `movementEnd`, `positionChanged`, `facingChanged`, `movementBlocked`, `teleported`, `moveToCompleted`, `characterEncounter`.
- Posizioni tipizzate, blocked deduplicato, completed solo per risultato ordinario, niente movement event per zero distanza salvo completed.
- Handler concettuale `addMovementHandler({event,actor?,characterId?,once?,handler})`. Decisioni sincrone, globali prima dei locali, ultimo valore esplicito per proprietà prevale, isolamento eccezioni via `handlerError`, snapshot delle liste, no serializzazione; encounter supporta `preventDefault`. Firme precise da consolidare nella fase API.

## 6. Routine e NQTR
- NQTR gestisce collocazione e priorità/onRun nativi. Rpgoria legge le routine, non duplica il loro sistema.
- `rpgoria.routines.refresh()` dopo avanzamento clock: routine invariata mantiene posizione NPC; nuova routine con inizio attraversato dall'ultimo refresh: stessa Room cammina dalla posizione attuale, Room diversa spawn nella Room target e cammina; alla prima inizializzazione della routine attiva teleport direttamente alla posizione di partenza.
- Waypoint e callback custom supportati; nessun trigger automatico di prossimità.
- Movimenti custom delle routine condividono risultati/cancellazioni/eventi/collisioni di movement e lo stesso ticker, senza ticker indipendente.
- Hook NQTR sincroni; effetti grafici asincroni successivi.

## 7. Mappe, layer e rendering
- Adapter per Tiled, LDtk, Paper Maker, futuro Three; registrazione mappa tramite Room ID. Esempio indicativo `registerTiledMap(roomId,{asset})`: asset alias, URL o oggetto già caricato; `prepare(roomId)` esplicito, **nessun** `maps.activate` (Room NQTR è autorevole).
- **Sostituito da API 87 revisionata:** non richiedere un root `rpgoria.layers.container` da aggiungere manualmente. Ogni layer RPG viene creato con `rpgoria.layers.create(id, options)`, che chiama `canvas.layers.add(id, container)` di Pixi'VN. Il developer usa le API Pixi'VN per lookup, rimozione e gestione della gerarchia.
- Nel root si aggiungono **singoli layer** della mappa preparata, non il container di tutta la mappa; developer decide ordine/composizione.
- Layer `static` (default) e `movement` (Y-sort). Più movement layer ordinati separatamente, nessun sort incrociato. Elementi statici/dinamici individuali ordinabili per depthY con priorità proprietà > override > base automatica. Tile Layer piatto come unità; tile object in Object Layer individualmente ordinabili.
- Le view grafiche eventualmente associate alle sottoclassi di CharacterMapStoredClass vengono ottenute e aggiunte dal developer al layer prescelto, senza un accessor imposto da Rpgoria. Layer lifecycle interamente developer-controlled, inclusi cambio Room, ricostruzione e cleanup. Rpgoria deve evitare riferimenti stale a oggetti distrutti.
- Un ticker condiviso Pixi'VN per tutti i movimenti attivi in modalità `auto`, si arresta quando idle salvo controlli di oggetti dinamici sortable; modalità `manual` con update esterno alternativa. Ticker effetti separati ammessi.

## 8. Fuori scope / TODO
- Fade automatico di tetti/edifici per aree interne, con Motion.js alpha e proprietà Tiled: TODO, non MVP.
- Edifici 2.5D e personaggi 3D futuri: rendering 3D su texture poi PixiJS sprite in movement layer, Y-sort come unità; niente coordinate 3D nel core adesso.
- Interazioni, inventario, combattimento, trigger e altri sistemi gameplay: TODO.
- Scrolloria futuro: side-scroller X-only free, salto opzionale, attraversamento personaggi default, encounter può bloccare; non implementare qui.

## 9. Verifiche obbligatorie prima di scrivere implementazione
1. Ispezionare API e sorgente di `RegisteredCharacters` Pixi'VN: variadic add, remove, get inesistente, duplicati, values, typing, import ufficiale ID.
2. Ispezionare `Game.init()` Pixi'VN: ripetizione chiamate e ticker/lifecycle.
3. Ispezionare NQTR: API reali Room corrente, query personaggi per Room, routine e refresh, eventi, priorità/onRun. Non ipotizzare nomi.
4. Ispezionare Motion.js e dipendenze già presenti prima di fissare shape di `MovementModel`.
5. Risolvere i dettagli API **ancora aperti** con l'utente prima di esporli come contratti stabili, in particolare eventuali ulteriori informazioni di animazione in `VisualState` (es. movimento walk/run/custom), altri eventuali hook, firme eventi/handler, adapter e map/layer helpers. **API 22 chiusa:** nessun accessor obbligatorio della view.
6. Aggiungere test per registro stile Pixi'VN, filtro Room, current dinamico, esclusione mancanti, readonly typing, cambio Room, movimento e collisioni.

## 10. Priorità prima implementazione
1. Setup package e verifiche dipendenze.
2. `CharacterMapStoredClass` obbligatoria, registro e `rpgoria.characters` integrati con NQTR.
3. Init singleton e movement manager con ticker auto/manual.
4. Movimento grid/free, collisioni e pathfinding.
5. Routine NQTR, eventi e handler.
6. Map adapters e layer composition.
7. Test e documentazione. Segnalare API ancora da approvare anziché inventarle.

**Nota di stato:** questa è una fotografia completa delle decisioni principali emerse finora, non una specifica finale di ogni firma. Eventuali nuove decisioni API prevalgono su questo handoff e devono essere riportate in successivi handoff numerati.


### API 27 — Dati di animazione
`VisualState` include `movementId: string | null`, `speed: number` e `deltaTime: number` (secondi). `speed` è il modulo della velocità visiva effettiva in pixel/secondo, ottenuta dalle componenti `velocityX` e `velocityY` dopo interpolazione Motion, inclusi diagonali, easing e rallentamenti. La view e le animazioni restano interamente sotto il controllo dello sviluppatore.


### API 28 — Ultimo aggiornamento visivo
Alla conclusione effettiva del movimento, Rpgoria invoca un'ultima volta `onVisualUpdate(state)` con `speed: 0`, `velocityX: 0`, `velocityY: 0`, `isMoving: false`, `movementId: null`; `x`, `y`, `facing` conservano i valori finali. Durante un rallentamento Motion `speed` decresce progressivamente e `isMoving` resta true fino al termine effettivo. Nessun aggiornamento continuo idle obbligatorio. La view/animazione idle è gestita dal developer.


### API 29 — Sincronizzazione iniziale alla registrazione
Quando `RegisteredCharacterMapElements.add(...)` registra un elemento, Rpgoria richiama automaticamente il suo `onVisualUpdate(state)` con posizione e facing iniziali, `velocityX: 0`, `velocityY: 0`, `speed: 0`, `isMoving: false`, `movementId: null`. Il developer predispone la propria view prima della registrazione; Rpgoria non crea né inserisce automaticamente la view nei layer. Lo stesso hook copre inizializzazione, movimento e arresto.


### API 30 — Facing cardinale e angolo direzionale
`VisualState.facing` resta a quattro direzioni (`up`, `down`, `left`, `right`), mentre si aggiunge `rotation: number` (radianti) e `angle: number` (gradi) per rendere disponibile la direzione effettiva del movimento, anche diagonale. Angolo in radianti da `atan2(velocityY, velocityX)` nelle coordinate schermo (0 destra, π/2 basso, π sinistra, -π/2 alto); quando il personaggio è fermo conserva l'ultimo angolo valido. Il developer può usare facing, angolo o entrambi per la propria view 2D/3D; Rpgoria non ruota la view automaticamente.


### API 31 — Facing durante le diagonali
Il `facing` cardinale viene determinato dall'asse della velocità effettiva predominante: se `abs(velocityX) > abs(velocityY)`, left/right secondo il segno di X; se `abs(velocityY) > abs(velocityX)`, up/down secondo il segno di Y. In caso di parità, conservare il facing precedente se compatibile con una delle due direzioni componenti del movimento. `rotation`/`angle` continua a esprimere la direzione diagonale precisa. Da specificare successivamente il fallback deterministico in parità quando il facing precedente non è compatibile.


### API 32 — Revisione di movementAngle
Decisione successiva che sostituisce il precedente comportamento nullo dell'API 30: `rotation`/`angle` è sempre `number` e mantiene l'ultimo angolo valido quando il movimento termina; non serve `facingAngle`. Alla conclusione del movimento `speed=0`, `isMoving=false`, `movementId=null`, mentre `rotation`/`angle` e `facing` restano quelli precedenti. Il valore iniziale per un personaggio mai mosso è ancora da confermare.


### API 33 — Nomi angolari come PixiJS (sostituisce API 30/32)
Il contratto pubblico `VisualState` NON espone più `movementAngle`: espone `rotation: number` in radianti e `angle: number` in gradi, due rappresentazioni della medesima direzione (`angle = rotation * 180 / Math.PI`). Entrambe persistono da fermo, conservando l'ultima direzione. La scelta se applicare tali valori alla view PixiJS/Spine/3D rimane esclusivamente del developer. I riferimenti storici alle API 30/32 vanno interpretati secondo questa decisione più recente. **API 34 confermata:** valore iniziale derivato automaticamente dal facing iniziale: right = 0 rad / 0°, down = π/2 rad / 90°, left = π rad / 180°, up = -π/2 rad / -90°. Disponibile già nel primo VisualState alla registrazione.


### API 35 — Nessuna interpolazione angolare nel core
`VisualState.rotation` (radianti) e `angle` (gradi) riflettono immediatamente la direzione effettiva del movimento, comprese le diagonali. Rpgoria non interpola la rotazione grafica: eventuale smoothing/turning animation via Motion o altre tecniche è responsabilità del developer sulla propria view. L'interpolazione della posizione rimane a carico di Rpgoria. Quando il personaggio è fermo, gli angoli persistono.


### API 36 — Aggiornamento visivo su cambio di facing da fermo
Quando cambia soltanto il facing senza spostamento, Rpgoria richiama immediatamente `onVisualUpdate(state)` con nuovo facing, `rotation` e `angle` coerenti, posizione invariata, `velocityX=0`, `velocityY=0`, `speed=0`, `isMoving=false`, `movementId=null`. Il callback non richiede un movimento attivo. L'evento pubblico `facingChanged` resta disponibile per logica di gioco; il developer decide come aggiornare la propria view.


### API 37 — Sincronizzazione visiva dopo teletrasporto
Dopo un teletrasporto riuscito Rpgoria richiama immediatamente `onVisualUpdate(state)` con nuova posizione non interpolata, `velocityX=0`, `velocityY=0`, `speed=0`, `isMoving=false`, `movementId=null`; facing, rotation e angle riflettono l'orientamento risultante. Nessun hook dedicato `onTeleported()` nella classe base. L'evento pubblico `teleported` rimane disponibile per effetti speciali gestiti dal developer.


### API 37 — Sincronizzazione visiva dopo teletrasporto
Dopo un teletrasporto riuscito, Rpgoria invoca immediatamente `onVisualUpdate(state)` con posizione finale non interpolata, `velocityX=0`, `velocityY=0`, `speed=0`, `isMoving=false`, `movementId=null`. Facing/rotation/angle vengono aggiornati se il teletrasporto specifica un orientamento, altrimenti persistono. Nessun hook `onTeleported()` obbligatorio nella classe base. L'evento pubblico `teleported` resta disponibile per effetti e logica di gioco.


### API 38 — Gestione errori di onVisualUpdate (decisione aperta)
Pixi'VN dispone di `Game.addOnError()` (documentazione: https://jsdocs.pixi-vn.com/pixi-vn/index/namespaces/Game/functions/addOnError), per registrare handler di errore. La registrazione degli handler NON implica che questa funzione possa essere usata da un modulo esterno per emettere errori. Prima dell'implementazione, Codex deve ispezionare il sorgente Pixi'VN e verificare se esiste un punto di ingresso pubblico/supportato per inoltrare errori al sistema centrale. Preferire il sistema di error handling di Pixi'VN, evitando duplicazioni; non introdurre `handlerError` in Rpgoria per gli errori di `onVisualUpdate()` senza ulteriore decisione. **API 40 confermata:** gli errori di callback visuale sono isolati e non interrompono movimento/collisioni; continuare a invocare il callback agli aggiornamenti successivi. Rimane da verificare soltanto il canale pubblico di segnalazione tramite Pixi'VN.


### API 39 — Contesto strutturato degli errori visuali
Decisione confermata: gli errori di `onVisualUpdate()` devono preservare `characterId`, `operation: "onVisualUpdate"` e `cause` (errore originale), per diagnosi e correlazione al personaggio. Integrarli con il sistema di error handling Pixi'VN solo se il sorgente conferma un punto di ingresso pubblico supportato; non inventare API di emissione. Non creare un sistema di error handling parallelo. L'isolamento è definito dall'API 40; resta da verificare nel sorgente Pixi'VN il canale pubblico di segnalazione.


### API 40 — Isolamento degli errori del callback visivo
Se `onVisualUpdate()` genera un errore, Rpgoria lo intercetta senza interrompere movimento, collisioni o altri personaggi. Conserva contesto strutturato (`characterId`, `operation`, `cause`) secondo API 39; ritenta normalmente il callback ai successivi aggiornamenti (nessuna disabilitazione permanente automatica). La view potrebbe essere temporaneamente non sincronizzata. Inoltrare al sistema errori Pixi'VN soltanto tramite un punto di ingresso pubblico verificato nel suo sorgente; non inventare API o un sistema parallelo.


### API 41 — Hook visivo strettamente sincrono
`CharacterMapStoredClass` espone `protected onVisualUpdate(state: VisualState): void` come callback sincrono. Rpgoria non attende Promise né operazioni di caricamento durante il ticker. Il developer prepara e carica le risorse grafiche separatamente e gestisce l'eventuale asincronia fuori dal callback. Le eccezioni sincrone del callback vengono isolate secondo API 40 e contestualizzate secondo API 39.


### API 42 — Rimozione dal registro e ownership della view
Decisione confermata: `RegisteredCharacterMapElements.remove(...)` mantiene firma/semantica del registro Pixi'VN (da verificare nel sorgente), ma la rimozione interrompe ogni successiva invocazione di `onVisualUpdate()` sull'elemento rimosso. Rpgoria non rimuove né distrugge automaticamente Sprite, Container, Spine, modelli 3D o altre risorse grafiche: gestione e cleanup della view rimangono esclusivamente al developer. Non introdurre `onRemoved()` nella classe base per questa finalità. Garantire che eventuali update già pianificati non richiamino un elemento deregistrato.


### API 43 — Deregistrazione durante il movimento
Decisione confermata: quando `RegisteredCharacterMapElements.remove(...)` deregistra un personaggio, Rpgoria annulla **immediatamente** qualunque movimento in corso, anche un passo grid non ancora completato, libera le prenotazioni di celle e rimuove il personaggio dalle collisioni. Ogni `moveTo()` pendente si risolve con `{ status: "cancelled", reason: "unregistered" }`. Aggiungere `"unregistered"` a `MoveToCancelReason`. Non invocare più `onVisualUpdate()` dopo la deregistrazione; nessun cleanup della view, che rimane del developer (API 42). Gestire in modo deterministico il caso di deregistrazione durante un callback/tick, evitando aggiornamenti successivi sul personaggio e risorse collisione residue. Precisare la precedenza del motivo `unregistered` rispetto a cancellazioni già avviate nella definizione finale della state machine.


### API 44 — Nuova registrazione con stato spaziale persistente
Decisione confermata: dopo `RegisteredCharacterMapElements.remove(...)`, una successiva `add(...)` di un elemento con lo stesso `characterId` recupera posizione e orientamento logici persistenti per ID (inclusi `facing`, `rotation`, `angle`). Non ripristina movimenti precedenti, `moveTo()` cancellati, velocità o prenotazioni di celle; il nuovo elemento parte fermo e riceve immediatamente `onVisualUpdate()` iniziale con posizione/orientamento recuperati, velocità e speed zero, `isMoving: false`, `movementId: null` (API 29). La deregistrazione libera gli ostacoli dinamici secondo API 43; la nuova registrazione ricrea la partecipazione alle collisioni solo secondo le regole correnti. Precisare successivamente la durata e l'eventuale cancellazione esplicita dello stato persistente per ID.


### API 45 — Persistenza dello stato spaziale nei salvataggi
Decisione confermata: lo stato spaziale logico dei personaggi (`characterId` → posizione e orientamento, inclusi `facing`, `rotation`, `angle`) persiste nei dati della partita e nei salvataggi, anche quando l'elemento è stato rimosso da `RegisteredCharacterMapElements`. La deregistrazione NON cancella tale stato; una successiva registrazione con lo stesso ID lo recupera (API 44). Lo stato si elimina soltanto al reset dei dati della partita o con un'API esplicita di cancellazione da definire. Non serializzare né riprendere automaticamente movimenti in corso, `moveTo()` pendenti, velocità o prenotazioni collisione per il solo fatto di registrare nuovamente l'elemento. Integrare con il meccanismo ufficiale di salvataggio/caricamento Pixi'VN, verificandone le API pubbliche nel sorgente prima dell'implementazione.


### API 46 — Cancellazione esplicita dello stato spaziale
Decisione confermata: esporre `rpgoria.characters.clearState(characterId)` per eliminare lo stato spaziale persistente associato all'ID (posizione e orientamento, inclusi `facing`, `rotation`, `angle`, nei dati di salvataggio). È consentito **soltanto se il personaggio non è registrato** in `RegisteredCharacterMapElements`: se ancora registrato, l'operazione viene rifiutata senza mutare lo stato. Il developer deve prima deregistrarlo (API 42–43). Non distruggere né modificare view o risorse grafiche. Una successiva registrazione dopo `clearState` usa i valori iniziali dell'elemento, non lo stato cancellato. La firma di ritorno/errore specifica resta da consolidare con le convenzioni Pixi'VN.


### API 47 — Idempotenza di clearState
Decisione confermata: `rpgoria.characters.clearState(characterId)` è idempotente se il personaggio non è registrato: elimina lo stato persistente quando presente, altrimenti termina normalmente senza errori. Se il personaggio è tuttora registrato in `RegisteredCharacterMapElements`, genera un errore e non modifica alcun dato, anche quando non esiste stato persistente. L'API deve essere sicura da richiamare ripetutamente dopo la deregistrazione; la firma precisa del valore di ritorno sarà consolidata separatamente.


### Revisione proposta dopo API 47 — Rimozione personaggio dalla mappa tramite handler del layer (dettagli da confermare)
Il developer propone di sostituire l'accesso pubblico a `rpgoria.characters.clearState(characterId)` con un'utility interna, richiamata da una nuova API pubblica per rimuovere un personaggio dalla mappa (nome provvisorio `rpgoria.characters.removeFromMap(characterId)`). Quando si registra un layer per personaggi, il developer deve fornire almeno un handler per la rimozione della view; la rimozione coordinata da Rpgoria invoca uno o più handler per staccare effettivamente la rappresentazione dal layer, senza imporre tipo di view né distruzione automatica. Da chiarire: se la rimozione dalla mappa debba sempre cancellare lo stato spaziale persistente oppure preservarlo per trasferimenti/cambi Room; semantica precisa della deregistrazione e dell'handler. Fino alla conferma, le API 45–48 e la precedente ownership della view vanno considerate oggetto di revisione, non implementare entrambe le interfacce pubbliche in conflitto.


### API 48 — REVISIONE CONFERMATA: rimozione definitiva dalla mappa
Decisione: esporre `rpgoria.characters.removeFromMap(characterId)` come operazione pubblica per rimuovere **definitivamente** il personaggio dalla mappa. Deve (1) interrompere immediatamente movimento e `moveTo()` pendenti; (2) liberare prenotazioni/collisioni; (3) invocare uno o più handler di rimozione della view, forniti obbligatoriamente quando si registra il layer dei personaggi; (4) deregistrare l'elemento e cessare `onVisualUpdate()`; (5) cancellare **sempre** lo stato spaziale persistente (posizione/orientamento) dai dati della partita. La funzione di cancellazione dello stato è **utility interna non accessibile al developer**: non esporre `rpgoria.characters.clearState()` come API pubblica. Questa revisione **sostituisce** le proposte di API 46 e 47 per l'accesso pubblico a `clearState` e prevale sulla proposta iniziale di API 48 relativa al tipo di ritorno di `clearState`. Le API 44–45 sulla persistenza in caso di semplice deregistrazione rimangono applicabili; `removeFromMap` è una cancellazione definitiva distinta. Rpgoria non assume tipo di view né chiama `destroy()` automaticamente; gli handler gestiscono la rimozione grafica. Da precisare in successive decisioni: contratto e molteplicità degli handler, registrazione del layer, gestione errori e atomicità dell'operazione.


### API 49 — Handler obbligatorio nella configurazione del layer dei personaggi
Decisione confermata: quando il developer registra/configura un **layer destinato ai personaggi**, deve fornire `onCharacterRemove` nella configurazione del layer stesso. La registrazione di un layer personaggi senza tale handler è invalida. Durante `rpgoria.characters.removeFromMap(characterId)` (API 48 revisionata), Rpgoria invoca l'handler associato al layer per rimuovere la view; il developer controlla come staccare Sprite/Container/Spine/altre rappresentazioni, senza `destroy()` automatico da parte di Rpgoria. La firma concreta dell'API di registrazione del layer e dell'handler è ancora da definire, e deve rispettare il sistema layer già stabilito; `registerCharacterLayer({ id, onCharacterRemove })` è solo un esempio concettuale, non una firma confermata. Non imporre handler di rimozione ai layer che non sono destinati ai personaggi.


### API 50 — Firma di onCharacterRemove: ID personaggio e oggetto layer
Decisione confermata dall'utente: l'handler obbligatorio `onCharacterRemove` del layer personaggi riceve **`characterId` e l'oggetto effettivo del layer** (`onCharacterRemove(characterId, layer)`), NON l'elemento `CharacterMapStoredClass` e NON il solo `layerId`. Il developer usa il layer e l'ID per trovare e staccare la rappresentazione grafica del personaggio. La tipizzazione esatta dell'oggetto layer va allineata alle API PixiJS/Rpgoria già definite; non assumere che sia una stringa. La gestione/distruzione della view rimane responsabilità del developer. La firma è sincrona o asincrona secondo una decisione successiva.


### API 51 — Handler di rimozione strettamente sincrono
Decisione confermata: `onCharacterRemove(characterId, layer): void` è **solo sincrono**. La rimozione della view dal layer deve avvenire durante la chiamata; Rpgoria non attende Promise o animazioni. `removeFromMap()` non deve attendere operazioni asincrone per completare la rimozione grafica. Eventuali effetti/animazioni di scomparsa sono orchestrati separatamente dal developer. Restano valide API 48–50: `removeFromMap` interrompe movimento, invoca handler del layer, deregistra e cancella lo stato persistente.


### API 52 — Errori degli handler di rimozione isolati
Decisione confermata: se un `onCharacterRemove(characterId, layer)` sincrono genera un'eccezione, Rpgoria la intercetta, la segnala attraverso il sistema errori Pixi'VN **solo se esiste un ingresso pubblico supportato verificato nel sorgente**, e **completa comunque** `removeFromMap()`: interrompe movimenti, libera prenotazioni/collisioni, deregistra il personaggio e cancella lo stato spaziale persistente. Se sono presenti più handler di rimozione applicabili, il fallimento di uno non impedisce l'esecuzione degli altri. La view dell'handler fallito potrebbe rimanere nel layer; Rpgoria non tenta di distruggerla. La diagnostica dell'errore deve conservare `characterId`, operazione `onCharacterRemove`, layer interessato e causa originale, senza inventare un nuovo canale di error handling parallelo. Non propagare l'errore al chiamante come fallimento dell'operazione logica.


### API 53 — Invocare gli handler di tutti i layer personaggi
Decisione confermata: `rpgoria.characters.removeFromMap(characterId)` invoca `onCharacterRemove(characterId, layer)` **su tutti i layer attualmente registrati come layer dei personaggi**, non soltanto sul layer principale o su quelli esplicitamente associati all'ID. Ogni handler decide autonomamente se il layer contiene una view del personaggio e la rimuove se presente; deve quindi tollerare l'assenza della view. Questo copre sprite, ombre e altri elementi distribuiti su layer differenti senza un registro personaggio→layer. Gli errori di un handler non impediscono di chiamare gli altri (API 52). L'ordine di esecuzione e le eventuali mutazioni dell'elenco dei layer durante la rimozione restano da definire.


### API 54 — Ordine degli handler di rimozione
Decisione confermata: durante `rpgoria.characters.removeFromMap(characterId)`, invocare gli handler sincroni `onCharacterRemove(characterId, layer)` di **tutti i layer dei personaggi registrati nell'ordine cronologico di registrazione dei layer** (dal primo all'ultimo). Nessuna priorità configurabile o inversione. Ogni handler deve tollerare l'assenza della view nel proprio layer. Gli errori sono isolati (API 52) e non impediscono l'esecuzione dei successivi. Il comportamento in caso di registrazione/rimozione di layer durante l'iterazione è da definire separatamente.


### API 55 — Snapshot dei layer all'inizio di removeFromMap
Decisione confermata: all'inizio di `rpgoria.characters.removeFromMap(characterId)` Rpgoria acquisisce uno **snapshot dei layer dei personaggi registrati e dei rispettivi handler `onCharacterRemove`**, nell'ordine cronologico di registrazione (API 54). Invoca tutti gli handler presenti nello snapshot anche se, durante l'operazione, un handler aggiunge o rimuove layer; tali mutazioni hanno effetto soltanto dalle successive chiamate a `removeFromMap`. Lo snapshot deve mantenere il riferimento all'oggetto layer originario per ogni callback (`onCharacterRemove(characterId, layer)`). Restano validi l'isolamento degli errori (API 52) e la rimozione logica definitiva (API 48 revisionata).


### API 56 — Idempotenza di removeFromMap
Decisione confermata: `rpgoria.characters.removeFromMap(characterId)` è **idempotente** quando il personaggio è già stato rimosso e non esiste più stato spaziale persistente per il suo `characterId`: termina normalmente, senza invocare di nuovo alcun `onCharacterRemove`, senza modificare layer/collisioni e senza generare errori per l'assenza. La prima rimozione esegue la sequenza completa delle API 48–55. Il caso distinto di personaggio non registrato ma con stato persistente ancora presente richiede una decisione esplicita successiva.


### API 57 — removeFromMap su personaggio deregistrato ma con stato persistente
Decisione confermata: se `rpgoria.characters.removeFromMap(characterId)` è chiamato quando il personaggio **non è registrato** in `RegisteredCharacterMapElements`, ma esiste ancora stato spaziale persistente per quell'ID, cancella **soltanto** posizione/orientamento salvati tramite l'utility interna e **non invoca** gli handler `onCharacterRemove` dei layer, né modifica view/layer/collisioni. Se invece è registrato, esegue la rimozione completa con handler (API 48–55). Se non è registrato e non esiste stato, è no-op idempotente (API 56). Queste tre condizioni definiscono il branching pubblico di `removeFromMap`.


### API 58 — Firma pubblica sincrona di removeFromMap
Decisione confermata: `rpgoria.characters.removeFromMap(characterId): void` è **sincrona**, senza Promise né valore di ritorno. Quando il metodo ritorna, Rpgoria ha eseguito gli handler `onCharacterRemove` applicabili (API 49–55), annullato immediatamente i movimenti, liberato le collisioni, deregistrato il personaggio se presente e cancellato il relativo stato spaziale persistente (API 48, 57). Rimane idempotente quando non esistono né registrazione né stato (API 56). Gli errori degli handler grafici sono isolati (API 52), senza compromettere la pulizia logica. Non esporre `clearState()` come API pubblica (revisione API 48).


### API 59 — Tipo esplicito dei layer personaggi
Decisione confermata: nella configurazione dei layer, un layer destinato ai personaggi si identifica esplicitamente tramite `type: "characters"`. Per tale tipo, `onCharacterRemove(characterId, layer): void` è **obbligatorio** a livello TypeScript (union discriminata o equivalente) e deve essere validato anche a runtime; la configurazione priva dell'handler non è valida. I layer normali non sono obbligati a fornire questo handler. `removeFromMap` richiama gli handler di tutti i layer con `type: "characters"` secondo API 53–55. Distinguere questo `type` di ruolo dai comportamenti preesistenti `static`/`movement` per composizione e ordinamento Y: la loro compatibilità/configurazione precisa va decisa separatamente, senza perdere le funzionalità già confermate.


### API 60 — sortMode indipendente con default condizionale
Decisione confermata: `type` identifica il ruolo del layer (es. `"characters"`, con `onCharacterRemove` obbligatorio secondo API 59), mentre `sortMode?: "y" | "none"` configura indipendentemente l'ordinamento. **Se `sortMode` è omesso**: `type: "characters"` → default `"y"`; per tutti gli altri tipi → default `"none"`. **Se `sortMode` è specificato, prevale sempre** sul default del tipo, anche `characters` con `sortMode: "none"` o layer normale con `sortMode: "y"`. Il Y-sort non è riservato ai personaggi. Separare concettualmente la classificazione del layer dal comportamento di ordinamento, preservando le capacità già decise di ordinamento indipendente dei layer e degli oggetti dinamici. Esempi: `{ type: "characters" }` → y; `{ type: "characters", sortMode: "none" }` → none; `{ type: "static", sortMode: "y" }` → y.


### API 61 — Container condiviso dei personaggi e documentazione wiki
Decisione dell'utente: le rappresentazioni dei personaggi in uno stesso layer dei personaggi devono essere **figli diretti del medesimo Container** del layer, per semplificare il Y-sort. È ammesso che ciascun personaggio sia rappresentato da un Container con figli interni (Sprite, Spine, ecc.), ma il nodo principale del personaggio deve essere figlio diretto del Container condiviso. Il Y-sort può così utilizzare la coordinata `y` locale del nodo principale nel layer, senza attraversare trasformazioni di gerarchie genitore differenti. Questa convenzione e i relativi esempi di struttura e ancoraggio al punto base/piedi devono essere **spiegati chiaramente nella wiki**. Rimane valida la priorità dei valori espliciti di profondità già stabilita; questa decisione semplifica soltanto il fallback automatico. Per layer non destinati ai personaggi, l'applicazione della medesima convenzione va chiarita separatamente.


### API 62 — Container condiviso obbligatorio per ogni layer Y-sort
Decisione confermata: **qualsiasi** layer con `sortMode: "y"`, indipendentemente dal `type`, richiede che **tutti gli oggetti ordinabili siano figli diretti del Container condiviso di quel layer**. Il fallback automatico di profondità usa la coordinata `y` locale del nodo principale, senza calcoli di trasformazioni attraverso gerarchie esterne. Un nodo ordinabile può essere un Container con figli annidati: è ordinato come singola unità. Rimane valida la priorità dei valori espliciti di profondità già decisa. La wiki deve spiegare la convenzione per layer personaggi e non, con esempi corretti/errati e indicazioni sul punto base/ancoraggio. Questa decisione estende e chiarisce API 61, senza limitare `sortMode: "y"` ai layer `characters` (API 60).


### API 63 — Y-sort solo dei figli diretti, senza correzioni o errori
Decisione confermata: per tutti i layer con `sortMode: "y"`, Rpgoria **ordina soltanto i figli diretti** del Container condiviso del layer. Non attraversa Container annidati per trovare oggetti ordinabili, non genera errori se il developer usa una struttura diversa e non sposta/riparenta automaticamente gli oggetti. Un Container figlio diretto può contenere la gerarchia grafica interna di un personaggio/oggetto ed è ordinato come **una sola unità**. La responsabilità della corretta struttura è del developer e la wiki deve documentarla chiaramente con esempi. Estende API 61–62.


### API 64 — Y-sort a ogni frame del ticker attivo
Decisione confermata: per ogni layer con `sortMode: "y"`, Rpgoria esegue l'ordinamento **a ogni frame in cui il ticker condiviso è attivo**, anche quando nessun personaggio sta compiendo un movimento gestito da Rpgoria. Questo include oggetti spostati/animati direttamente dal developer. Non implica che il ticker debba restare sempre acceso: rispettare il ciclo di vita già deciso (ticker attivo quando richiesto da movimenti o sortable dinamici; modalità manuale secondo configurazione). Ordinare soltanto i figli diretti del Container del layer (API 62–63), usando la priorità di profondità esplicita già stabilita e la `y` locale come fallback. Il comportamento quando il ticker è inattivo e la possibilità di un ordinamento esplicito restano da precisare.


### API 65 — Ticker mantenuto attivo dagli oggetti ordinabili dinamici
Decisione confermata: in modalità `updateMode: "auto"`, il ticker condiviso di Rpgoria rimane attivo se **almeno un layer con `sortMode: "y"` contiene oggetti ordinabili dichiarati dinamici**, anche quando non ci sono movimenti gestiti da Rpgoria. Gli oggetti statici non richiedono da soli aggiornamenti continui. Se non ci sono movimenti né oggetti dinamici che necessitano di Y-sort, il ticker può fermarsi. A ogni frame attivo il Y-sort viene eseguito secondo API 64. In `updateMode: "manual"` il developer resta responsabile degli aggiornamenti. Resta da definire come un oggetto viene dichiarato dinamico e come cambia tale stato.


### API 66 — Oggetti Y-sort inseriti direttamente nel layer, senza registrazione
Chiarimento dell'utente: il developer **aggiunge direttamente** gli oggetti al Container del layer (es. `layer.addChild(view)`). Se si trovano come figli diretti in un layer con `sortMode: "y"`, sono ordinati automaticamente: **nessuna registrazione esplicita** `registerDynamic`, nessuna proprietà `rpgoriaDynamic`, nessuna dichiarazione per-oggetto necessaria per partecipare allo Y-sort. Si applica a tutti i figli diretti del layer (API 62–64). **Questione aperta:** la precedente API 65 parla di oggetti «dichiarati dinamici» per tenere acceso il ticker; senza dichiarazione per-oggetto occorre chiarire quale condizione lo mantenga attivo in assenza di movimento. Non assumere un criterio implicito senza ulteriore decisione.


### API 67 — Ticker attivo se un layer Y-sort contiene figli diretti (sostituisce API 65)
Decisione confermata: in `updateMode: "auto"`, il ticker condiviso resta attivo se **almeno un layer con `sortMode: "y"` contiene uno o più figli diretti**, anche senza movimento gestito da Rpgoria. Un layer Y-sort vuoto non mantiene attivo il ticker. Se non ci sono movimenti né layer Y-sort con figli diretti, il ticker può fermarsi. Durante ogni frame attivo si ordinano tutti i figli diretti dei layer Y-sort (API 62–64). **Questa decisione sostituisce esplicitamente il criterio dell'API 65** basato su oggetti «dichiarati dinamici»: non occorre alcuna dichiarazione, registrazione o proprietà per-oggetto (API 66). In `updateMode: "manual"`, il developer controlla gli aggiornamenti. Il controllo della presenza dei figli diretti deve consentire avvio/arresto del ticker quando vengono aggiunti o rimossi, senza richiedere registrazione per-oggetto.


### API 68 — Ticker condiviso sempre attivo in modalità auto (revisione API 65 e 67)
L'utente ha cambiato la decisione: in `updateMode: "auto"` il **ticker condiviso di Rpgoria gira continuamente, anche a vuoto**, quando non ci sono movimenti né oggetti da ordinare. Non occorre rilevare `addChild`/`removeChild`, registrare oggetti, mantenere un ticker di controllo o richiedere `layers.refresh()`. I layer con `sortMode: "y"` ordinano i loro figli diretti a ogni frame (API 62–64, 66), anche se aggiunti dinamicamente. **Questa decisione prevale sulle condizioni di avvio/arresto del ticker definite in precedenza, incluse API 65 e 67 e la precedente scelta di arrestarlo quando inattivo**. In `updateMode: "manual"` rimane il controllo esterno del developer. La scelta comporta un costo continuo del ticker accettato consapevolmente per semplicità.


### API 69 — Ordinamento Y stabile in caso di parità
Decisione confermata: per tutti i layer con `sortMode: "y"`, quando due oggetti hanno la **stessa profondità effettiva**, lo Y-sort è **stabile** e conserva il loro ordine relativo precedente nel Container. Non introdurre una proprietà `sortPriority` come criterio secondario e non risolvere le parità con identificatori interni indipendenti dall'ordine preesistente. Resta invariata la precedenza già decisa per i valori di profondità espliciti rispetto alla `y` locale automatica. La stabilità deve valere tra frame successivi e per i figli diretti ordinabili (API 62–64). In modalità `auto`, il ticker rimane sempre attivo secondo API 68.


### API 70 — Y-sort include gli oggetti nascosti
Decisione confermata: tutti i figli diretti di un layer con `sortMode: "y"` partecipano all'ordinamento **anche quando `visible === false`**. La visibilità non altera il valore di profondità né esclude l'oggetto dal sort; quando torna visibile, è già ordinato correttamente. Non introdurre opzioni di inclusione/esclusione per visibilità. Resta stabile l'ordinamento a parità di profondità (API 69), con aggiornamento a ogni frame del ticker in modalità auto (API 64, 68) e soli figli diretti (API 62–63).


### API 71 — Y-sort Rpgoria prioritario rispetto a PixiJS zIndex
Decisione confermata: nei layer con `sortMode: "y"`, l'ordine visivo dei figli diretti è determinato **esclusivamente dall'ordinamento per profondità Y effettiva di Rpgoria** (con le precedenze esplicite già definite), non da `DisplayObject.zIndex` di PixiJS. `zIndex` non è né criterio primario, né secondario, né sostituto della profondità Y. L'implementazione deve impedire che un eventuale ordinamento automatico di PixiJS basato su `zIndex` sovrascriva il risultato di Rpgoria, usando le API pubbliche effettivamente supportate dalla versione PixiJS adottata. Rimangono l'ordinamento stabile a parità di profondità (API 69), l'inclusione dei figli invisibili (API 70) e il vincolo dei soli figli diretti (API 62–63). Documentare la precedenza nella wiki.


### API 72 — Fallback per profondità Y non finita
Decisione confermata: durante lo Y-sort, i valori di profondità non finiti (`NaN`, `Infinity`, `-Infinity`) non devono compromettere l'ordinamento. Rispettando la precedenza delle fonti di profondità già stabilita, utilizzare una profondità esplicita soltanto se `Number.isFinite(value)`; altrimenti provare la `y` locale del figlio diretto se finita; se anch'essa non è finita, usare `0` come fallback deterministico. Nessun errore di configurazione per questo caso. Rimane l'ordinamento stabile in caso di parità (API 69), anche per oggetti invisibili (API 70); `zIndex` non partecipa (API 71).


### API 73 — Profondità Y negative valide e ordinamento crescente
Decisione confermata: le profondità Y **negative e finite** sono perfettamente valide; lo Y-sort ordina per valore numerico **crescente naturale**, per esempio `-100, -20, 0, 50, 200`. Non applicare clamp/normalizzazione a zero né introdurre un'opzione per gestire i valori negativi. L'API 72 applica fallback solo ai valori **non finiti** (`NaN`, `Infinity`, `-Infinity`), non ai valori negativi finiti. Restano valide la stabilità in caso di parità (API 69) e l'esclusione di `zIndex` come criterio (API 71).


### API 74 — Verifica ordine prima di modificare il Container
Decisione confermata: a **ogni frame** del ticker (API 64, 68), per ciascun layer con `sortMode: "y"`, Rpgoria **verifica se i figli diretti sono già ordinati** per profondità effettiva crescente. Se lo sono, **non modifica** l'ordine/gerarchia del Container e non esegue riordinamenti superflui; se non lo sono, riordina. In caso di profondità uguali preserva l'ordine relativo precedente (sort stabile, API 69). Il controllo deve considerare valori espliciti di profondità, fallback per valori non finiti (API 72), valori negativi validi (API 73), figli invisibili (API 70), e ignorare `zIndex` (API 71). Non è richiesta una cache delle profondità tra frame; l'obiettivo è evitare mutazioni del Container quando l'ordine è già corretto.


### API 75 — Ordine di aggiornamento nel frame: movimento → visual → Y-sort
Decisione confermata: in ogni aggiornamento/frame di Rpgoria, sia con `updateMode: "auto"` sia con `"manual"`, l'ordine garantito è: **(1)** aggiornare movimento e posizioni logiche/visive; **(2)** invocare i callback sincroni `onVisualUpdate(state)` applicabili, così che il developer aggiorni le view; **(3)** eseguire il controllo e, se necessario, il riordino Y dei figli diretti di tutti i layer con `sortMode: "y"`. Lo Y-sort deve quindi osservare le coordinate grafiche aggiornate nello **stesso frame**, senza un frame di ritardo. Rispettare l'isolamento degli errori di `onVisualUpdate` (API 39–41), senza impedire il successivo Y-sort. Resta la verifica dell'ordine prima di mutare il Container (API 74) e il ticker sempre attivo in auto (API 68).


### API 76 — Y-sort sui figli presenti al momento del sort
Decisione confermata: dopo la fase `onVisualUpdate()` prevista dall'API 75, lo Y-sort **legge i figli diretti attuali** del Container di ciascun layer con `sortMode: "y"`, al momento effettivo del sort. I figli aggiunti durante `onVisualUpdate()` partecipano **nello stesso frame**; quelli rimossi prima del sort non partecipano. **Nessuno snapshot dei figli a inizio frame**, nessun differimento automatico di aggiunte/rimozioni. Restano il vincolo dei soli figli diretti (API 62–63), la verifica prima di mutare l'ordine (API 74), la stabilità (API 69) e l'ordine movimento → callback visivi → sort (API 75).


### API 77 — Snapshot dei figli all'inizio del singolo Y-sort
Decisione confermata: per ciascun layer con `sortMode: "y"`, **all'inizio della singola operazione di Y-sort**, Rpgoria acquisisce uno snapshot dei figli diretti presenti in quel momento (dopo `onVisualUpdate`, secondo API 75–76). Durante il riordinamento, prima di operare su ciascun oggetto, verifica che **appartenga ancora al Container**; non reinserisce oggetti rimossi. I figli aggiunti **dopo lo snapshot del sort** saranno ordinati nel frame successivo. Non iterare direttamente su una lista live soggetta a mutazioni e non ripetere indefinitamente il sort in caso di modifiche strutturali. Lo snapshot è **per sort**, non a inizio frame, dunque include le aggiunte avvenute durante `onVisualUpdate` (API 76). Conservare ordinamento stabile e controllo prima delle mutazioni (API 69, 74).


### API 78 — Profondità effettive acquisite una sola volta per singolo Y-sort
Decisione confermata: all'inizio di ogni singola operazione Y-sort di un layer, Rpgoria (1) acquisisce lo **snapshot dei figli diretti** secondo API 77 e (2) **calcola una sola volta** la profondità effettiva di ciascun figlio nello snapshot, rispettando le priorità esplicite, il fallback per valori non finiti (API 72) e le profondità negative valide (API 73). Per tutta la durata di quel sort usa esclusivamente i valori acquisiti, **senza rileggere le profondità durante i confronti**. Cambiamenti di profondità successivi allo snapshot sono considerati nel **frame successivo**, senza secondo sort nello stesso frame. Rimane obbligatorio verificare l'appartenenza al Container prima di riordinare un oggetto (API 77), preservare la stabilità a parità di profondità (API 69) ed evitare mutazioni se già ordinato (API 74).


### API 79 — Ordine di esecuzione dello Y-sort tra layer
Decisione confermata: quando più layer hanno `sortMode: "y"`, Rpgoria esegue il rispettivo Y-sort **nell'ordine di registrazione dei layer**. Ciascun layer è ordinato indipendentemente con il proprio snapshot dei figli e delle profondità (API 77–78). L'ordine di esecuzione non dipende dalla posizione corrente dei layer in `rpgoria.layers.container`, né modifica l'ordine di rendering/composizione dei layer scelto dal developer. Mantenere l'ordine di registrazione deterministico.


### API 80 — Snapshot dei layer a inizio fase Y-sort
Decisione confermata: **all'inizio della fase Y-sort di ogni frame**, Rpgoria acquisisce uno snapshot dei layer registrati con `sortMode: "y"`, nell'ordine di registrazione (API 79). Prima di ordinare ciascun layer nello snapshot, verifica che sia **ancora registrato**; se è stato deregistrato durante la fase, lo salta. I layer aggiunti dopo lo snapshot vengono ordinati **dal frame successivo**. Le operazioni di registrazione/rimozione dei layer hanno effetto immediato sul registro; non vengono differite, ma la fase corrente usa lo snapshot. Per ciascun layer ancora valido si acquisiscono poi gli snapshot di figli e profondità al momento del suo singolo sort (API 77–78), dopo `onVisualUpdate` (API 75–76).


### API 81 — Nuova identità per layer deregistrato e registrato di nuovo
Decisione confermata: ogni **operazione di registrazione** di un layer ha una propria identità interna distinta, anche quando il developer registra nuovamente lo **stesso oggetto layer PixiJS**. Se un layer viene deregistrato e registrato di nuovo durante la medesima fase Y-sort, la voce della **vecchia registrazione** nello snapshot iniziale (API 80) è invalida e va saltata, anche se il medesimo oggetto è di nuovo presente nel registro. La **nuova registrazione** partecipa allo Y-sort soltanto dal **frame successivo**, nella sua nuova posizione nell'ordine di registrazione (API 79). Nessun errore per questa sequenza; non basta verificare la sola identità dell'oggetto layer, occorre verificare l'identità della specifica registrazione.


### API 82 — Registrazione duplicata dello stesso layer vietata
Decisione confermata: lo **stesso oggetto layer PixiJS** può avere **una sola registrazione attiva** in Rpgoria. Un tentativo di registrare nuovamente tale oggetto mentre la registrazione precedente è ancora attiva **genera un errore** e non modifica la registrazione esistente: identità interna, configurazione, handler e ordine di registrazione rimangono invariati. Non trattare il duplicato come operazione idempotente e non sostituire automaticamente la registrazione. È invece lecito deregistrare prima il layer e registrare successivamente lo stesso oggetto con una **nuova identità** (API 81); durante la fase Y-sort questa nuova registrazione partecipa dal frame successivo.


### API 83 — Deregistrazione layer esclusivamente logica
Decisione confermata: la deregistrazione di un layer rimuove **solo la registrazione logica** in Rpgoria e interrompe la gestione del layer (incluso il suo Y-sort); **non** rimuove o riposiziona il Container PixiJS del layer in `rpgoria.layers.container` o altrove nella gerarchia, **non** distrugge il layer e **non** distrugge né modifica i suoi figli. Il developer resta responsabile della gestione della scena PixiJS. È consentito registrare nuovamente lo stesso oggetto layer dopo la deregistrazione, con nuova identità (API 81); una seconda registrazione mentre è ancora attivo resta un errore (API 82).


### API 84 — Deregistrazione di layer inesistente idempotente
Decisione confermata: tentare di deregistrare un layer **mai registrato** o **già deregistrato** è un **no-op senza errori**. Non modifica alcuno stato, né la gerarchia o gli oggetti PixiJS. La deregistrazione ripetuta è sicura. Quando invece esiste una registrazione attiva, si applica la sola rimozione logica stabilita dall'API 83; non è richiesta una segnalazione di errore per registrazione assente.


### API 85 — Deregistrazione tramite ID stringa univoco
Decisione confermata: ogni layer registrato in Rpgoria deve avere un **ID pubblico di tipo stringa univoco tra le registrazioni attive**. L'API pubblica di deregistrazione accetta **l'ID stringa**, non il riferimento all'oggetto layer PixiJS. Due layer attivi non possono condividere lo stesso ID. Deregistrare un ID non presente nel registro è un **no-op senza errori** (API 84); la deregistrazione rimane esclusivamente logica, senza modificare la gerarchia PixiJS o distruggere oggetti (API 83). L'ID pubblico del layer è distinto dall'**identità interna della specifica registrazione** usata per validare gli snapshot Y-sort (API 81): anche riutilizzando un ID dopo deregistrazione, la nuova registrazione deve avere nuova identità interna. Resta vietata una doppia registrazione attiva dello stesso oggetto layer (API 82).


### API 86 — ID pubblico duplicato: errore senza modifiche
Decisione confermata: tentare di registrare un layer usando un **ID stringa già associato a una registrazione attiva** genera un **errore** e non modifica la registrazione preesistente, anche quando il nuovo layer è un oggetto PixiJS differente. Devono restare invariati oggetto, configurazione, handler, identità interna della registrazione e posizione nell'ordine di registrazione. Non sostituire automaticamente il layer e non ignorare silenziosamente il conflitto. Un ID torna disponibile dopo la deregistrazione del layer corrispondente (API 85); il suo riutilizzo costituisce una **nuova registrazione** con nuova identità interna (API 81). Resta inoltre vietata la doppia registrazione attiva dello stesso oggetto layer anche sotto ID diversi (API 82).


### API 87 (revisione confermata) — Ownership dei layer delegata a Pixi'VN; rettifica API 79–86
**Decisione prevalente e vincolante:** la gestione pubblica dei layer (ID, inserimento nella scena, recupero, rimozione e gerarchia) appartiene **esclusivamente a Pixi'VN**. Rpgoria **crea** i Container dei layer RPG e li **aggiunge automaticamente** tramite l'API ufficiale Pixi'VN. Firma pubblica concettuale confermata: `rpgoria.layers.create(id, options)`, che crea un Container, lo configura secondo le opzioni RPG (inclusi `type`, `sortMode` e `onCharacterRemove` quando richiesto), chiama `canvas.layers.add(id, container)` e restituisce il Container creato. Verificare firme, tipi e semantica reali di `canvas.layers.add` nei sorgenti Pixi'VN prima di implementare. Non introdurre `rpgoria.layers.get(id)`, `rpgoria.layers.remove(id)`, `rpgoria.layers.unregister(id)`, né un secondo registry pubblico dei layer. Il developer usa **`canvas.layers.get(id)`, `canvas.layers.remove(id)` e le altre API Pixi'VN** per ogni gestione successiva e per la gerarchia. Non implementare la precedente proposta di root obbligatorio `rpgoria.layers.container` da inserire manualmente: ogni layer è aggiunto tramite Pixi'VN.

**Rettifiche esplicite alle API precedenti, per evitare implementazioni contraddittorie:**
- **API 79:** resta l'ordine deterministico di creazione/aggiunta dei layer RPG per l'esecuzione dello Y-sort, ma **non** implica un registry pubblico Rpgoria né un root `rpgoria.layers.container`. La gerarchia e l'ordine visivo tra layer restano di Pixi'VN.
- **API 80–81:** restano obiettivi di snapshot, validità e identità di un'istanza di gestione Y-sort, **solo come meccanismi interni eventualmente necessari**, subordinati al ciclo di vita reale Pixi'VN. Non richiedere al developer registrazione/deregistrazione separata in Rpgoria. Una rimozione Pixi'VN non deve lasciare un layer fantasma gestito da Rpgoria. Verificare come osservare l'effettiva rimozione/riaggiunta attraverso le API pubbliche Pixi'VN, senza inventare hook.
- **API 82:** la regola di errore per doppia registrazione attiva dello stesso oggetto **non costituisce un'API pubblica di registrazione Rpgoria**. La creazione via `create` produce un nuovo Container; i conflitti devono rispettare la semantica Pixi'VN, senza un registro pubblico duplicato.
- **API 83–85:** **superate nella parte che definisce una deregistrazione pubblica Rpgoria, logica/idempotente o per ID**. La rimozione è `canvas.layers.remove(id)` e ne segue la semantica effettiva di Pixi'VN (inclusa l'eventuale modifica della gerarchia e l'eventuale comportamento su ID assenti); Rpgoria non impone che sia no-op, che non modifichi la gerarchia o che non distrugga oggetti. L'ID è quello passato a `canvas.layers.add`, non un secondo namespace.
- **API 86:** l'univocità degli ID e i conflitti vanno **delegati alle API Pixi'VN**. Non introdurre un errore di duplicato distinto e potenzialmente incompatibile in Rpgoria; verificare il comportamento Pixi'VN e propagare gli errori della creazione/aggiunta senza lasciare una registrazione interna orfana.
- **API 59–60 e 49–55:** restano validi ruolo, opzioni RPG, Y-sort e handler `onCharacterRemove`; l'associazione di tali comportamenti al layer creato deve restare interna e coerente con il suo ciclo di vita Pixi'VN. Il developer continua a creare e aggiungere manualmente le **view dei personaggi**, non i Container layer RPG.

**Punti tecnici da verificare, non nuove decisioni:** come rilevare attraverso le API effettive Pixi'VN la rimozione di un layer per disattivare Y-sort/handler, e come trattare la riaggiunta dello stesso Container tramite Pixi'VN senza ricreazione Rpgoria. Non assumere che Pixi'VN esponga eventi di lifecycle prima di ispezionare i sorgenti. **Nessuna modifica al codice sorgente autorizzata da questa decisione.**


### API 88 — Validazione tramite Pixi'VN prima di ogni Y-sort di layer
Decisione confermata: **prima di ordinare ogni layer RPG con `sortMode: "y"`**, Rpgoria consulta **`canvas.layers.get(id)`** e verifica che il risultato sia **lo stesso oggetto Container PixiJS creato e aggiunto da Rpgoria** con `rpgoria.layers.create(id, options)` (API 87 revisionata). Se il layer non esiste più in Pixi'VN, oppure lo stesso ID identifica ora un Container differente, **saltare lo Y-sort** del vecchio Container. Non è sufficiente controllare la presenza dell'ID: confrontare l'identità dell'oggetto. Il controllo è eseguito al momento del turno di ciascun layer nello snapshot della fase Y-sort (API 79–80), non soltanto all'inizio del frame. La gestione pubblica di aggiunta/rimozione/gerarchia resta esclusivamente Pixi'VN; non introdurre una API pubblica di deregistrazione Rpgoria. Verificare la firma e il comportamento reali di `canvas.layers.get` nei sorgenti Pixi'VN prima dell'implementazione.


### API 89 — Ripresa automatica dopo riaggiunta dello stesso Container tramite Pixi'VN
Decisione confermata: se un layer creato con `rpgoria.layers.create(id, options)` viene rimosso da Pixi'VN mediante `canvas.layers.remove(id)` e poi **riaggiunto tramite `canvas.layers.add(id, sameContainer)` usando lo stesso ID e la stessa istanza di Container PixiJS**, Rpgoria **riprende automaticamente** lo Y-sort e le altre responsabilità RPG associate al layer, senza richiedere una nuova creazione, registrazione o chiamata esplicita di riattivazione. La configurazione RPG originale (`type`, `sortMode`, `onCharacterRemove` se applicabile) rimane associata al Container creato. Quando `canvas.layers.get(id)` non restituisce esattamente quel Container, il suo Y-sort viene saltato (API 88); quando torna a restituirlo, il sort riprende automaticamente. La semplice assenza temporanea in Pixi'VN **non cancella** l'associazione interna di Rpgoria. Non introdurre un lifecycle pubblico parallelo a Pixi'VN (API 87 revisionata). La validazione dell'identità è effettuata al momento del singolo sort (API 88).


### API 90 — Handler onCharacterRemove solo per layer presenti in Pixi'VN
Decisione confermata: durante `rpgoria.characters.removeFromMap(characterId)`, considerare per l'invocazione di `onCharacterRemove(characterId, layer)` **solo i layer RPG con `type: "characters"` che risultano attualmente presenti in Pixi'VN con identità corrispondente**, ossia per cui `canvas.layers.get(id) === containerCreatoDaRpgoria`. Layer rimossi da Pixi'VN o sostituiti da un Container diverso sotto lo stesso ID **non partecipano** agli handler di quella chiamata. Integrare con API 54–55: **all'inizio della chiamata**, costruire lo snapshot dei layer personaggi *idonei* e dei rispettivi handler nell'ordine di creazione/registrazione RPG; dopo lo snapshot, eseguire tutti gli handler acquisiti anche se la presenza dei layer in Pixi'VN cambia durante l'esecuzione. Gli errori rimangono isolati (API 52) e la pulizia logica definitiva del personaggio è sempre completata (API 48, 58). Un layer originale riaggiunto in Pixi'VN torna idoneo alle chiamate successive (API 89). Nessuna API pubblica parallela di registrazione/rimozione layer in Rpgoria (API 87 revisionata).


### API 91 — Creazione atomica del layer tramite Pixi'VN
Decisione confermata: `rpgoria.layers.create(id, options)` deve essere **atomica rispetto alla riuscita dell'aggiunta tramite Pixi'VN**. Rpgoria crea il Container e chiama `canvas.layers.add(id, container)`; **solo se l'aggiunta riesce** conserva internamente la configurazione e i riferimenti necessari a Y-sort e `onCharacterRemove`, e restituisce il Container. Se `canvas.layers.add` fallisce (per esempio per un ID occupato, se questa è la semantica effettiva della versione Pixi'VN), Rpgoria **propaga l'errore**, non conserva alcuna registrazione/configurazione interna del nuovo layer e non tenta rimozioni, sostituzioni o recuperi automatici. Non alterare il layer Pixi'VN preesistente. Rispettare le semantiche reali di `canvas.layers.add` dopo ispezione dei sorgenti Pixi'VN (API 87). Questa decisione rafforza la gestione dei conflitti delegata a Pixi'VN e prevale su qualsiasi proposta di registrazione preventiva interna o sostituzione implicita.


### API 92 — Container del layer sotto il controllo del developer
Decisione confermata: il Container PixiJS creato e restituito da `rpgoria.layers.create(id, options)` resta **interamente controllabile dal developer** tramite le normali API PixiJS/Pixi'VN. Rpgoria modifica esclusivamente ciò che è necessario per i **comportamenti RPG esplicitamente configurati**, in particolare l'ordine dei figli diretti quando `sortMode: "y"` richiede un riordinamento (API 62–78), senza imporre o ripristinare valori di `visible`, posizione, trasformazioni, filtri o contenuti grafici. Il developer può aggiungere/rimuovere i figli direttamente con `layer.addChild(...)` e le altre API PixiJS, e controlla la gerarchia interna delle view. Rpgoria non protegge né blocca le modifiche dirette al Container, non introduce una modalità di proprietà protette o opzioni aggiuntive per controllarle. Restano validi l'ordinamento Y autorevole rispetto a `zIndex` (API 71) e la necessità di impedire che l'ordinamento nativo PixiJS sovrascriva lo Y-sort, limitatamente a quanto indispensabile per questo comportamento. La gestione pubblica dei layer resta di Pixi'VN (API 87–91).


### API 93 — Profondità esplicita opzionale `rpgDepth`, altrimenti `y`
Decisione dell'utente confermata: per ogni figlio diretto di un layer con `sortMode: "y"`, **se l'oggetto ha una proprietà `rpgDepth` con valore numerico finito, usare tale valore come profondità effettiva; altrimenti usare la sua coordinata locale `y`**. Se anche `y` non è finita, profondità effettiva `0` (API 72). Esempio concettuale: `const depth = Number.isFinite(view.rpgDepth) ? view.rpgDepth : Number.isFinite(view.y) ? view.y : 0`. La proprietà `rpgDepth` è **opzionale**; non richiedere che gli oggetti estendano una classe Rpgoria, non introdurre `getDepth` obbligatoria per layer né registrazione per singolo oggetto. Definire una tipizzazione TypeScript appropriata senza imporre augmentation globale non concordata. Continuano a valere profondità negative (API 73), ordine crescente stabile (API 69), snapshot delle profondità una sola volta per singolo sort (API 78), esclusione di `zIndex` (API 71) e Y-sort solo dei figli diretti (API 62–63). Questa decisione specifica la fonte di profondità esplicita precedentemente lasciata astratta.


### API 94 — Estensione TypeScript dei tipi PixiJS con `rpgDepth?`
Decisione confermata: Rpgoria deve fornire una **estensione dei tipi TypeScript PixiJS pertinenti** (module augmentation o meccanismo compatibile con la versione effettivamente adottata) che esponga la proprietà opzionale `rpgDepth?: number` sugli oggetti grafici ordinabili. Il developer deve poter scrivere direttamente `view.rpgDepth = 120` **senza cast, helper o classi base obbligatorie**. Verificare nei sorgenti e nelle dichiarazioni della versione PixiJS usata quale tipo/interfaccia sia realmente augmentabile, senza assumere nomi di classi o moduli non verificati. L'estensione è solo di tipizzazione: non assegna `rpgDepth` a runtime, non cambia automaticamente il comportamento di PixiJS e non modifica la proprietà `y`. Durante lo Y-sort, Rpgoria usa `rpgDepth` solo se numerico finito, altrimenti `y` finita, altrimenti `0` (API 93 e 72); ignorare `zIndex` (API 71). Non introdurre un helper `setRpgDepth` come API pubblica necessaria.


### API 95 (revisione confermata) — Persistenza di elementi e layer tramite Memory/setMemory di Pixi'VN
Decisione dell'utente: **ogni tipologia di elemento grafico RPG deve essere dichiarata tramite un'API analoga ai custom components di Pixi'VN** (riferimento: https://pixi-vn.com/it/start/canvas-components#custom-components), usando i contratti **`Memory` e `setMemory()`** per serializzare e ripristinare il proprio stato. Questo include `rpgDepth` quando presente/necessario, senza introdurre un salvataggio speciale e parallelo della singola proprietà. La decisione **sostituisce la precedente domanda API 95** che contrapponeva salvataggio automatico di `rpgDepth` e sola proprietà grafica non persistita.

**Anche lo stato di ogni layer RPG deve essere salvato e ripristinato attraverso il sistema di Pixi'VN**, coerentemente con API 87–94: Pixi'VN resta proprietario della gestione pubblica dei layer e della loro integrazione nel canvas. Se Pixi'VN non dispone già delle utility necessarie per persistere/ricostruire layer e relativi elementi, **progettare una utility nella libreria Pixi'VN**, da utilizzare in Rpgoria; **non creare un sistema autonomo di serializzazione dei layer in Rpgoria**. Verificare prima la documentazione e soprattutto i sorgenti effettivi di Pixi'VN: firme reali di `Memory`/`setMemory()`, registrazione dei custom components, serializzazione dei Container/layer, ricostruzione della gerarchia, salvataggio e caricamento. Non assumere l'assenza di una utility senza ispezione.

Lo stato serializzabile (ID, opzioni e dati grafici effettivamente necessari) va distinto dalle **funzioni** come `onCharacterRemove`: non serializzare funzioni come dati; definire un meccanismo coerente con Pixi'VN per riassociare i comportamenti dopo il ripristino, da precisare in una decisione successiva. La precedente regola di API 94 sull'estensione dei tipi PixiJS con `rpgDepth?: number` e la precedenza di `rpgDepth` rispetto a `y` (API 93) restano valide, ma il loro ripristino deve seguire il contratto `Memory`/`setMemory()` del componente. **Non modificare ora codice sorgente né repository Pixi'VN: questo handoff registra requisiti architetturali, non autorizza implementazione.**


### API 96 — Riutilizzo della persistenza delle funzioni di Pixi'VN
Decisione confermata (opzione A): per i riferimenti a funzioni e callback RPG persistenti, inclusi gli handler dei layer come `onCharacterRemove`, **riutilizzare la stessa strategia di identificazione, serializzazione del riferimento e risoluzione/ripristino delle funzioni già adottata da Pixi'VN**, eventualmente sfruttando `eventDecorator` se compatibile con il caso d'uso. **Non introdurre un registro parallelo di funzioni in Rpgoria.** La persistenza di componenti e layer rimane quella di Pixi'VN con `Memory`/`setMemory()` come da API 95; le funzioni non devono essere serializzate come codice JavaScript. **Non fissare ora nomi, firme, obbligatorietà del decorator o dettagli di implementazione:** il developer fornirà il codice sorgente di Pixi'VN come contesto durante lo sviluppo; ispezionarlo prima di stabilire l'integrazione concreta. Se necessario, prevedere utility direttamente in Pixi'VN e non duplicazioni in Rpgoria. Questa decisione sostituisce l'ipotesi di un registro separato dei tipi/handler dei layer avanzata come possibile opzione API 96. Nessuna modifica al codice sorgente autorizzata.


### API 97 — Elementi grafici RPG come custom components Pixi'VN
Decisione confermata (opzione A): **ogni tipologia di elemento grafico RPG deve essere dichiarabile direttamente come classe custom component di Pixi'VN**, seguendo il suo contratto effettivo (https://pixi-vn.com/it/start/canvas-components#custom-components), **senza classi base RPG obbligatorie** quali `RpgCharacterView` o `RpgMapObject`. Il developer definisce e registra i propri componenti tramite il meccanismo Pixi'VN; Rpgoria vi applica i comportamenti configurati (es. Y-sort dei figli diretti) senza introdurre un registro o una gerarchia di componenti paralleli. `Memory`/`setMemory()` e la strategia di persistenza delle funzioni di Pixi'VN rimangono i contratti di riferimento (API 95–96); `rpgDepth?: number` resta disponibile (API 93–94) e va incluso nello stato persistente del componente quando appropriato. **Distinguere le view grafiche dal modello dati dei personaggi**: la precedente obbligatorietà di `CharacterMapStoredClass` per gli elementi di stato/registry dei personaggi non viene abrogata. Le firme concrete e l'integrazione devono essere verificate sul codice Pixi'VN fornito in fase di sviluppo. Nessuna modifica al codice sorgente autorizzata.


### API 98 — Collegamento view-personaggio tramite proprietà `characterId`
Decisione confermata (opzione A): **ogni componente grafico Pixi'VN che rappresenta un personaggio RPG espone la proprietà `characterId`**, tipizzata usando il tipo ufficiale di identificatore dei personaggi di Pixi'VN. Rpgoria associa il componente al personaggio leggendo questa proprietà; **non richiede un'API pubblica `bindView(characterId, view)` né una convenzione che imponga la coincidenza tra ID del componente e `characterId`**. La proprietà deve poter essere ripristinata tramite il contratto `Memory`/`setMemory()` del componente (API 95 e 97), nel rispetto delle API effettive di Pixi'VN da verificare sui sorgenti in fase di implementazione. La view non è obbligatoria per l'istanza di `CharacterMapStoredClass` e il modello dati rimane distinto dal componente grafico. Questa decisione non autorizza modifiche al codice sorgente.


### API 99 — Collegamento semplice delle view, nessun controllo aggiuntivo sui duplicati
Decisione dell'utente: `CharacterMapStoredClass` richiede `characterId` nel costruttore e lo espone come proprietà (conferma del contratto già deciso); i componenti grafici Pixi'VN che rappresentano personaggi espongono lo stesso `characterId` per il collegamento (API 98). Il developer crea le view e le aggiunge **direttamente** al Container del layer tramite `layer.addChild(view)` (API 66). **Non introdurre API `bindView`, registrazioni delle view, scansioni preventive di unicità per `characterId`, errori sui duplicati, né sincronizzazione automatica di più view associate allo stesso personaggio.** La corretta associazione è responsabilità del developer. Distinguere la garanzia di PixiJS sulla singola istanza DisplayObject/Container dalla possibile presenza di due istanze distinte con lo stesso `characterId`: PixiJS non garantisce l'unicità semantica di `characterId`, ma Rpgoria non aggiunge controlli per questo caso. Questa decisione sostituisce la proposta di API 99 sul comportamento multi-view. Non assumere che il solo `characterId` definisca già il meccanismo di inoltro degli aggiornamenti `onVisualUpdate` dal modello alla view; questo dettaglio resta da definire separatamente se necessario. Nessuna modifica al codice sorgente.


### API 100 — `onVisualUpdate` predefinito e sovrascrivibile in `CharacterMapStoredClass`
Decisione dell'utente: mantenere **`onVisualUpdate(state: VisualState): void` nella classe base `CharacterMapStoredClass`**, con una **implementazione predefinita concreta** che applichi almeno il movimento/aggiornamento della posizione grafica del personaggio; il developer **può eseguire l'override** del metodo per personalizzare animazioni, rotazione e altri effetti, richiamando facoltativamente `super.onVisualUpdate(state)` per conservare il comportamento di base. Non imporre al developer l'implementazione del metodo per i casi standard. Conservare il calendario delle invocazioni e il contratto `VisualState` già stabiliti (movimento, fine movimento, registrazione iniziale, teleport, cambio direzione da fermo; API precedenti), nonché l'ordine aggiornamento visivo prima dello Y-sort (API 75). **Resta da decidere come l'implementazione di base trovi il componente grafico Pixi'VN** usando `characterId` (API 98), senza introdurre registrazione esplicita delle view o controlli di duplicazione (API 99); non presumere una ricerca/lookup già approvata. Questa decisione sostituisce la proposta API 100 di affidare obbligatoriamente l'aggiornamento della view al codice del developer. Nessuna modifica al codice sorgente autorizzata.


### API 101 — `RpgCharacterInterface` e `CharacterAnimatedSprite`, callback nel costruttore, spritesheet semplice
Decisione confermata (opzione A con rettifica esplicita): **sostituire il requisito di estendere obbligatoriamente `CharacterMapStoredClass` con un'interfaccia `RpgCharacterInterface`**, implementabile direttamente dai custom components Pixi'VN del developer. L'interfaccia definisce il contratto RPG del personaggio: `characterId` (tipo ufficiale Pixi'VN), `x`, `y`, `rpgDepth?`, compatibilità con `Memory`/`setMemory()` Pixi'VN e `onVisualUpdate(state: VisualState)`. Il developer può implementare direttamente l'interfaccia senza ereditare da classi RPG. **Fornire una classe base opzionale `CharacterAnimatedSprite`** (nome definitivo) che estende `AnimatedSprite` di Pixi'VN e implementa il contratto RPG, previa verifica delle API reali nel codice sorgente Pixi'VN. La classe base implementa il comportamento visivo standard: aggiornare `x/y`, selezionare la direzione, avviare/arrestare l'animazione e adeguare la velocità di riproduzione alla velocità del movimento.

Il costruttore di `CharacterAnimatedSprite` accetta un **callback opzionale `onVisualUpdate`**, che viene chiamato **dopo** il comportamento visivo standard; non richiedere al developer la registrazione di listener esterni Pixi'VN per questo callback. Conservare `VisualState` e la cadenza delle chiamate già decisi; il riferimento al callback deve essere compatibile con la strategia di persistenza delle funzioni Pixi'VN (API 96), da verificare sul codice. L'interfaccia definisce il metodo, mentre il callback del costruttore è una comodità della classe base; evitare di confondere le due firme.

**Tipo `CharacterSpritesheet` confermato senza estensioni:** `type CharacterSpritesheet = string | { up: string; down: string; left: string; right: string };`. Una stringa rappresenta **una sola animazione condivisa fra le direzioni**; l'oggetto permette un'animazione per ciascuna direzione. Il parametro `spritesheet` è accettato dal costruttore della classe base. **Non aggiungere ora** una struttura con `idle`, `walk`, `run`: l'idea è stata discussa ma il developer ha esplicitamente richiesto di mantenere il tipo originale. Verificare nel codice Pixi'VN come risolvere stringhe e animazioni prima di implementare.

**Revisioni esplicite delle decisioni precedenti:** il modello `CharacterMapStoredClass` non è più una superclass obbligatoria per registrare personaggi (sostituisce le precedenti API relative alla sua obbligatorietà); il personaggio RPG può coincidere direttamente con il componente grafico Pixi'VN. La proprietà `characterId` continua a collegare l'identità del personaggio (API 98). La classe base `CharacterAnimatedSprite` è opzionale e non contraddice il principio API 97 di nessuna ereditarietà RPG obbligatoria. L'implementazione predefinita di `onVisualUpdate` prima attribuita a `CharacterMapStoredClass` (API 100) passa alla nuova classe base opzionale; il metodo rimane nel contratto dell'interfaccia. La proposta API 101 di ricerca della view nel layer è superata per i componenti che implementano direttamente l'interfaccia. Il developer continua ad aggiungere direttamente gli elementi al layer con `addChild()` e non si introducono controlli di duplicazione aggiuntivi (API 99). Questa è una decisione architetturale, non un'autorizzazione a modificare codice sorgente.


### API 102 — Velocità animazione automatica; `spritesheet` rimosso dal costruttore
Decisione confermata (opzione B): `CharacterAnimatedSprite` calcola **di default** la velocità di riproduzione dell'animazione automaticamente e in modo proporzionale alla velocità di movimento (`state.speed`), tramite un fattore di conversione interno a Rpgoria, senza richiedere al developer `animationSpeed` o `referenceMovementSpeed` nel costruttore. Il callback opzionale `onVisualUpdate` del costruttore viene invocato **dopo** l'aggiornamento standard e può personalizzare/sovrascrivere la velocità dell'animazione e gli altri effetti visivi. Conservare le regole precedenti di posizione, direzione e play/stop.

**Rettifica esplicita di API 101:** **rimuovere `spritesheet` dai parametri del costruttore di `CharacterAnimatedSprite`**. Non imporre né aggiungere tale parametro all'API del costruttore RPG. Il tipo `CharacterSpritesheet = string | { up: string; down: string; left: string; right: string }` è stato discusso e definito in API 101, ma la sua eventuale collocazione/utilità futura non è stata decisa; **non presumere che debba essere usato nel costruttore o che esista già una nuova API per configurarlo**. Verificare il contratto effettivo di `AnimatedSprite` Pixi'VN prima di definire come configurare animazioni/spritesheet. Nessuna modifica al codice sorgente autorizzata.


### API 103 — `defaultSpritesheet: string` obbligatorio e documentazione dell'override
Decisione dell'utente: il costruttore di `CharacterAnimatedSprite` deve accettare **`defaultSpritesheet` di tipo `string`, obbligatorio (non opzionale)**. Questo sostituisce la rimozione totale di `spritesheet` dal costruttore stabilita in API 102: rientra **soltanto** `defaultSpritesheet: string`, non il precedente parametro `spritesheet` né l'unione `CharacterSpritesheet = string | { up; down; left; right }`. Non introdurre per ora API obbligatorie di configurazione delle animazioni direzionali.

**Documentazione richiesta:** nella documentazione di `CharacterAnimatedSprite`, consigliare esplicitamente al developer di creare una propria classe che **estende `CharacterAnimatedSprite`** e di eseguire **l'override di `onVisualUpdate(state: VisualState)`** per personalizzare animazioni (direzioni, camminata, corsa, ecc.), potendo richiamare `super.onVisualUpdate(state)` per conservare posizione e velocità dell'animazione predefinite. Il comportamento di base di movimento, play/stop e velocità automatica proporzionale a `state.speed` (API 100–102) resta valido. **Il callback opzionale `onVisualUpdate` nel costruttore concordato in API 101 non è stato esplicitamente revocato**: mantenerlo per ora come modalità aggiuntiva, ma verificare durante la progettazione come coesista con l'override del metodo, senza presumere un ordine di esecuzione tra i due non ancora definito. Verificare la firma effettiva di `AnimatedSprite` Pixi'VN sul sorgente prima di implementare. Nessuna modifica al codice sorgente autorizzata.


### API 104 — Supportare sia override di `onVisualUpdate` sia callback nel costruttore
Decisione confermata (opzione B): `CharacterAnimatedSprite` deve supportare **entrambe** le modalità di personalizzazione: (1) il developer può creare una sottoclasse ed eseguire `override onVisualUpdate(state: VisualState)`; (2) il costruttore può ricevere un **callback opzionale `onVisualUpdate`**. Il developer può chiamare **`super.onVisualUpdate(state)`** nel metodo sovrascritto per riutilizzare il comportamento predefinito (posizione, direzione, play/stop e velocità dell'animazione proporzionale a `state.speed`); se non chiama `super`, può sostituirlo. **Ordine previsto quando entrambe le modalità sono presenti:** prima il metodo effettivo, incluso l'override e l'eventuale chiamata a `super`; **poi** il callback del costruttore. Progettare il dispatch in modo che il callback venga invocato una volta sola e che il comportamento di base non sia duplicato automaticamente. Verificare i contratti Pixi'VN e il ripristino dei riferimenti alle funzioni tramite la strategia già concordata (API 96). Conservare `defaultSpritesheet: string` obbligatorio (API 103) e documentare l'override. Nessuna modifica al codice sorgente autorizzata.


### API 105 — Animazione continua anche da fermo
Decisione confermata (opzione C): `CharacterAnimatedSprite` **non interrompe automaticamente la riproduzione dell'animazione quando `state.isMoving` diventa `false`**. Non invocare `stop()` automaticamente alla fine del movimento e non riportare il frame al primo frame per il solo fatto che il personaggio è fermo; l'animazione corrente può continuare anche in idle. Il developer può personalizzare la gestione delle animazioni idle tramite `override onVisualUpdate(state)` (eventualmente chiamando `super.onVisualUpdate(state)`) oppure tramite il callback opzionale nel costruttore (API 104). **Questa decisione sostituisce esplicitamente** ogni precedente riferimento nelle API 100–104 all'arresto automatico/play-stop quando il personaggio si ferma. Restano validi aggiornamento della posizione e calcolo automatico della velocità di animazione durante il movimento (API 102); la velocità da applicare da fermo va definita separatamente, senza presumere che diventi zero e blocchi di fatto l'animazione. Nessuna modifica al codice sorgente autorizzata.


### API 106 — Animazione idle e velocità base quando `state.speed === 0`
Decisione dell'utente: aggiungere **`idle`** ai tipi di animazione predefiniti gestiti da `CharacterAnimatedSprite` (oltre alle animazioni di movimento). Quando il personaggio è fermo, `state.speed` **rimane correttamente `0`**: non alterarlo per mantenere attiva l'animazione. Se è disponibile un'animazione **`idle`**, selezionarla automaticamente e riprodurla alla **velocità base dell'animazione**, senza applicare la proporzione con la velocità di movimento. Il developer può personalizzare selezione e velocità tramite `onVisualUpdate` (override e/o callback del costruttore, API 104). Se `idle` non è disponibile, non imporre `stop()` né il ritorno al primo frame: preservare il principio API 105 che l'animazione può continuare da fermo. Durante il movimento resta valido il calcolo automatico della velocità proporzionale a `state.speed` (API 102). Questa decisione supera la proposta API 106 di conservare l'ultima velocità di movimento anche in idle.

**Nota di progettazione:** il developer si riferisce a un tipo/configurazione di animazioni predefinite cui aggiungere `idle`; **non è ancora stata stabilita la firma esatta di tale configurazione**, né come si combina con il parametro obbligatorio `defaultSpritesheet: string` (API 103). Non reintrodurre senza conferma il precedente oggetto `spritesheet` nel costruttore. Verificare prima le API Pixi'VN e precisare la configurazione in una successiva decisione. Nessuna modifica al codice sorgente autorizzata.


### API 107 — `idleSpritesheet?: string` e fallback al primo frame
Decisione confermata (opzione A, con precisazione): il costruttore di `CharacterAnimatedSprite` accetta **`defaultSpritesheet: string` obbligatorio** (API 103) e **`idleSpritesheet?: string` opzionale**. Quando `state.speed === 0`, se `idleSpritesheet` è fornito viene selezionata/riprodotta l'animazione idle alla **velocità base** (API 106), senza usare `state.speed` per determinarne la velocità di playback. **Se `idleSpritesheet` non è fornito, mostrare il primo frame di `defaultSpritesheet` come posa idle statica**, senza continuare a riprodurre l'animazione di movimento. Questo fallback statico è una **rettifica esplicita** alla regola generica API 105 (continuare l'animazione anche da fermo): l'animazione prosegue da fermo se è configurata un'animazione idle; altrimenti il primo frame di default è mostrato staticamente. Il developer può personalizzare il comportamento tramite override di `onVisualUpdate(state)` (con eventuale `super.onVisualUpdate(state)`) e/o callback opzionale del costruttore (API 104). Verificare firme/semantica reali Pixi'VN prima dell'implementazione; non modificare sorgenti senza richiesta.


### API 108 — `onVisualUpdate(state, changes)` change-driven, escluso `deltaTime`
Decisione confermata (opzione B): il contratto di `onVisualUpdate` diventa `onVisualUpdate(state: VisualState, changes: Partial<Omit<VisualState, "deltaTime">>): void`. `state` conserva tutti i campi precedenti, **incluso `deltaTime`**. `changes` contiene **solo i campi il cui valore è cambiato** rispetto all'ultima notifica, **escludendo sempre `deltaTime`**. Il callback viene invocato **solo se almeno un campo rilevante è cambiato**; una variazione del solo `deltaTime` non causa notifiche. Alla prima notifica/inizializzazione di un personaggio, `changes` include tutti i campi rilevanti disponibili. Applicare il contratto sia all'override del metodo `CharacterAnimatedSprite.onVisualUpdate` sia al callback opzionale del costruttore (API 104), e in generale a `RpgCharacterInterface`. `speed`, `velocityX` e `velocityY` descrivono la **velocità effettiva istantanea del movimento**, includendo l'effetto di easing, non semplicemente la velocità nominale/configurata; durante un easing possono quindi cambiare a ogni frame e provocare correttamente notifiche frequenti. La selezione automatica delle animazioni non deve essere reimpostata inutilmente se lo stato di movimento non cambia (compatibilmente con la personalizzazione via `onVisualUpdate`). Aggiornare tutte le firme precedenti con il secondo parametro `changes`. Nessuna modifica al codice sorgente autorizzata.
