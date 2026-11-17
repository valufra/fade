# CR3 – Nuovo operatore di mutazione e nuovo sintomo

## 0. Scheda

| Campo | Valore |
|---|---|
| ID | CR3 |
| Titolo | Dimostrazione dell'estendibilità: aggiunta di un operatore di mutazione e di un bias symptom |
| Versione | 0.1 |
| Data | 2026-11-17 |
| Stato | Proposta |
| Priorità | Media |
| Tipo di manutenzione | Perfettiva (estensione funzionale) |
| Autori | Gruppo FADE ISTA 2026/27 |
| Baseline | commit `33a9dd3`, branch `Management` (l'impatto è stimato assumendo la CR1 completata) |
| Dipende da | CR1 (registro e interfacce); CR2 per i parametri di default nella configurazione |
| Abilita | — |

Convenzioni (profondità B/M/A/N/X, SIS/CIS) e vincoli trasversali: [00_panoramica.md](00_panoramica.md).
Componenti e rischi: [01_reverse_engineering.md](../01_reverse_engineering.md).

## 1. Descrizione e motivazione

FADE offre oggi 5 operatori di mutazione e 13 sintomi. Nella versione originale aggiungerne uno richiede
di modificare almeno tre punti del codice (R6). La CR1 introduce un registro proprio per eliminare questo
problema: la CR3 ne è la **verifica sul campo** e, allo stesso tempo, amplia le analisi possibili con il
tool.

**Beneficio atteso:** un operatore e un sintomo in più a disposizione degli utenti, e una prova concreta
(misurabile sul diff) che la nuova architettura è estendibile.

## 2. Cambiamento proposto

**In perimetro:** aggiungere **un operatore di mutazione e un sintomo senza modificare il nucleo**, più i
relativi test, i parametri di default nella configurazione e la documentazione (incluso un breve
"come aggiungere un componente" nel README).

Candidati, da scegliere nel gruppo (§4):

| Tipo | Candidato | Note |
|---|---|---|
| Operatore numerico | *Outlier injection*: alcuni valori portati oltre i percentili estremi | Nessuna dipendenza nuova |
| Operatore categorico | *Category merge*: accorpamento di una categoria rara in un'altra | Riduce la cardinalità e sollecita i sintomi di diversità |
| Sintomo | *Disparate Impact*: rapporto tra P(Y=1) nel gruppo non privilegiato e P(Y=1) nel gruppo privilegiato | Metrica di fairness standard; riusa l'inferenza dei gruppi; va gestito il denominatore nullo |
| Sintomo | *Coefficient of Variation* (solo feature numeriche) | Semplice; `None` per le categoriche, come Kurtosis |
| Operatore scartato | *Iniezione di valori mancanti* | Sconsigliato: introduce NaN dopo il pre-processing e rompe i sintomi (sklearn, query sui gruppi) |

**Fuori perimetro:** modifiche al nucleo, agli operatori e ai sintomi esistenti; nuove colonne nel CSV;
nuove analisi statistiche.

## 3. Analisi d'impatto preliminare

### 3.1 SIS

Solo file nuovi: il modulo del nuovo operatore (in `mutation/`) e quello del nuovo sintomo
(in `symptoms/`).

### 3.2 CIS con profondità

| Comp. | Prof. | Cosa cambia |
|---|---|---|
| Nuovo operatore (`mutation/…`) | **N** | Classe + registrazione |
| Nuovo sintomo (`symptoms/…`) | **N** | Classe + registrazione |
| Nucleo (`core/`, operatori e sintomi esistenti) | **—** | **Nessuna modifica**: è ciò che la CR deve dimostrare |
| Configurazione (da CR2) | **B** | Parametri di default del nuovo operatore |
| `io/` e schema CSV | **—** | Nuovi valori nelle colonne `mutation_type` e `symptom_name`; nessuna colonna nuova |
| C10 `run_analysis_rq1.py` | **—** (da verificare) | Elabora operatori e sintomi in modo generico (`groupby`, ciclo sui `symptom_name`). Da verificare il trattamento di `inf` (già filtrati) e `None` (righe scartate da C8) |
| D2 `results/` | **M** (dati) | I risultati esistenti non contengono il nuovo operatore e il nuovo sintomo: le analisi RQ1 vanno rigenerate o presentate separatamente |
| `README.md` | **B** | Elenchi di operatori e sintomi; guida "come aggiungere un componente" |
| `tests/` | **N** | Test unitari dei nuovi componenti; test della scoperta tramite registro |

Dipendenze seguite per il CIS: registro → runner (scoperta automatica); nomi dei nuovi componenti → CSV
→ C10 (stringhe e file).

### 3.3 Dimensione

Circa lo **0% del codice esistente**; ≈100–200 LOC nuove (componenti e test).

## 4. Decisioni aperte

| Questione | Opzioni | Proposta del redattore |
|---|---|---|
| Quale operatore | Outlier injection oppure Category merge | Category merge: copre il ramo categorico, dove FADE ha già più operatori e si confronta meglio |
| Quale sintomo | Disparate Impact oppure Coefficient of Variation | Disparate Impact: più rilevante per la fairness e collegato ai sintomi di gruppo già presenti |
| Risultati in `results/` | Rigenerare le analisi RQ1 con i nuovi componenti oppure presentarli a parte | Presentarli a parte, per non alterare i risultati originali |

## 5. Dipendenze e ordine

- **Prima:** CR1 (registro e interfacce base stabili).
- **In parallelo o dopo:** CR2. Le classi si possono scrivere appena le interfacce sono stabili; i
  parametri di default si integrano nella configurazione quando la CR2 è completata.

## 6. Rischi e mitigazioni

| Rischio | Rif. | Mitigazione |
|---|---|---|
| Serve toccare il nucleo per aggiungere i componenti | R6 | È un difetto di progettazione della CR1: va documentato come deviazione e corretto lì, non aggirato |
| Valori anomali del nuovo sintomo (divisione per zero, `inf`, `None`) | — | Casi limite coperti da test unitari; verifica del comportamento di C8 e C10 |
| Aumento del tempo di esecuzione | — | Ogni nuovo operatore aggiunge F × F calcoli di sintomi per feature applicabile; ogni nuovo sintomo F × (numero di mutazioni). Possibilità di disabilitarli da configurazione (CR2) |

## 7. Criteri di accettazione

1. **Il diff della CR contiene solo file nuovi, configurazione, test e documentazione**: nessun file del
   nucleo né degli operatori e sintomi esistenti è modificato (verificabile con `git diff --stat`).
2. Il nuovo operatore e il nuovo sintomo sono scoperti automaticamente dal registro e compaiono nel CSV
   dei risultati con il loro nome.
3. Test unitari dei nuovi componenti, inclusi i casi limite (denominatore nullo, feature non
   applicabile).
4. L'analisi statistica (C10) gira senza errori su risultati che contengono i nuovi componenti.
5. Il README elenca i nuovi componenti e spiega come aggiungerne altri.

## 8. Test di regressione

Il nucleo non cambia, quindi il CIS è minimo:

| Test | Motivo |
|---|---|
| Test del registro | Verificano che i componenti esistenti siano ancora tutti registrati e che si aggiungano i nuovi |
| Schema del CSV (C8) | Nuovi valori nelle colonne esistenti |
| Golden master end-to-end | Con i nuovi componenti disabilitati deve restare identico; con quelli abilitati va esteso |
| Analisi statistica (C10) | Trattamento dei nuovi nomi e dei valori anomali |

I test unitari e di proprietà dei componenti esistenti sono fuori dal CIS.
