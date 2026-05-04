# Analisi Sentiment — Pre-Consultazione (LabAut, Università di Pavia)

Pipeline di analisi del testo applicata alle risposte aperte del questionario pre-consultazione di 22 partecipanti. I notebook elaborano le risposte in italiano su sei domande qualitative, classificando le emozioni con due approcci complementari (modello NLP e dizionario lessicale) e confrontano i gruppi diagnostici su metriche quantitative.

---

## Struttura del progetto

```
/
├── 000_STATO_AVANZAMENTO_ANALISI.xlsx      # Registro reclutamento e stato avanzamento
├── analisi_sentiment_modello_ai.ipynb      # Notebook 1 — Sentiment con modello AI
├── analisi_sentiment_dizionario.ipynb      # Notebook 2 — Sentiment con dizionario lessicale
├── analisi_test_conteggi.ipynb             # Notebook 3 — Conteggi e test statistici
└── risposte/                               # Sottocartella (non versionata)
    ├── 01_risposte.xlsx
    ├── 02_risposte.xlsx
    ├── …
    └── 22_risposte.xlsx
```

### File generati (non versionati)

| File | Prodotto da |
|---|---|
| `analisi_sentiment_risposte.pdf` | Notebook 1 |
| `media_emozioni_per_domanda.csv` | Notebook 1 |
| `media_emozioni_per_domanda.png` | Notebook 1 |
| `conteggio_periodi_parole_caratteri_emozione.csv` | Notebook 1 |
| `conteggio_periodi_parole_caratteri_emozione.png` | Notebook 1 |
| `emozioni_nominate_dettaglio.csv` | Notebook 2 |
| `emozioni_nominate_pivot.csv` | Notebook 2 |
| `emozioni_nominate_pivot_arricchito.csv` | Notebook 2 |
| `dizionario_periodi_dettaglio_v3.csv` | Notebook 2 |
| `dizionario_periodi_riepilogo_v3.csv` | Notebook 2 |
| `parole_caratteri_dizionario_v3.png` | Notebook 2 |
| `wordcloud_globale.png` | Notebook 2 |
| `wordcloud_partecipanti.png` | Notebook 2 |
| `wordcloud_domande.png` | Notebook 2 |
| `demo_analisi_sentiment.pdf` | Notebook 2 |
| `boxplot_adhd_vs_non_adhd.png` | Notebook 3 |
| `test_statistici_ADHD_vs_nonADHD.pdf` | Notebook 3 |

---

## Struttura dei dati di input

### `000_STATO_AVANZAMENTO_ANALISI.xlsx` — foglio `reclutamento`

Registro principale dei partecipanti. Le colonne rilevanti per l'analisi sono:

| Colonna | Contenuto |
|---|---|
| `codice` | Identificativo anonimizzato (es. `RD_01`) |
| `domanda` | Diagnosi ipotizzata al momento del reclutamento |
| `diagnosi` | Diagnosi ricevuta (usata dai notebook per la segmentazione dei gruppi) |
| `GF pre` | Stato del colloquio GF pre-consultazione |
| `pre sottogruppi` | Stato dell'analisi per sottogruppo |
| `pre consenso` | Stato del consenso informato |

### File `NN_risposte.xlsx` — foglio `PRE_CONS`

Ogni file corrisponde a un partecipante. Il foglio `PRE_CONS` ha questa struttura:

- **Riga 1**: intestazioni delle domande nelle colonne B, D, F, H, J, L (indici 1, 3, 5, 7, 9, 11)
- **Riga 2**: risposte corrispondenti nelle stesse colonne
- **Riga 3**: subheader `espressioni salienti` nelle colonne C, E, G, I, K, M (la colonna immediatamente successiva a ogni domanda)
- **Righe 4+**: annotazioni delle espressioni salienti

Le sei domande corrispondono a: Q1, Q2, Q3, Q5, Q6, Q7 del questionario originale.

### Normalizzazione diagnosi

La colonna `diagnosi` del file registro viene normalizzata in etichette standard:

| Testo grezzo (case-insensitive) | Etichetta normalizzata |
|---|---|
| contiene `autismo` | `Autismo` |
| contiene `adhd` | `ADHD` |
| contiene `plusdotazione` | `Plusdotazione` |
| contiene `apc` | `APC` |
| contiene `dsa` | `DSA` |
| cella vuota o `None` | `diagnosi in corso` |
| valore non riconosciuto | valore originale (strip) |

Celle con diagnosi multiple separate da `,`, `;` o `/` vengono splittate e normalizzate individualmente.

---

## Notebook 1 — `analisi_sentiment_modello_ai.ipynb`

### Scopo

Applica il modello NLP `MilaNLProc/feel-it-italian-emotion` (HuggingFace) alle risposte dei 22 partecipanti. Il modello classifica ogni periodo in quattro emozioni: **joy**, **sadness**, **anger**, **fear**.

### Pipeline

1. Ogni risposta viene suddivisa in periodi usando come delimitatori `.?!:;` e i ritorni a capo (periodi con meno di 10 caratteri vengono scartati).
2. Ogni periodo viene passato al modello (troncato a 512 token), che restituisce uno score (0–100%) per ciascuna delle quattro emozioni.
3. L'emozione con lo score più alto determina l'etichetta del periodo (`emozione_top`).
4. La media per domanda è **pesata per numero di parole** del periodo: periodi più lunghi contribuiscono di più alla media aggregata.
5. Parole e caratteri sono distribuiti tra le quattro emozioni **proporzionalmente agli score**: ogni periodo "pesa" le sue parole/caratteri in proporzione alla distribuzione emotiva assegnata dal modello.

### Output principali

**`analisi_sentiment_risposte.pdf`** — per ogni partecipante e ogni domanda: testo di ogni periodo, etichetta emotiva con score, tabella riepilogativa (score medio, parole e caratteri pesati per emozione).

**`conteggio_periodi_parole_caratteri_emozione.csv`** — aggregato su tutti i partecipanti e tutte le domande:

| Colonna | Descrizione |
|---|---|
| `emozione` | joy / sadness / anger / fear |
| `periodi` | N. periodi in cui l'emozione è dominante |
| `perc_periodi` | % sul totale periodi |
| `parole` | N. parole pesate proporzionalmente agli score |
| `perc_parole` | % sul totale parole |
| `caratteri` | N. caratteri pesati proporzionalmente agli score |
| `perc_caratteri` | % sul totale caratteri |

**`media_emozioni_per_domanda.csv`** — score medio di ogni emozione per ciascuna delle 6 domande, aggregato sui 22 partecipanti.

### Dipendenze Python

```python
pip install transformers torch openpyxl pandas reportlab matplotlib
```

Il modello viene scaricato automaticamente da HuggingFace al primo avvio (~500 MB). È necessaria una connessione internet per il download iniziale; successivamente viene usata la cache locale.

---

## Notebook 2 — `analisi_sentiment_dizionario.ipynb`

### Scopo

Approccio lessicale complementare al modello AI: estrae le emozioni **nominate esplicitamente** dai partecipanti usando un dizionario emotivo italiano e, facoltativamente, KeyBERT per il rilevamento automatico di termini emotivi non presenti nel dizionario.

### Pipeline

1. Stessa logica di lettura file del Notebook 1 (foglio `PRE_CONS`, stesse 6 colonne).
2. Il testo viene suddiviso in periodi usando sia la punteggiatura che le congiunzioni italiane avversative e coordinanti (es. `ma`, `però`, `tuttavia`, `e`, `quindi`…).
3. Per ogni periodo viene effettuata una ricerca nel **dizionario emotivo** (`DIZIONARIO_EMOZIONI`): un lessico di circa 50 voci organizzate in quattro macro-categorie (gioia/positivo, tristezza, paura/ansia, rabbia), ciascuna con le varianti morfologiche (singolare, plurale, forme verbali, avverbi).
4. KeyBERT può essere usato per estrarre automaticamente termini emotivi non previsti dal dizionario.

### Dizionario emotivo

Il dizionario copre le seguenti macro-categorie con le relative voci principali:

**Gioia / Positivo:** gioia, felicità, entusiasmo, sollievo, speranza, soddisfazione, curiosità, gratitudine, orgoglio, fiducia, serenità, ottimismo, amore, affetto, aspettativa, contentezza

**Tristezza:** tristezza, malinconia, dolore, sofferenza, dispiacere, delusione, sconforto, disperazione, solitudine, rimpianto, depressione, malessere

**Paura / Ansia:** paura, ansia, preoccupazione, timore, terrore, angoscia, inquietudine, insicurezza, incertezza, vulnerabilità

**Rabbia:** rabbia, frustrazione, irritazione, risentimento, indignazione, collera, odio

### Output principali

**`emozioni_nominate_dettaglio.csv`** — una riga per ogni occorrenza emotiva rilevata: partecipante, domanda, forma base, variante trovata nel testo, metodo (dizionario / KeyBERT).

**`dizionario_periodi_dettaglio_v3.csv`** / **`dizionario_periodi_riepilogo_v3.csv`** — conteggio di parole, caratteri e periodi per emozione applicando la stessa logica di ponderazione proporzionale del Notebook 1.

**Word cloud (PNG):** `wordcloud_globale.png` (tutti i partecipanti × tutte le domande), `wordcloud_partecipanti.png` (un riquadro per partecipante), `wordcloud_domande.png` (un riquadro per domanda).

**`demo_analisi_sentiment.pdf`** — esempi dimostrativi dell'algoritmo di estrazione su frasi sintetiche con intensificatori (es. `molto felice`, `per niente soddisfatto`, `abbastanza sereno`).

### Dipendenze Python

```python
pip install openpyxl pandas keybert sentence-transformers matplotlib
```

KeyBERT e sentence-transformers sono facoltativi: il dizionario funziona autonomamente.

---

## Notebook 3 — `analisi_test_conteggi.ipynb`

### Scopo

Calcola metriche quantitative sulle risposte (caratteri, parole, espressioni salienti per domanda) e confronta statisticamente il gruppo **ADHD** contro il gruppo **non-ADHD**.

### Pipeline

1. Legge le diagnosi da `000_STATO_AVANZAMENTO_ANALISI.xlsx` (foglio `reclutamento`, colonna `diagnosi`) e le normalizza con la stessa funzione degli altri notebook.
2. Apre i 22 file `NN_risposte.xlsx` e costruisce due DataFrame:
   - **DataFrame risposte**: righe = soggetti, colonne = Q1…Q6 + diagnosi
   - **DataFrame conteggi**: per ogni soggetto e ogni domanda, conta caratteri, parole ed espressioni salienti (annotate in riga 3+ del foglio PRE_CONS)
3. Genera istogrammi per domanda delle tre metriche.
4. Esegue i test statistici ADHD vs non-ADHD.

### Test statistici

Per ogni metrica (caratteri, parole, espressioni salienti) su ogni domanda e sui totali vengono eseguiti due test indipendenti:

**Test sulla media:**
- Shapiro-Wilk per verificare la normalità in entrambi i gruppi
- Se entrambi normali → t-test di Welch
- Altrimenti → Mann-Whitney U (two-sided)

**Test sulla mediana:** test di Mood (χ²). Non richiede normalità; verifica se i due gruppi si distribuiscono simmetricamente attorno alla mediana comune.

**Effect size:** Cohen's d — soglie: piccolo |d| < 0.5, medio 0.5 ≤ |d| < 0.8, grande |d| ≥ 0.8.

### Segmentazione ADHD

Il gruppo ADHD include tutti i soggetti con `ADHD` nella stringa normalizzata della colonna `diagnosi`, incluse le diagnosi in comorbidità (es. `Autismo, ADHD`). Il gruppo non-ADHD include tutti gli altri soggetti con diagnosi nota.

### Output principali

**`boxplot_adhd_vs_non_adhd.png`** — boxplot affiancati ADHD vs non-ADHD per caratteri totali, parole totali ed espressioni salienti totali, con p-value annotato e singoli soggetti sovrapposti (jitter).

**`test_statistici_ADHD_vs_nonADHD.pdf`** — PDF A4 con:
- Tabella riepilogativa di tutti i test (sfondo rosso = p < α)
- Dettaglio per metrica: statistiche descrittive, risultato Shapiro-Wilk, test sulla media, test sulla mediana, Cohen's d
- Boxplot allegato

### Dipendenze Python

```python
pip install openpyxl pandas matplotlib scipy reportlab
```

---

## Requisiti comuni

Python ≥ 3.10. Tutti i notebook vanno eseguiti con la cartella del notebook come working directory (`BASE_DIR = Path(".")`), con la sottocartella `risposte/` presente nella stessa posizione.

```python
pip install transformers torch openpyxl pandas reportlab matplotlib scipy \
            keybert sentence-transformers
```

---

## Ordine di esecuzione

I tre notebook sono **indipendenti** tra loro: ciascuno legge autonomamente i file Excel di input e non dipende dall'output degli altri. Possono essere eseguiti in qualsiasi ordine.

```
Notebook 1  →  sentiment con modello AI
Notebook 2  →  sentiment con dizionario + word cloud
Notebook 3  →  conteggi quantitativi + test ADHD vs non-ADHD
```

---

## Dati e privacy

I file `NN_risposte.xlsx` contengono risposte aperte dei partecipanti e non devono essere versionati. Aggiungere al `.gitignore`:

```
risposte/
000_STATO_AVANZAMENTO_ANALISI.xlsx
```
