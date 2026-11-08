# Ferie Planner

Pianificazione ferie, permessi, ex festività, malattia e malattia figlio con **login PIN a 6 cifre** e **sincronizzazione cloud su Supabase**.

## Tema Apple / reverse-brand (UI)
L'interfaccia usa i token Apple estratti in `../reverse-brand/out/www.apple.com/tokens.css` e i componenti di `../reverse-brand/out/www.apple.com/brand-guide.html`. Non è un clone: i token sono riadattati all'app mantenendo struttura e funzioni esistenti.

- **Font**: stack di sistema `-apple-system, BlinkMacSystemFont, "SF Pro Display", "SF Pro Text", "Helvetica Neue", Helvetica, Arial, sans-serif` (nessun font remoto; `Inter` rimosso).
- **Colori base**: azione `#0071e3` (dark `#2997ff`), superfici `#f5f5f7` / `#ffffff`, testo `#1d1d1f`, secondario `#6e6e73`, muted `#86868b`, bordo `#d2d2d7`.
- **Categorie** (mappate sui colori di sistema Apple): ferie `#0071e3`, permesso `#34c759`, ex festività `#ff9f0a`, malattia `#ff3b30`, malattia figlio `#af52de`. Le varianti `-bg` / `-light` sono tinte soft per le celle del calendario.
- **Forma**: card `--radius: 28px`, controlli/input `12px`, bottoni e pill `--radius-pill: 980px`, modali `28px` (bottom-sheet `24px` sotto 600px).
- **Superfici**: card bianche con bordo quasi invisibile (`--card-border`) e ombra minima; niente ombre pesanti. Vetro (`--glass`, `backdrop-filter: saturate(180%) blur(24px)`) per header sticky, dropdown profilo e login.
- **Header**: barra sticky in vetro con `.app-header-text` (titolo + sottotitolo) a sinistra e `.app-header-actions` (profilo, Salva, sync badge, Esci, tema) a destra. Sotto 760px va a capo; il sottotitolo sparisce sotto 760px.
- **Sfondo calendario**: markup `.app-bg` fisso (`z-index:0`, `pointer-events:none`) con due bagliori radiali (blu/ambra), un calendario decorativo ruotato (barra gradiente, griglia di puntini, barrette evento) e una griglia di puntini. `.container` sta a `z-index:1`; il login resta sopra (`z-index:99999`). Il calendario decorativo è nascosto sotto 600px (restano glow e puntini).
- **Input**: sfondo `--surface-2`, bordo trasparente, radius 12, focus con bordo `--action` e ring `0 0 0 4px rgba(0,113,227,.16)`.
- **Calendario**: celle `border-radius: 14px` (10px mobile), hover su `--surface-2`, domenica con tinta rossa soft, oggi con bordo `--action`, selezione con outline blu e fondo `rgba(0,113,227,.12)`. Gli eventi usano fondo `*-light` e barra sinistra 3px del colore categoria.
- **Griglia calendario**: `grid-template-columns: repeat(7, minmax(0, 1fr))` e celle con `min-width: 0; overflow: hidden`. Senza `minmax(0, 1fr)` le pill con `white-space: nowrap` allargavano le colonne con eventi e stringevano quelle vuote (calendario “sformattato” su iPhone 12, riprodotto a 390px). Sotto 600px le pill mostrano solo le ore (`8h`): l'etichetta categoria è in `.pill-label` ed è nascosta via CSS, mentre colore e barra sinistra continuano a identificare la categoria.
- **Altri componenti**: barra selezione con superficie + ombra, modal con overlay blur, toast pill scuro, tabelle con header maiuscoletto e bordi `--border`, login con card in vetro e tastierino pill.
- **Dark mode**: `[data-theme="dark"]` ricalibra le stesse variabili su nero/`#1d1d1f`/`#2c2c2e`, con accenti più luminosi (`#2997ff`, `#30d158`, `#ff453a`, `#bf5af2`). `color-scheme` è impostato su `light`/`dark` per i controlli nativi.
- **theme-color**: `toggleTheme()` e `initTheme()` aggiornano il meta `theme-color` (`#f5f5f7` light, `#000000` dark); il manifest usa `background_color`/`theme_color` `#f5f5f7` e un'icona calendario blu.
- **Dove sta il CSS**: il grosso del restyle è in un blocco di override in fondo al `<style>`, dopo le media query originali, con proprie media query (900/760/599/400px) e `prefers-reduced-motion`. Non rimuoverlo: molti selettori originali restano e vengono sovrascritti qui.
- **Vincoli**: gli ID e le funzioni JS esistenti sono invariati (`#theme-toggle`, `#logoutBtn`, `#btn-sync`, `#sync-badge`, `toggleCard`, `moveCardBy`, `applyToSelected`, …). `#logoutBtn` e `.btn-sync` sono stati riportati da `position:absolute` a `static` dentro l'header flessibile.

## Categorie evento
Un registro centrale `CATEGORIES` in `index.html` è l'unica fonte di verità per le 5 categorie (chiave stato, chiave bozza, classe CSS, etichetta, id HTML):

| Categoria | Chiave stato | Bozza | Colore | Note |
|---|---|---|---|---|
| Ferie | `ferie` | `draftFerie` | blu | — |
| Permesso | `permesso` | `draftPermesso` | verde | — |
| Ex festività | `exFesta` | `draftExFesta` | ambra | — |
| Malattia | `malattia` | `draftMalattia` | rosso | nessun tetto |
| Malattia figlio | `malattiaFiglio` | `draftMalattiaFiglio` | viola | **max 10 giorni/anno** (`MALATTIA_FIGLIO_MAX_H = 80` ore, 8h/giorno) |

- **Limite malattia figlio**: avviso *soft* (toast + testo rosso nella sezione Rimanenza e nel dettaglio categoria). Non blocca mai l'inserimento.
- **Retrocompatibilità**: `defaultInitial()` / `normalizeInitial()` aggiungono le chiavi mancanti ai dati vecchi. La malattia figlio viene precompilata a 80h quando la chiave è assente; uno `0` esplicito viene rispettato.
- **Nessuna migrazione DB**: `initial` ed `events` sono JSONB, quindi le nuove chiavi viaggiano nel formato esistente.

## Ripartizione malattia figlio per figlio
- Configurazione in `state.children = [{ name, days }]` (per profilo-anno), editabile nella card **Saldo Iniziale → Figli**. `days` = giorni spettanti nell'anno (default 5 al primo figlio, 5 al secondo).
- Ogni evento mantiene il totale numerico `malattiaFiglio` / `draftMalattiaFiglio` (usato da calendario, riepiloghi e limite globale) **più** la mappa di attribuzione `malattiaFiglioBy` / `draftMalattiaFiglioBy` (`{ "Figlio 1": 8 }`).
- I dati salvati prima dei tag vengono migrati automaticamente in `Generico` (`normalizeEventChildMaps`), senza perdere ore.
- UI: selettore del figlio nel modal e nella barra di applicazione multipla; ripartizione con giorni spettanti nella Rimanenza e nel dettaglio; select per riassegnare le ore di un giorno a un altro figlio.
- Il selettore è **sempre visibile**: elenca i figli configurati, `Generico`, `＋ Nuovo figlio…` (crea il figlio al volo) e `✎ Rinomina figlio…`. Con figli configurati parte dal primo figlio; `Generico` resta selezionato solo se scelto a mano.
- La sync cloud e l'export/import includono `children` insieme a `initial`/`events`.

### Nomi dei figli personalizzabili
- Il nome è modificabile in due punti: campo di testo nella card **Saldo Iniziale → Figli** (`updateChild(index, 'name', value)` su `change`) e voce `✎ Rinomina figlio…` nel selettore del modal/bulk (`onChildPick`).
- `renameChildInEvents(oldName, newName)` migra le ore già registrate: rinomina la chiave in `malattiaFiglioBy` e `draftMalattiaFiglioBy` su tutti gli eventi, sommando eventuali collisioni. Nessuna ora va persa.
- Nomi unici case-insensitive: un duplicato viene rifiutato con toast e la UI torna al valore precedente. Un nome vuoto annulla la modifica.
- Rimuovere un figlio con ore registrate chiede conferma e sposta le ore su `Generico` (non le cancella).
- Il selettore conserva la scelta anche dopo la rinomina (`selectChildInPickers`).

## Card comprimibili
Le card **Saldo Iniziale**, **Rimanenza**, **Calendario** e **Riepilogo Mensile** possono essere chiuse/aperte singolarmente.

- Markup: `div.card[data-card="saldo|rimanenza|calendario|riepilogo"]` con header `.card-head` (o `.cal-header` + `.cal-tools` per il calendario), pulsante `.card-collapse[data-card-toggle]` e corpo `[data-card-body]`.
- CSS: `[data-card].collapsed > [data-card-body] { display:none }`; il chevron ruota e l'etichetta passa da “Comprimi” a “Espandi” (nascosta sotto 600px, resta il chevron).
- Stato persistito in `localStorage` (`ferie-planner-card-state`, mappa `{ card: collapsed }`) tramite `getCardState` / `setCardCollapsed` / `toggleCard`; `applyCardState()` riapplica lo stato all'avvio dopo `renderAll()`.
- Da chiusa, l'header mostra un riepilogo compatto (`#saldo-compact`, `#remaining-compact`) calcolato da `renderCompactSummaries()` e aggiornato da `renderAll()` e `updateInitial()`.
- Il **Riepilogo Mensile** è una card autonoma (`#monthly-summary`) dopo il Calendario, con collapse indipendente. `renderMonthlySummary()` non riscrive più l'intero `innerHTML`: mostra/nasconde `#monthly-table-wrap` e `#monthly-empty`, così header e pulsante restano intatti.
- La **Vista Annuale** resta una card separata (`#annual-card`) con il suo pulsante di chiusura.
- **Mobile (<600px)**: da chiuse, il titolo e il riepilogo compatto sono in colonna (`.card-head h2` diventa `flex-direction: column`) e il riepilogo va a capo invece di essere troncato. Le parti del riepilogo usano spazi non-breaking (`\u00A0`, es. `Ferie\u00A025.0\u00A0g`) per non spezzare numero e unità.
- **Calendario chiuso (<480px)**: header compatto su una riga. `.cal-tools` diventa `position: absolute` in alto a destra, `.annual-toggle` è nascosto e `.cal-header` perde il margine inferiore (card ~57px invece di ~110px). Il titolo del mese è centrato con una griglia `minmax(0,1fr) auto minmax(0,1fr)` e i pulsanti mese hanno padding `5px 7px` per non sovrapporsi al titolo a 320px.

## Card riposizionabili (drag & drop)
Tutte e cinque le card sono spostabili: Saldo Iniziale, Rimanenza, Vista Annuale, Calendario e Riepilogo Mensile.

- Markup: le card stanno dentro `#cards-stack` e ognuna ha `data-sortable="saldo|rimanenza|annuale|calendario|riepilogo"`. Ogni header espone un handle `.card-drag[data-card-drag]` (`⠿`) separato dai pulsanti collapse/nav, così trascinare non attiva altri comandi.
- Ordine default: `['saldo','rimanenza','annuale','calendario','riepilogo']`. L'ordine corrente è persistito in `localStorage` con chiave `ferie-planner-card-order` (array di nomi, solo locale: non fa parte dello stato cloud/Supabase).
- `getCardOrder()` normalizza l'array salvato (scarta nomi ignoti/duplicati e appende i mancanti), `applyCardOrder()` riordina i nodi con `appendChild` in quell'ordine e `saveCardOrder()` riscrive l'array leggendo il DOM. `applyCardOrder()` è invocato nell'init (IIFE `initCardReorder`), quindi l'ordine è applicato al primo render.
- Drag con **Pointer Events** (`pointerdown` su `#cards-stack` con delegation, poi `pointermove`/`pointerup`/`pointercancel`/`blur` su `window`): si evita `setPointerCapture` perché il riordino sposta i nodi nel DOM e il capture sul nodo spostato è fragile. `touch-action: none` sull'handle abilita il drag touch su PWA iPhone.
- Riordino live durante il drag confrontando la `clientY` con il midpoint delle card visibili (`visibleSortableCards`, filtra `display:none` come Vista Annuale/Riepilogo chiusi). Auto-scroll quando si trascina a meno di 70px dai bordi.
- Accessibilità: con l'handle a fuoco le frecce `↑`/`↓` spostano la card di una posizione (`moveCardBy`), con `aria-label` e `title` descrittivi.
- `body.card-dragging` disabilita la selezione del testo e forza il cursore `grabbing`; `.card.dragging` alza `z-index` e mostra l'ombra.

## Selezione multipla (barra bulk)
- `#selection-bar` è nascosta con `display:none` e appare solo con `.visible` (classe aggiunta da `updateSelectionBar()`), quindi non riserva più ~300px di spazio vuoto sopra il calendario su mobile.
- Durante il drag `moveDrag()` chiama `updateSelectionBar(true)`: aggiorna il contatore ma non mostra la barra. La barra compare al rilascio (`endDrag`), così il calendario non si sposta sotto il dito mentre si seleziona.
- Importante: non reintrodurre `visibility:hidden` inline sull'elemento (il vecchio markup lo aveva e rendeva la barra invisibile anche con `.visible`, perché lo stile inline vince sulla classe). La vecchia `transition: visibility` è stata rimossa insieme allo spazio riservato.
- Su mobile (<600px) le celle calendario hanno `aspect-ratio: auto`: con `aspect-ratio: 1` e `min-height: 46px` la griglia forzava 7 colonne da 46px e andava in overflow orizzontale a 320px.

## Struttura
```
Ferie-Planner/
├── index.html                    — App completa (tema Apple, sfondo calendario, calendario, saldi, vista annuale, card comprimibili, dark mode, login PIN)
├── manifest.json                 — PWA manifest
├── sw.js                         — Service Worker pass-through
├── config/
│   └── supabase-config.js        — window.SUPABASE_CONFIG {url, anonKey, pinEmail} (pubblico, committato)
├── .github/workflows/keepalive.yml — Anti-pausa free tier (URL e anon key da Actions variables)
├── data.json                     — ⚠️ gitignored: vecchio export, mai nel repo
└── AGENTS.md / README.md
```

## Stack
- HTML/CSS/JS vanilla, lucide-icons
- PWA, PWA installabile (manifest + service worker pass-through)
- Backend: **Supabase** (Postgres + Auth + RLS) — SDK `supabase-js` CDN UMD.

## Login
- **PIN 6 cifre = password** dell'account Supabase (`config/supabase-config.js → pinEmail`). Il PIN vive solo su Supabase, mai in repo.
- Sessione NON persistita (`auth.persistSession:false`): PIN ad ogni apertura. Button "🚪 Esci" per uscire.
- Primo accesso: se l'utente non esiste viene creato (`signUp`); con "Conferma email" attiva serve la mail di conferma una volta.
- Se appare "email rate limit exceeded": NON riprovare, attendere ~1 ora con lo stesso PIN.

## Database (Supabase, progetto `gfglazxhxxplhoteaahr`, schema `ferie`)
- `ferie.state(user_id uuid PK → auth.users, profiles jsonb, timestamps jsonb, active_profile text, last_update timestamptz, updated_at)` — **una riga per utente**; le chiavi di `profiles` (e di `timestamps`) sono `"Nome-YYYY"` → `{initial, events}` (template timestamps = ISO per-chiave).
- RLS: `authenticated` legge/scrive solo la propria riga (`auth.uid()`); policy anon `using(false)` per il keepalive (200 senza dati).
- Salvataggi: ogni mutazione scrive su localStorage (cache/mirror) e fa upsert della riga `state` via SDK `.schema('ferie')`, con debounce ~400ms.

## Sync cloud (merge per-chiave + retry)
- **Mirror locale**: localStorage `ferie-planner-{Nome}-{Anno}` (+ profili/active/theme). Una mappa `ferie-planner-sync-timestamps` (e campo `timestamps` sul cloud) traccia la ISO di ultima modifica per ogni chiave `Nome-YYYY`.
- **Merge**: a ogni sync si legge la riga esistente e si uniscono le chiavi campo-per-campo: per ogni `Nome-YYYY` vince la più recente tra locale e cloud (mai rimpiazzo cieco; il cloud non viene sovrascritto dal locale obsoleto e viceversa).
- **Robustezza**: se l'upsert fallisce → backoff esponenziale (2s→60s) con retry automatico; badge persistente in topbar (`⏳ Non sincronizzato`) finché non torna ok; retry anche su evento `online`; flush best-effort su `beforeunload`. Eventuali limiti rate (Supabase) vengono ritentati senza spam di toast.
- All'avvio (dopo login) i dati si caricano dal cloud e popolano il mirror localStorage; all'inizio viene anche fatto un sync per spingere eventuali dati locali più recenti.

## Note operative
- Actions variables del repo: `SUPABASE_URL` e `SUPABASE_ANON_KEY` (Settings → Secrets and variables → Actions → Variables), altrimenti keepalive saltato con warning.
- `config/supabase-config.js` è committato (serve a Pages) ma contiene SOLO dati pubblici (url, anon key, `pinEmail`). La `service_role` key sta SOLO in `Config Utility/` (fase progetto) — mai nel repo.
- Le festività sono codificate con mesi 0-based: Ognissanti = `{m:10,d:1}` (1 nov), Immacolata = `{m:11,d:8}` (8 dic).