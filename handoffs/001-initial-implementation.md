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
- Rpgoria non crea né distrugge automaticamente sprite, container o layer dell'utente; l'utente crea e compone gli oggetti grafici.
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
- Rpgoria espone root `rpgoria.layers.container` (PixiJS Container); developer lo aggiunge a Pixi'VN, es. `canvas.layers.add("rpg",rpgoria.layers.container)`.
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
