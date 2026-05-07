# Esercizzi-Test

Di seguito trovi uno **schema completo stile AS/400 (IBM i)** con:

- schermata (DSPF)
- programma RPG
- esempio `UPDATE` / `WRITE`
- subfile (griglia)
- menu applicativo
- file logico LF
- programma CL che chiama RPG
- mini esempio “gestionale vero anni 2000”

---

## 1) Schema architetturale (panoramica)

```text
UTENTE
  │
  ▼
DSPF (schermata/menu/sottfile)
  │
  ▼
RPGLE (logica applicativa)
  │        ├─ CHAIN/READ su PF (anagrafiche, ordini)
  │        ├─ WRITE (nuovi record)
  │        └─ UPDATE (modifica record esistenti)
  ▼
LF (viste/ordinamenti)
  ▼
PF (dati fisici DB2 for i)

CL (programma di lancio)
  └─ richiama MENU o RPG principale
```

---

## 2) Database: PF + LF

### PF esempio: `CLI001` (Clienti)
Campi tipici:

- `CLCOD` (chiave cliente)
- `CLNOM` (ragione sociale)
- `CLCIT` (città)
- `CLTEL` (telefono)
- `CLSTA` (stato: A=attivo, B=bloccato)

### PF esempio: `ORD001` (Ordini)
Campi tipici:

- `ORNUM` (numero ordine)
- `ORDAT` (data ordine)
- `CLCOD` (cliente)
- `ORIMP` (importo)
- `ORSTA` (stato ordine)

### LF esempio: `LFCLI01` (Clienti per nome)
Scopo: visualizzare i clienti in ordine alfabetico, senza toccare la struttura fisica del PF.

- Base: `CLI001`
- Key: `CLNOM`, `CLCOD`
- Uso: ricerche veloci, subfile ordinata

---

## 3) Schermata base (DSPF) + tasti funzione

Una schermata classica anni 2000 contiene:

- testata con titolo
- campi input chiave/filtri
- area messaggi
- tasti funzione (`F3=Esci`, `F5=Aggiorna`, `F6=Nuovo`, `F12=Annulla`)

Flusso tipico:

1. utente inserisce codice cliente
2. RPG fa `CHAIN` su `CLI001`
3. se trovato: mostra dati
4. se non trovato: messaggio “Cliente inesistente”

---

## 4) Esempio RPGLE: `WRITE` e `UPDATE`

> Pseudocodice RPG free-form, semplificato e didattico.

```rpgle
ctl-opt dftactgrp(*no) actgrp(*new);

dcl-f CLI001 usage(*update:*output) keyed;

dcl-ds ClienteRec extname('CLI001') end-ds;

dcl-s trovato ind;

// Cerca cliente
trovato = %lookup(CLCOD : *all); // placeholder logico

chain (CLCOD) CLI001;
if %found(CLI001);
   // UPDATE: modifica record esistente
   CLTEL = '0111234567';
   CLSTA = 'A';
   update CLI001;
else;
   // WRITE: inserisce nuovo record
   CLCOD = '000123';
   CLNOM = 'ALFA SRL';
   CLCIT = 'TORINO';
   CLTEL = '0111234567';
   CLSTA = 'A';
   write CLI001;
endif;

*inlr = *on;
return;
```

Regola pratica:

- `WRITE` → nuovo record
- `UPDATE` → record letto prima con `CHAIN`/`READ`

---

## 5) Subfile (griglia AS400)

La subfile serve per mostrare liste (es. elenco clienti/ordini) in formato tabellare.

Componenti classici nel DSPF:

- Record formato controllo subfile (`SFLCTL`)
- Record formato riga subfile (`SFL`)
- Indicatori:
  - `SFLDSP`
  - `SFLDSPCTL`
  - `SFLCLR`
  - `SFLEND(*MORE)`

Flusso RPG:

1. pulisci subfile (`SFLCLR`)
2. leggi file (`READ` su LF/PF)
3. per ogni record fai `WRITE` del formato riga subfile
4. attiva display e mostra (`EXFMT` controllo)
5. intercetta opzioni utente (2=modifica, 5=dettaglio, 4=cancella)

---

## 6) Menu applicativo

Menu tipico “gestionale anni 2000”:

```text
1 - Anagrafica Clienti
2 - Inserimento Ordini
3 - Consultazione Ordini
4 - Stampe
9 - Utility
F3 - Uscita
```

Implementazione possibile:

- DSPF menu + programma RPG dispatcher
- oppure comando `GO` custom con opzioni

Il dispatcher chiama i programmi operativi in base all’opzione.

---

## 7) Programma CL che chiama RPG

Esempio minimale:

```cl
PGM
   DCL VAR(&SCELTA) TYPE(*CHAR) LEN(1)

   /* Lancio menu principale */
   CALL PGM(MENU001)

   /* Oppure lancio diretto anagrafica */
   /* CALL PGM(CLI001R) */

ENDPGM
```

Uso reale:

- entry point utente
- set librerie (`CHGLIBL`)
- eventuale controllo profilo/autorità
- chiamata programma RPG

---

## 8) Esempio completo “gestionale vero” (scenario)

### Caso: inserimento ordine cliente

1. **Menu** → scelta “Inserimento Ordini”
2. **CL** richiama `ORDINSR` (RPG)
3. **Schermata testata ordine**:
   - numero ordine
   - data
   - codice cliente
4. RPG fa `CHAIN` su `CLI001`:
   - se cliente non esiste: errore
   - se bloccato: stop operazione
5. Se valido:
   - `WRITE` testata in `ORD001`
   - gestione righe ordine (subfile)
   - calcolo totale
6. Conferma:
   - `UPDATE` stato ordine (`ORSTA='C'` confermato)
7. ritorno al menu

### Oggetti tipici coinvolti

- `MENU001` (DSPF + RPG)
- `CLI001` (PF clienti)
- `LFCLI01` (LF clienti per nome)
- `ORD001` (PF ordini)
- `ORDINSR` (RPG inserimento ordine)
- `STARTAPP` (CL avvio)

---

## 9) Blueprint rapido (da copiare in progetto reale)

- **DB**: PF anagrafiche + LF per chiavi di consultazione
- **UI**: DSPF input + DSPF subfile lista
- **Logica**: RPG service/program con procedure CRUD
- **Orchestrazione**: CL di startup + menu
- **Standard**:
  - F3 uscita
  - F12 annulla
  - messaggi chiari in riga messaggio
  - lock record gestito (monitor su update)

---

Se vuoi, nel prossimo step posso generarti anche i sorgenti separati in stile membro sorgente (`QRPGLESRC`, `QDDSSRC`, `QCLSRC`) con nomi oggetto coerenti e pronti da compilare.
