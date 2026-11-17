# CR2 – CLI non interattiva, file di configurazione, seed riproducibile

## 0. Scheda

| Campo | Valore |
|---|---|
| ID | CR2 |
| Titolo | Interfaccia a riga di comando non interattiva, file di configurazione dell'esecuzione e seed riproducibile |
| Versione | 0.1 |
| Data | 2026-11-17 |
| Stato | Proposta |
| Priorità | Media |
| Tipo di manutenzione | Adattativa / perfettiva (adattamento dell'interfaccia) |
| Autori | Gruppo FADE ISTA 2026/27 |
| Baseline | commit `33a9dd3`, branch `Management` (l'impatto è stimato assumendo la CR1 completata) |
| Dipende da | CR1 |
| Abilita | CR3 (parametri di default dei nuovi componenti) |

Convenzioni (profondità B/M/A/N/X, SIS/CIS) e vincoli trasversali: [00_panoramica.md](00_panoramica.md).
Componenti e rischi: [01_reverse_engineering.md](../01_reverse_engineering.md).

## 1. Descrizione e motivazione

Oggi FADE si usa solo in modo interattivo: `main.py` chiede dataset, target e positive label con
`input()`. Non è quindi possibile:

- lanciarlo da script, in batch su più dataset o in pipeline automatiche;
- scegliere quali operatori e sintomi eseguire, né cambiarne i parametri senza modificare il codice (R7);
- ripetere un'esecuzione e ottenere gli stessi risultati: non c'è un seed e le sorgenti di casualità
  sono globali o nascoste (R2), quindi i risultati non sono riproducibili né confrontabili.

**Beneficio atteso:** esecuzioni scriptabili, configurabili e riproducibili, con la configurazione usata
salvata accanto ai risultati (tracciabilità). La riproducibilità rende anche più solidi i test di
regressione delle CR successive.

## 2. Cambiamento proposto

**In perimetro:**

1. **Entry point a riga di comando** con `argparse` (nessuna nuova dipendenza). Parametri: dataset,
   target, positive label, operatori e sintomi da abilitare, cartella di output, seed, livello di
   verbosità, file di configurazione.
2. **File di configurazione dell'esecuzione** con gli stessi campi della CLI più i parametri dei singoli
   operatori e sintomi (quelli dichiarati dai componenti nella CR1). Precedenza:
   *default dei componenti < file di configurazione < argomenti da riga di comando*.
3. **Validazione degli input:** target esistente nel dataset, operatori e sintomi presenti nel registro,
   parametri del tipo atteso. In caso di errore, messaggio chiaro e codice di uscita diverso da zero.
4. **Seed unico** propagato a tutte le sorgenti di casualità: generatore iniettato negli operatori,
   `random_state` di `mutual_info_classif`/`mutual_info_regression`, `df.sample()`.
5. **Tracciabilità:** seed e configurazione effettiva salvati nella cartella dei risultati.
6. **README:** sezioni Installation, Usage e Configuration riscritte.

**Fuori perimetro:**

- nuovi operatori o sintomi (CR3);
- modifiche allo schema CSV (vincolo trasversale);
- interfacce grafiche o servizi web.

## 3. Analisi d'impatto preliminare

### 3.1 SIS

C1 `main.py` (sostituzione degli `input()`), nuovo modulo CLI, nuovo modulo di configurazione
dell'esecuzione.

### 3.2 CIS con profondità (assumendo la CR1 completata)

| Comp. | Prof. | Cosa cambia |
|---|---|---|
| C1 `main.py` / nuovo `cli.py` | **A** / **N** | `input()` sostituiti da argomenti; validazione; eventuale modalità interattiva di fallback (§4) |
| Nuovo modulo di configurazione + schema | **N** | Caricamento, validazione e merge delle configurazioni |
| C4 `dataset_config.json` | **M** | Integrato nel nuovo file di configurazione (`ignore_features` per dataset) oppure mantenuto separato (§4) |
| C3 `preprocessor.py` | **B** | Riceve `ignore_features` come parametro invece di leggere il JSON da sé |
| `core/runner.py`, `core/context.py` (da CR1) | **B** | Ricevono configurazione e `numpy.random.Generator` dall'esterno |
| Operatori (ex C6) | **B** | RNG già iniettato dalla CR1. *Senza la CR1 sarebbe M: ogni chiamata a `random.*`/`np.random.*` e ogni `df.sample()` andrebbe modificata* |
| Sintomi (ex C5) | **B** | Passaggio del `random_state` alle funzioni di sklearn: senza, la Mutual Information resta non deterministica anche con il seed |
| `io/` (ex C8, C9) | **B** | Cartella di output come parametro; salvataggio di seed e configurazione accanto ai risultati |
| C10 `run_analysis_rq1.py` | **B** | Argomenti per le cartelle di input e output (oggi `results/` fisso). Opzionale ma coerente con la CR |
| `README.md` | **A** | Installation, Usage e Configuration riscritte |
| `tests/` | **M / N** | Test della CLI, del merge delle configurazioni e del determinismo |

Dipendenze seguite per il CIS: CLI → runner → operatori e sintomi (import, passaggio di RNG e
parametri); configurazione → C3 e C4 (file); `io/` → cartella dei risultati (file).

### 3.3 Dimensione

Circa il **15–20% del codice esistente**, toccato con profondità prevalentemente bassa, più
≈200–300 LOC nuove (CLI, configurazione, test).

## 4. Decisioni aperte

| Questione | Opzioni | Impatto | Proposta del redattore |
|---|---|---|---|
| Formato del file di configurazione | JSON (nessuna dipendenza) oppure YAML (richiede `pyyaml`) | YAML aggiunge una dipendenza ma è più leggibile e ammette commenti | JSON, coerente con `dataset_config.json` |
| Modalità interattiva | Eliminarla oppure mantenerla come fallback quando mancano gli argomenti obbligatori | Mantenerla aggiunge codice e test, ma preserva l'uso documentato oggi | Mantenerla come opzione esplicita (`--interactive`) |
| `dataset_config.json` | Integrarlo nel file di esecuzione oppure tenerlo separato | Tenerlo separato limita l'impatto su C3 e C4 | Tenerlo separato |
| Parametri di C10 | Includerli in questa CR oppure no | Impatto basso su C10 | Includerli |

## 5. Dipendenze e ordine

- **Prima:** CR1. Se la CR2 venisse fatta prima, la gestione del seed toccherebbe in profondità C5 e C6
  e quel lavoro andrebbe poi rifatto dalla CR1.
- **In parallelo:** la CR3 può procedere con le nuove classi; i loro parametri di default si integrano
  nella configurazione al termine della CR2.

## 6. Rischi e mitigazioni

| Rischio | Rif. | Mitigazione |
|---|---|---|
| Determinismo incompleto (sorgenti nascoste: sklearn, `df.sample`, WordNet) | R2 | Il test "stesso seed → stessi risultati" è un criterio di accettazione; censimento delle sorgenti di casualità |
| Precedenze di configurazione ambigue | — | Regola unica (default < file < CLI) documentata e coperta da test |
| Perdita della modalità d'uso documentata | — | Decisione esplicita sulla modalità interattiva (§4); README aggiornato |
| Dipendenza forte dalla CR1 | — | Ordine di lavoro vincolato (§5) |

## 7. Criteri di accettazione

1. Un'esecuzione completa si lancia da riga di comando senza alcun `input()`; `--help` documenta tutti i
   parametri.
2. Gli stessi parametri si possono fornire da file di configurazione, e la precedenza
   default < file < CLI è verificata da test.
3. **Due esecuzioni con lo stesso seed e la stessa configurazione producono risultati identici**
   (contenuto dei CSV uguale, a parte eventuali timestamp nei nomi dei file).
4. Input non validi (target inesistente, operatore non registrato, parametro di tipo errato) producono un
   messaggio chiaro e un codice di uscita diverso da zero.
5. Seed e configurazione effettiva sono salvati insieme ai risultati.
6. Schema del CSV e nomi di operatori e sintomi invariati.
7. Il README spiega installazione, uso e configurazione con esempi eseguibili.

## 8. Test di regressione

| Test | Motivo |
|---|---|
| Golden master end-to-end | Il runner riceve ora configurazione e RNG dall'esterno; con il seed diventa anche un test di determinismo |
| Proprietà degli operatori | Passaggio dell'RNG e dei parametri |
| Unitari dei sintomi basati su sklearn (Mutual Information) | Introduzione del `random_state` |
| Pre-processing (C3) | `ignore_features` passato come parametro |
| Schema del CSV (C8) | Cartella di output parametrica |
| Analisi statistica (C10) | Solo se si aggiungono i suoi parametri |

I test unitari dei sintomi deterministici e quelli del registro (fuori dal CIS) non vanno rieseguiti
obbligatoriamente, ma si consiglia comunque l'esecuzione completa prima del merge, dato il costo basso.
