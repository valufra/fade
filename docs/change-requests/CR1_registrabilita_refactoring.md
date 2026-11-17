# CR1 – Componenti registrabili e configurabili + refactoring generale

## 0. Scheda

| Campo | Valore |
|---|---|
| ID | CR1 |
| Titolo | Operatori di mutazione e sintomi come componenti registrabili e configurabili; refactoring generale del codice |
| Versione | 0.1 |
| Data | 2026-11-17 |
| Stato | Proposta |
| Priorità | Alta (prerequisito di CR2 e CR3) |
| Tipo di manutenzione | Preventiva / perfettiva (ristrutturazione) |
| Autori | Gruppo FADE ISTA 2026/27 |
| Baseline | commit `33a9dd3`, branch `Management` |
| Dipende da | Test preliminari di caratterizzazione ([panoramica §4](00_panoramica.md)) |
| Abilita | CR2, CR3 |

Convenzioni (profondità B/M/A/N/X, SIS/CIS) e vincoli trasversali: [00_panoramica.md](00_panoramica.md).
Componenti e rischi: [01_reverse_engineering.md](../01_reverse_engineering.md).

## 1. Descrizione e motivazione

Oggi gli operatori di mutazione e i sintomi sono cablati nel codice:

- gli operatori sono metodi di un'unica classe (C6), scelti in `main.py` con una catena `if/elif` sui
  nomi e filtrati da una lista separata in C7. Aggiungere un operatore richiede di modificare almeno tre
  punti (R6);
- i 13 sintomi sono calcolati da un unico metodo di C5, e le soglie (20 valori distinti, percentuali di
  mutazione, 45 valori, 80° percentile…) sono hard-coded in più file (R7);
- `main.py` è uno script monolitico dentro `if __name__ == '__main__'`, senza funzioni testabili (R1);
  calcolo, I/O e stampe a console sono mescolati;
- ci sono codice morto (R8, ≈200 LOC) e dipendenze pesanti inutilizzate (R9: torch, tensorflow,
  transformers, …), che rendono l'installazione lenta e fragile;
- la casualità usa `random` e `np.random` globali, senza un punto unico di controllo (R2).

**Beneficio atteso:** un nucleo che non conosce i singoli componenti, in cui aggiungere un operatore o
un sintomo significa scrivere una classe e registrarla; parametri esposti in modo uniforme; codice
diviso in moduli testabili. È la base su cui poggiano la CR2 (configurazione e seed) e la CR3
(estensione).

## 2. Cambiamento proposto

**In perimetro:**

1. **Interfacce comuni e registro.** Una classe base astratta per gli operatori
   (`name`, `is_applicable(series)`, `apply(df, col, rng, **params)`) e una per i sintomi
   (`name`, `category`, `is_applicable(series)`, `compute(ctx, feature)`); un registro generico con
   decoratore `@register`. Il nucleo itera sul registro invece di usare nomi cablati.
2. **Parametri dichiarati dai componenti.** Ogni componente dichiara i propri parametri con i default
   attuali; i valori hard-coded diventano quei default. L'esposizione tramite CLI e file è compito della
   CR2.
3. **Refactoring generale** (indicazione del tutor):
   - scomposizione di `main.py` in runner (ciclo baseline → mutazione → ricalcolo), I/O e reporting a
     console; resta un entry point di poche righe, **ancora interattivo** fino alla CR2;
   - separazione delle responsabilità: `_show_comparison` e le stampe passano al reporting;
   - rimozione del codice morto (R8) e delle dipendenze inutili (R9);
   - centralizzazione di path e soglie;
   - **iniezione del generatore casuale** (`numpy.random.Generator`) al posto delle chiamate globali;
   - sostituzione di `is_categorical_dtype`, deprecato (R12).

**Architettura target (indicativa):**

```
src/
  fade/
    core/
      runner.py          # ciclo baseline -> mutazione -> ricalcolo (ex corpo di main.py)
      registry.py        # registro generico (decoratore @register)
      context.py         # dataset, target, positive_label, rng, parametri
    mutation/
      base.py            # MutationOperator (classe astratta)
      numeric.py         # increment_decrement_feature, scale_values
      categorical.py     # category_flip, replace_synonyms, add_noise
    symptoms/
      base.py            # Symptom (classe astratta)
      distribution.py    # Gini, Shannon, Simpson, Imbalance Ratio, Kurtosis, Skewness
      relationship.py    # Mutual Information, Kendall Tau
      group_fairness.py  # DSP, UPP, PPP, Unpriv/Priv Group Unbalance
      groups.py          # inferenza dei gruppi (ex infer_group_conditions)
    data/
      loader.py, preprocessor.py
    io/
      results_writer.py  # ex result_handlers + export dei dataset trasformati
      paths.py
    reporting/
      console.py         # stampe dei delta e confronti pre/post
  main.py                # entry point sottile
  statistical_analysis/run_analysis_rq1.py
```

In fase di progettazione questa architettura sarà espressa come evoluzione dei package e class diagram
estratti (§2.4–2.5 del documento di reverse engineering).

**Fuori perimetro:**

- CLI non interattiva, file di configurazione dell'esecuzione e seed (CR2);
- nuovi operatori o sintomi (CR3);
- plugin esterni (entry point di setuptools): basta un registro interno;
- modifiche alle formule dei sintomi, allo schema CSV e ai nomi esistenti (vincoli trasversali).

## 3. Analisi d'impatto preliminare

### 3.1 SIS

C1 `main.py`, C5 `symptom_calculator.py`, C6 `mutation_operator.py`, C7 `analysis_helpers.py`: sono i
moduli che contengono dispatch, nomi cablati e parametri hard-coded, e vengono modificati direttamente.

### 3.2 CIS con profondità

| Comp. | File | Prof. | Cosa cambia |
|---|---|---|---|
| C1 | `main.py` | **A** | Il corpo (≈200 LOC) passa a `core/runner.py` e `reporting/console.py`; il dispatch `if/elif` è sostituito dal registro; l'export dei dataset trasformati passa a `io/`. Resta un entry point di poche righe |
| C5 | `analysis/symptom_calculator.py` | **A** | Il metodo unico `calculate_symptoms_for_feature` è scomposto in 13 componenti registrati; l'inferenza dei gruppi va in un modulo a sé; le soglie diventano parametri. Le formule restano identiche |
| C6 | `mutation/mutation_operator.py` | **A** | La classe unica con 5 metodi diventa 5 classi con base astratta; RNG iniettato; `_show_comparison` passa al reporting; rimozione di `import torch`; percentuali e soglia dei 45 valori come parametri |
| C7 | `utils/analysis_helpers.py` | **X / A** | `get_applicable_operators` è sostituita da `is_applicable` + registro. `get_user_defined_conditions` (non usata): da rimuovere o da spostare in `groups.py` (§4) |
| C3 | `data/preprocessor.py` | **M** | Rimozione del codice morto (one-hot encoding, `fit`/`transform`/`fit_transform`/`apply_temporary_encoding`, `get_preprocessing_summary`: ≈120 LOC); sostituzione di `is_categorical_dtype`; path del config centralizzato |
| C8 | `utils/result_handlers.py` | **M** | Rimozione di `generate_comprehensive_final_dataset` (non usata); il risultato passa da dict con prefissi `pre_`/`post_` a oggetto strutturato; **schema CSV invariato** |
| C9 | `utils/file_utils.py` | **M / X** | Rimozione di `export_transformed_datasets_hierarchical` (non usata); il resto confluisce in `io/` |
| C2 | `data/loader.py` | **B** | Spostamento e import |
| C4 | `config/dataset_config.json` | **B** | Contenuto invariato; letto tramite il modulo dei path |
| C10 | `statistical_analysis/run_analysis_rq1.py` | **B** | Scomposizione in funzioni per renderlo testabile; correzione di R5 se approvata (§4). Logica statistica invariata |
| — | `requirements.txt` | **M** | Rimozione di torch, tensorflow, transformers, flax, tf-keras, sentencepiece, matplotlib; requisiti di sviluppo separati |
| — | `README.md` | **M** | Sezioni Architecture, Project Structure, Workflow |
| — | `tests/` | **M** | Aggiornamento degli import; rigenerazione controllata del golden master |
| D2 | `results/` | — | Nessun impatto, grazie al vincolo sullo schema CSV |

Dipendenze seguite per il CIS: C1 → C5, C6, C7, C8, C9 (import); nomi di operatori e sintomi → CSV →
C10 (stringhe e file, contenuti dal vincolo trasversale); C3 → C4 (file).

### 3.3 Dimensione

Circa il **75–80% del codice sorgente** (≈1.300 delle ≈1.700 LOC) viene toccato, di cui ≈900 LOC in
profondità alta (C1, C5, C6, C7). Il codice nuovo riguarda soprattutto `core/`, le classi base e il
registro.

## 4. Decisioni aperte

Il refactoring dovrebbe **preservare il comportamento**. Alcuni difetti emersi, però, sono veri bug:
correggerli cambia gli output, e va dichiarato.

| Rif. | Questione | Opzioni | Impatto se si corregge | Proposta del redattore |
|---|---|---|---|---|
| R3 | `positive_label` chiesto ma ignorato (target assunto 0/1) | (a) correggere in CR1; (b) documentare e rimandare | C5: gruppi e Group Unbalance. Output diversi solo per target non 0/1; per i 4 benchmark nessun cambiamento | (a): il contesto (`context.py`) lo trasporta comunque |
| R4 | Struttura di `results/` diversa da quella prodotta dal codice | Allineare il codice alla struttura committata (`results/<ds>/feature_<f>/`) oppure lasciare com'è | C8, C9 (M); README | Allineare, perché i risultati diventino rigenerabili |
| R5 | Doppio conteggio in C10 dopo un nuovo run (`glob("results_*.csv")`) | Pattern più restrittivo o elenco esplicito dei file | C10 (B) | Correggere |
| — | `symptom_percent_change = 0` quando il valore pre è 0 | Lasciare com'è oppure NaN | Con NaN quelle righe sono escluse dal CFS in C10, che quindi cambia | Lasciare com'è e documentarlo (compatibilità con D2) |
| R8 | `get_user_defined_conditions` non usata | Rimuovere oppure riusare in `groups.py` | C7 (X) oppure `groups.py` (M) | Rimuovere; eventuale riuso fuori perimetro |

Ogni correzione approvata va in un **commit separato** dal refactoring puro, così che le differenze di
output siano attribuibili.

## 5. Dipendenze e ordine

- **Prima:** test preliminari di caratterizzazione (rete di sicurezza per R1).
- **Dopo:** CR2 e CR3, che presuppongono il registro, le interfacce base e l'RNG iniettato.
- **Organizzazione interna:** si concordano insieme `base.py`, `registry.py` e `context.py`; poi il
  lavoro si divide per sotto-sistema (mutazioni / sintomi / runner, I/O e reporting). La CR è
  realizzata in più commit, uno per passo logico.

## 6. Rischi e mitigazioni

| Rischio | Rif. | Mitigazione |
|---|---|---|
| Regressioni silenziose | R1, R10 | Test preliminari prima di iniziare; eccezioni per operatore almeno registrate in un log invece che solo stampate |
| Rottura dei contratti impliciti (nomi stringa → CSV → C10) | R6 | Test sullo schema del CSV e sui valori di `mutation_type` e `symptom_name` |
| Output numericamente diversi per il cambio dell'ordine di consumo dell'RNG | R2 | Previsto e documentato; golden master rigenerato una sola volta, in un commit dedicato |
| Over-engineering | — | Registro interno semplice, nessun sistema di plugin esterno |
| Rottura con librerie recenti | R12 | Sostituzione di API deprecate durante lo spostamento dei moduli |

## 7. Criteri di accettazione

1. Tutti i test unitari e di proprietà preliminari passano senza modifiche alla loro logica (sono
   ammessi solo aggiornamenti degli import).
2. Il golden master è stato rigenerato al più una volta, in un commit dedicato che ne spiega il motivo.
3. Nel nucleo non ci sono più nomi di operatori o sintomi cablati né catene `if/elif` sui nomi.
4. Un test registra un operatore e un sintomo fittizi e verifica che vengano usati senza toccare il
   nucleo.
5. Lo schema del CSV dei risultati e i nomi di operatori e sintomi sono identici alla baseline
   (verificato da test).
6. Il codice morto elencato in R8 è stato rimosso (o spostato, secondo la decisione al §4).
7. `pip install -r requirements.txt` su Python 3.12 non installa più torch, tensorflow e le altre
   dipendenze inutilizzate, e `python main.py` funziona come prima (in modalità interattiva).
8. Le correzioni dei bug approvate al §4 sono in commit separati, ciascuna con un test.

## 8. Test di regressione

Il CIS copre quasi tutto il programma principale, quindi per la CR1 si riesegue **l'intera test suite**:

| Test | Motivo |
|---|---|
| Unitari dei sintomi (C5) | Formule spostate in 13 componenti |
| Proprietà degli operatori (C6) | Operatori riscritti come classi, con RNG iniettato |
| Applicabilità degli operatori (C7) | `get_applicable_operators` sostituita da `is_applicable` |
| Pre-processing (C3) | Rimozione di codice morto nello stesso modulo |
| Schema del CSV (C8) | Contratto verso C10 e D2 |
| Golden master end-to-end | Comportamento complessivo (rigenerato in modo controllato) |
| Analisi statistica (C10) | Solo se si applica la correzione di R5 o la scomposizione in funzioni |
