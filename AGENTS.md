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
- La sync cloud e l'export/import includono `children` insieme a `initial`/`events`.

## Struttura
```
Ferie-Planner/
├── index.html                    — App completa (calendario, saldi, vista annuale, dark mode, login PIN)
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