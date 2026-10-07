# Progetto PTQ su Qwen2.5-Coder

Replica in scala ridotta dello studio sull'impatto della quantizzazione post-training (PTQ) su LLM per la generazione di codice.

**Domanda di ricerca**: quanto perde in accuratezza (pass@1 su HumanEval) un modello linguistico
per il codice quando viene compresso a bit sempre più bassi (8, 4, 3, 2 bit)? Il calibration
dataset usato per la quantizzazione fa differenza, e quanto rispetto a non usarne nessuno?
Quanto spazio si risparmia ai vari livelli?

## Setup

- **Modello**: Qwen2.5-Coder-1.5B
- **Quantizzazione**: llama.cpp (k-quants), livelli Q8_0 / Q4_K_M / Q3_K_M / Q2_K
- **Calibration dataset**: Random (`allenai/c4`), Code (`codeparrot-clean-valid`), Mixed
  (codice + Stack Overflow)
- **Senza calibration dataset** (`none`): stessi livelli, ma `llama-quantize` lanciato senza `--imatrix`,
  usato come ulteriore termine di confronto
- **Benchmark**: HumanEval (164 problemi, pass@1, decoding greedy, un campione per problema)
- **Hardware**: MacBook Air M2, 8GB RAM

In totale 17 modelli: 1 baseline fp16 + 12 con calibration (3 dataset × 4 livelli) + 4 senza calibration.

## Struttura del progetto

```
notebooks/
├── 01_prepare_calibration_data.ipynb   # costruzione dei 3 calibration dataset
├── 02_quantize_models.ipynb            # imatrix + quantizzazione (12 modelli con calibration + 4 senza) e salvataggio dimensioni
├── 03_run_generation.ipynb             # generazione dei completamenti su HumanEval (17 modelli)
├── 04_evaluate.ipynb                   # calcolo del pass@1
├── 05_analyze_results.ipynb            # grafici e tabelle riassuntive
└── 06_conclusions.ipynb                # discussione, conclusioni e relazione finale

calibration_data/    # i 3 file di calibrazione (random.txt, mixed.txt, code.txt)
results/             # risultati (CSV, generazioni, grafici, model_sizes.csv con le dimensioni dei modelli)
```

> Nota: le cartelle `models/`, `imatrix/` e `llama.cpp/` non sono incluse nel repository
> (file troppo pesanti per Git) — vanno rigenerate localmente eseguendo i notebook in ordine.

## Risultati principali

Baseline fp16: pass@1 = **0.3902** (64 problemi su 164)

| Calibration | Q8_0 | Q4_K_M | Q3_K_M | Q2_K |
|---|---|---|---|---|
| random | 0.3780 | 0.3415 | 0.3659 | 0.2134 |
| mixed | 0.3780 | 0.3171 | 0.3537 | 0.2134 |
| code | 0.3780 | 0.3415 | 0.3537 | 0.2073 |
| none (senza calibration) | 0.3780 | 0.3598 | 0.2927 | 0.1098 |

Sintesi:

- **Accuratezza contro bit**: la perdita è trascurabile a 8 bit (-3.1%), moderata a 4 e 3 bit con calibrazione
  (dal -6% al -19%) e drastica a 2 bit (circa -46% con calibrazione, -72% senza).
- **Calibration dataset**: tra `random`, `mixed` e `code` le differenze sono di al massimo 4 problemi su 164,
  quindi *quale* dataset si usa conta poco. Rispetto a `none`, invece, *avere* una calibrazione conta molto a
  3 e 2 bit (+10/+12 problemi a Q3_K_M, +16/+17 a Q2_K), non cambia nulla a 8 bit, e a 4 bit `none` è
  leggermente migliore (3-7 problemi, al limite del rumore).
- **Spazio contro accuratezza**: il punto più conveniente è Q4_K_M (-68% di spazio, robusto anche senza
  calibrazione); Q3_K_M è valido solo con una imatrix; Q2_K è da evitare su un modello così piccolo.

### Dimensione dei modelli

La dimensione dipende solo dal livello di quantizzazione, non dalla calibration (valori da
`results/model_sizes.csv`; i "MB" sono in realtà MiB).

| Livello | Dimensione (MB) | Dimensione (GB) | Riduzione vs fp16 |
|---|---|---|---|
| fp16 (baseline) | 2950 | 2.88 | — |
| Q8_0 | 1570 | 1.53 | -46.8% |
| Q4_K_M | 940 | 0.92 | -68.1% |
| Q3_K_M | 786 | 0.77 | -73.4% |
| Q2_K | 645 | 0.63 | -78.1% |

Analisi completa, grafici e conclusioni nel notebook `06_conclusions.ipynb`.

> Con 164 problemi e un solo campione per problema, 1 problema vale circa 0.61 punti percentuali:
> differenze di 1-3 problemi vanno trattate come rumore.

## Come riprodurre

1. Attivare l'ambiente virtuale ed eseguire i notebook nell'ordine `01` → `06`
2. Il notebook `02` richiede `llama.cpp` compilato con supporto Metal; genera le imatrix e i 12 modelli con
   calibration, poi (sezione 3b) i 4 modelli `none` quantizzati senza `--imatrix`, e crea `results/model_sizes.csv`
3. Il notebook `03` richiede `llama-cpp-python`
4. Il notebook `04` richiede la libreria `human-eval`
