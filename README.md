# Ferie Planner

Pianificazione ferie, permessi ed ex festività, con **login PIN a 6 cifre** e **sincronizzazione cloud su Supabase** (cross-device).

- **Calendario interattivo**: clicca un giorno per assegnare/rimuovere ferie, permessi, ex festività.
- **Saldi parziali**: ore consumate per categoria e residui, divisi per la giornata lavorativa (8 ore).
- **Riepilogo mensile** e **vista annuale** (12×31) con colori per stato.
- **Dark mode**, **undo** (5 livelli), **export/import JSON**, festività italiane e tooltip.
- **Profilo multiplo** (Simone, Michela, …): ogni profilo salva i propri dati per anno.
- **Cloud**: ogni salvataggio aggiorna Supabase (schema `ferie`); all'apertura i dati vengono ripresi dal cloud dopo il login.

## Uso

1. Apri https://zizzo91.github.io/Ferie-Planner/
2. Inserisci il **PIN a 6 cifre** (la password dell'account `simone.marramao@hotmail.it`).
3. Usa il selettore profilo in alto e il calendario per gestire ferie/permessi.

> Il PIN viene richiesto ad ogni apertura (sessione non persistita). Se al primo accesso su un nuovo dispositivo serve confermare l'email, clicca il link nella mail di verifica e poi reinserisci lo stesso PIN.

## Dati

I dati sono salvati in localStorage (mirror) e sincronizzati su **Supabase** (progetto `gfglazxhxxplhoteaahr`, schema `ferie`, una riga per utente protetta da RLS).

## Sviluppo

Nessun build step: apri `index.html` in un browser o servila da GitHub Pages.
Per l'anti-pausa del free tier è configurato `.github/workflows/keepalive.yml` (richiede le Actions variables `SUPABASE_URL` e `SUPABASE_ANON_KEY`).