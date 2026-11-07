# Ferie Planner

Pianificazione ferie, permessi, ex festività, malattia e malattia figlio con **login PIN a 6 cifre** e **sincronizzazione cloud su Supabase**.

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

## Selezione multipla (barra bulk)
- `#selection-bar` è nascosta con `display:none` e appare solo con `.visible` (classe aggiunta da `updateSelectionBar()`), quindi non riserva più ~300px di spazio vuoto sopra il calendario su mobile.
- Durante il drag `moveDrag()` chiama `updateSelectionBar(true)`: aggiorna il contatore ma non mostra la barra. La barra compare al rilascio (`endDrag`), così il calendario non si sposta sotto il dito mentre si seleziona.
- Importante: non reintrodurre `visibility:hidden` inline sull'elemento (il vecchio markup lo aveva e rendeva la barra invisibile anche con `.visible`, perché lo stile inline vince sulla classe). La vecchia `transition: visibility` è stata rimossa insieme allo spazio riservato.
- Su mobile (<600px) le celle calendario hanno `aspect-ratio: auto`: con `aspect-ratio: 1` e `min-height: 46px` la griglia forzava 7 colonne da 46px e andava in overflow orizzontale a 320px.

## Struttura
```
Ferie-Planner/
├── index.html                    — App completa (calendario, saldi, vista annuale, card comprimibili, dark mode, login PIN)
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