# FADE – Change Request: panoramica e convenzioni

> Versione 0.1 – 2026-11-17. Baseline: commit `33a9dd3`, branch `Management`.
>
> Questo documento raccoglie ciò che è comune alle tre Change Request (CR): convenzioni, vincoli
> trasversali, attività preliminare, vista d'insieme degli impatti e ordine di lavoro.
> Ogni CR, con la propria analisi d'impatto preliminare, sta in un file a sé:
>
> - [CR1 – Componenti registrabili e configurabili + refactoring generale](CR1_registrabilita_refactoring.md)
> - [CR2 – CLI non interattiva, file di configurazione, seed riproducibile](CR2_cli_config_seed.md)
> - [CR3 – Nuovo operatore di mutazione e nuovo sintomo](CR3_nuovo_operatore_sintomo.md)
>
> Componenti (C1–C10, D1–D2) e rischi (R1–R12) sono quelli definiti nel documento di reverse
> engineering: [01_reverse_engineering.md](../01_reverse_engineering.md).

---

## 1. Struttura di una CR

Il corso non prevede un template: il gruppo ha adottato la struttura seguente, uguale per le tre CR.

| § | Sezione | Contenuto |
|---|---|---|
| 0 | Scheda | ID, titolo, codice di versione, stato, priorità, tipo di manutenzione, autori, dipendenze |
| 1 | Descrizione e motivazione | Problema attuale e beneficio atteso |
| 2 | Cambiamento proposto | Cosa si fa, cosa resta fuori (perimetro) |
| 3 | Analisi d'impatto preliminare | SIS e CIS con profondità per componente, dimensione stimata |
| 4 | Decisioni aperte | Scelte che cambiano l'impatto, con la proposta del redattore |
| 5 | Dipendenze e ordine | Rapporto con le altre CR |
| 6 | Rischi e mitigazioni | Riferiti ai rischi R1–R12 dove possibile |
| 7 | Criteri di accettazione | Condizioni verificabili per considerare chiusa la CR |
| 8 | Test di regressione | Test da rieseguire, scelti in base al CIS |

Gli **stati** possibili di una CR sono: *Proposta → Approvata → In lavorazione → Implementata → Chiusa*
(oppure *Respinta*). Ogni revisione del documento incrementa il codice di versione nella scheda.

## 2. Convenzioni dell'analisi d'impatto

**Profondità dell'impatto** (richiesta esplicitamente dal tutor per la CR1):

| Livello | Significato |
|---|---|
| — | Non toccato |
| **B** – Bassa | Modifiche locali e superficiali: import, path, rinomina, spostamento senza cambiare la logica |
| **M** – Media | Alcune funzioni o firme cambiano, la struttura del modulo resta riconoscibile |
| **A** – Alta | Il modulo viene ristrutturato o riscritto; la sua interfaccia cambia |
| **N** – Nuovo | File o modulo nuovo |
| **X** – Eliminato | Il file o il codice viene rimosso |

**Insiemi d'impatto:**

- **SIS** (*Starting Impact Set*): componenti modificati direttamente per realizzare la CR;
- **CIS** (*Candidate Impact Set*): SIS più i componenti raggiunti seguendo le dipendenze del §2.3 del
  documento di reverse engineering: import (**I**), stringhe o chiavi condivise (**S**), file o schema
  CSV (**F**).

Il CIS di ciascuna CR è anche la base per **selezionare i test di regressione** (§8 di ogni CR).

## 3. Vincoli trasversali

Durante tutte e tre le CR restano **invariati**:

1. lo **schema del CSV dei risultati** (colonne prodotte da C8);
2. i **nomi degli operatori e dei sintomi già esistenti**, che compaiono nei CSV e nei nomi dei file.

Rispettando questi due vincoli, l'analisi statistica (C10) e i risultati già committati (D2) restano
compatibili, e l'impatto si ferma ai confini del programma principale. Qualsiasi deroga richiede una
decisione esplicita del gruppo, da registrare nella CR interessata.

## 4. Attività preliminare: test di caratterizzazione

Non è una CR, ma è un prerequisito di tutte e tre: il progetto non ha alcun test automatico (R1) e
l'esecuzione non è deterministica (R2). Prima di modificare il codice si costruisce una test suite di
partenza, descritta in dettaglio nel test plan.

| Aspetto | Contenuto |
|---|---|
| Obiettivo | Fissare il comportamento attuale prima di modificarlo |
| Nuovi artefatti | `tests/` (pytest), dataset ridotti per i test (`tests/fixtures/`), `pytest` nei requisiti di sviluppo |
| Codice di produzione toccato | Nessuno, salvo un eventuale `conftest.py` |
| Test unitari | Le 4 funzioni di distribuzione e i sintomi di C5 su input fissati; inferenza dei gruppi; `clean_data_only`; `get_applicable_operators`; export di C8 (schema delle colonne) |
| Test di proprietà degli operatori | Percentuale di righe modificate, dtype preservato, valori entro i limiti (±5% del range, fattore 0.8–1.2), categorie di destinazione esistenti |
| Golden master end-to-end | Esecuzione completa su un dataset piccolo, con `random.seed` e `np.random.seed` fissati nel test |
| Limite noto | Il golden master dipende dall'ordine in cui viene consumato il generatore casuale. CR1 e CR2 cambiano quell'ordine: il golden master andrà **rigenerato una volta, in modo controllato e documentato**, mentre i test unitari e di proprietà devono restare verdi senza modifiche |

## 5. Vista d'insieme

### 5.1 Matrice componenti × CR (profondità)

| Componente | Test prelim. | CR1 | CR2 | CR3 |
|---|---|---|---|---|
| C1 `main.py` | — | **A** | **A** | — |
| C2 `loader.py` | — | B | — | — |
| C3 `preprocessor.py` | — | M | B | — |
| C4 `dataset_config.json` | — | B | M | — |
| C5 `symptom_calculator.py` → `symptoms/` | — | **A** | B | N |
| C6 `mutation_operator.py` → `mutation/` | — | **A** | B | N |
| C7 `analysis_helpers.py` | — | X / A | — | — |
| C8 `result_handlers.py` → `io/` | — | M | B | — |
| C9 `file_utils.py` → `io/` | — | M / X | B | — |
| C10 `run_analysis_rq1.py` | — | B | B | — |
| `core/` (nuovo) | — | N | B | — |
| CLI e configurazione di esecuzione (nuovi) | — | — | N | B |
| `requirements.txt` | B | M | — | — |
| `README.md` | — | M | A | B |
| `tests/` | N | M | M / N | N |
| D2 `results/` | — | — | — | M (dati) |

### 5.2 Ordine di lavoro

| Passo | Attività | Prerequisiti |
|---|---|---|
| 1 | Test preliminari | — |
| 2 | CR1 | Test preliminari |
| 3a | CR2 | CR1 |
| 3b | CR3 | CR1 (interfacce base); i default dei parametri si integrano dopo la CR2 |

- **C1, C5 e C6** sono toccati sia dalla CR1 sia dalla CR2: farle in sequenza evita di modificare due
  volte gli stessi punti in profondità. La gestione del seed costa poco solo se il generatore casuale è
  già iniettato (CR1).
- La **CR3** dipende dal registro introdotto dalla CR1. Le nuove classi possono essere sviluppate in
  parallelo alla CR2, una volta stabilizzate le interfacce base.
- **Con tre persone:** test preliminari divisi per modulo; la CR1, la più pesante, si divide per
  sotto-sistema (mutazioni / sintomi / runner e I/O) dopo aver concordato insieme le interfacce in
  `base.py` e `registry.py`.

### 5.3 Stima di sintesi

| | Codice esistente toccato | Profondità prevalente | Codice nuovo | Effort relativo |
|---|---|---|---|---|
| Test preliminari | 0% | — | medio (test) | Medio |
| CR1 | ~75–80% | Alta | medio | **Alto** |
| CR2 | ~15–20% | Bassa (dopo la CR1) | ~200–300 LOC | Medio |
| CR3 | ~0% | — | ~100–200 LOC | Basso |
