---
name: gym-excel-tracker
description: >
  Genera un file Excel (.xlsx) completo e personalizzato per il monitoraggio della scheda di palestra. 
  Raccoglie le informazioni sulla struttura della scheda (giorni, esercizi, serie/reps), le funzionalità 
  di tracking desiderate (carichi, 1RM, volume, PR, grafici) e il livello di complessità preferito, 
  poi costruisce il file Excel pronto all'uso con formule automatiche e analisi nel tempo. 
  Usa questa skill ogni volta che l'utente vuole creare un tracker per la palestra, un foglio Excel 
  per la scheda, monitorare i carichi o i progressi in palestra, o dice frasi come "fai un excel per 
  la palestra", "voglio tracciare i miei allenamenti", "crea una scheda excel per la gym", 
  "voglio monitorare i progressi in palestra", "traccia i miei carichi", anche se non specifica tutti i dettagli.
---

# Gym Excel Tracker

Sei un esperto di programmazione dell'allenamento e di Excel. Il tuo obiettivo è creare un file `.xlsx` 
personalizzato, funzionale e pronto all'uso per il monitoraggio della scheda di palestra dell'utente, 
con formule automatiche e analisi nel tempo.

---

## Step 1 — Raccolta informazioni (OBBLIGATORIO)

Prima di scrivere qualsiasi codice, raccogli tutte le informazioni necessarie in **un'unica risposta** 
usando il tool `ask_user_input_v0` con queste domande:

### Domanda 1 — Struttura della scheda
```
Tipo: single_select
Opzioni:
  - "Full Body (1 tipo di giorno)"
  - "Scheda A/B (2 giorni)"
  - "Scheda A/B/C (3 giorni)"
  - "Scheda A/B/C/D (4 giorni)"
  - "Personalizzata (più giorni misti)"
```

### Domanda 2 — Cosa vuoi monitorare per ogni esercizio
```
Tipo: multi_select
Opzioni:
  - "Serie e ripetizioni effettive"
  - "Carico (kg)"
  - "1RM stimato (formula Epley automatica)"
  - "Note / RPE / RIR"
  - "Tempo sotto tensione (TUT)"
```

### Domanda 3 — Analisi e funzionalità aggiuntive
```
Tipo: multi_select
Opzioni:
  - "Progressione del carico per esercizio (sessione per sessione)"
  - "Volume totale per sessione (kg × reps)"
  - "Grafico andamento nel tempo"
  - "Record Personali (PR tracker)"
  - "Media mobile del volume"
```

**Aspetta la risposta prima di procedere.**

---

## Step 2 — Raccolta esercizi

Dopo le risposte alle domande, chiedi gli esercizi in modo conversazionale:

> "Perfetto! Ora dimmi gli esercizi della tua scheda per ogni giorno. 
> Puoi mandarli in modo informale, ad esempio:
> **Giorno A:** panca piana 4×8, squat 3×10, lat machine 3×8...
> Includi anche le serie e le ripetizioni target se le hai."

Se l'utente non specifica serie/reps target, usa i default comuni:
- Esercizi di forza (squat, panca, stacco): 3×5-6
- Esercizi ipertrofia: 3×8-12
- Esercizi isolamento: 3×12-15

---

## Step 3 — Conferma struttura

Prima di generare il file, mostra un riepilogo in formato tabella Markdown:

```
📋 RIEPILOGO SCHEDA
─────────────────────────────────────
Giorni: [X]
Funzionalità: [lista scelta]
Esercizi totali: [N]

GIORNO A — [nome se fornito]
  • Esercizio 1 — Xs Y-Z reps
  • Esercizio 2 — ...

GIORNO B — [nome se fornito]
  • ...
─────────────────────────────────────
Vuoi modificare qualcosa prima che generi il file?
```

Aspetta conferma o correzioni prima di procedere.

---

## Step 4 — Generazione Excel

Leggi prima la skill xlsx:
`/mnt/skills/public/xlsx/SKILL.md`

Poi genera il file con `bash_tool` usando **openpyxl**. 
Struttura i fogli in base alle funzionalità selezionate dall'utente.

### Fogli SEMPRE presenti

#### 📋 Log Allenamenti (foglio principale)
- Titolo colorato in header
- Riga istruzioni
- Per ogni giorno: header colorato distinto + header colonne
- Colonne base: Data | Esercizio | Serie Target | Rep Target | [Serie input] | Kg Max | Note
- Colonne aggiuntive in base alla selezione:
  - Se "1RM": colonna 1RM con formula Epley `=ROUND(kg*(1+reps/30),1)`
  - Se "Volume": colonna Volume `=SUMPRODUCT(kg_cols * reps_cols)`  
  - Se "Note/RPE": colonna Note/RPE
- Celle input in azzurro chiaro (`E3F2FD`), formule su sfondo neutro
- Freeze pane sulla riga 3

#### ℹ️ Istruzioni
- Spiegazione di ogni foglio
- Come aggiornare il file (copy-paste righe per nuove settimane)
- Formula 1RM spiegata
- Sempre primo tab

### Fogli OPZIONALI (aggiungi solo se selezionati)

#### 📈 Progressione Carichi
- Attiva se: "Progressione del carico per esercizio"
- Tabella per ogni esercizio con 12 colonne sessione
- Righe: Data | Kg Max | Δ vs Prev (formula automatica)

#### 📊 Volume Sessioni  
- Attiva se: "Volume totale per sessione" o "Media mobile"
- 24 righe pre-compilate (8 cicli completi)
- Colonne: # | Data | Tipo Sessione | Volume Tot. | Δ vs Prev | Media Mobile 3 | Note
- Grafico LineChart se "Grafico andamento" selezionato

#### 🏆 Record Personali
- Attiva se: "Record Personali (PR tracker)"
- Una riga per esercizio
- Colonne: Esercizio | PR Kg | Reps x PR | 1RM Est. | Data PR | Note | Giorno
- Sfondo giallo per la colonna 1RM

### Palette colori raccomandata
```python
DARK_HEADER   = "1A1A2E"  # titoli principali
DAY_COLOR_1   = "16213E"  # giorno A
DAY_COLOR_2   = "0F3460"  # giorno B  
DAY_COLOR_3   = "533483"  # giorno C
DAY_COLOR_4   = "1B4332"  # giorno D
ACCENT        = "E94560"  # accenti e titolo principale
INPUT_BG      = "E3F2FD"  # celle input utente
ALT_ROW_1     = "E8EAF6"  # righe alternate
SUB_HEADER    = "2D2D44"  # header colonne
```

### Regole formule

**1RM Epley:**
```python
# Prende il MAX tra le serie disponibili
f'=IFERROR(ROUND(MAX(IF(E{r}<>"",E{r}*(1+F{r}/30),0), IF(G{r}<>"",G{r}*(1+H{r}/30),0), IF(I{r}<>"",I{r}*(1+J{r}/30),0)),1),"")'
```

**Volume sessione (solo serie pari × kg dispari):**
```python
# Usa SUMPRODUCT con colonne alternate kg/reps
f'=IFERROR(SUMPRODUCT((E{r}:I{r})*(F{r}:J{r})*(MOD(COLUMN(E{r}:I{r}),2)=1)),"")'
```

**Delta vs precedente:**
```python
f'=IFERROR(IF(D{r}="","",D{r}-D{r-1}),"—")'
```

**Media mobile a 3:**
```python
f'=IFERROR(ROUND(AVERAGE(D{r-2}:D{r}),0),"—")'
```

---

## Step 5 — Verifica e output

Dopo aver generato il file:

1. Esegui sempre la verifica delle formule:
```bash
python /mnt/skills/public/xlsx/scripts/recalc.py /home/claude/gym_tracker.xlsx 30
```

2. Se ci sono errori (`status: errors_found`), correggili e riesegui.

3. Copia il file in output:
```bash
cp /home/claude/gym_tracker.xlsx /mnt/user-data/outputs/gym_tracker.xlsx
```

4. Usa `present_files` per consegnare il file.

5. Dopo il link al file, aggiungi un riepilogo **brevissimo** (3-5 righe) con:
   - Quanti fogli contiene
   - Come si usa al primo allenamento
   - Come aggiornarlo nelle settimane successive (copy-paste righe)

---

## Note comportamentali

- **Non generare il file prima di aver ricevuto la lista esercizi** — senza quella non puoi costruire nulla di utile.
- Se l'utente manda la scheda in formato libero (es. foto, testo disorganizzato), estraila tu e mostra il riepilogo per conferma.
- Se l'utente vuole un tracker minimalista (solo log base), non aggiungere fogli non richiesti. Meno è meglio se non serve.
- Se l'utente vuole aggiornare un tracker esistente (caricato come file), usa `load_workbook` di openpyxl e preserva la struttura esistente.
- Le serie per ogni esercizio di default sono 3. Se l'utente specifica 4 o 5 serie, adatta le colonne di input di conseguenza (S1...S4 o S1...S5).
