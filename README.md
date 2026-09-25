# Progetto PTQ su Qwen2.5-Coder

Replica in scala ridotta dello studio sull'impatto della quantizzazione post-training (PTQ) su LLM per la generazione di codice.


**Domanda di ricerca**: quanto perde in accuratezza (pass@1 su HumanEval) un modello linguistico
per il codice quando viene compresso a bit sempre più bassi (8, 4, 3, 2 bit)? Il calibration
dataset usato per la quantizzazione fa differenza?

## Setup

- **Modello**: Qwen2.5-Coder-1.5B
- **Quantizzazione**: llama.cpp (k-quants), livelli Q8_0 / Q4_K_M / Q3_K_M / Q2_K
- **Calibration dataset**: Random (`allenai/c4`), Code (`codeparrot-clean-valid`), Mixed
  (codice + Stack Overflow)
- **Benchmark**: HumanEval (164 problemi, pass@1)
- **Hardware**: MacBook Air M2, 8GB RAM

## Struttura del progetto

```
notebooks/
├── 01_prepare_calibration_data.ipynb   # costruzione dei 3 calibration dataset
├── 02_quantize_models.ipynb            # generazione imatrix + quantizzazione (12 modelli)
├── 03_run_generation.ipynb             # generazione dei completamenti su HumanEval
├── 04_evaluate.ipynb                   # calcolo del pass@1
├── 05_analyze_results.ipynb            # grafici e tabelle riassuntive
└── 06_conclusions.ipynb                # discussione, conclusioni e relazione finale

calibration_data/    # i 3 file di calibrazione (random.txt, mixed.txt, code.txt)
results/             # risultati (CSV, generazioni, grafici)
```

> Nota: le cartelle `models/`, `imatrix/` e `llama.cpp/` non sono incluse nel repository
> (file troppo pesanti per Git) — vanno rigenerate localmente eseguendo i notebook in ordine.

## Risultati principali

| Calibration | Q8_0 | Q4_K_M | Q3_K_M | Q2_K |
|---|---|---|---|---|
| baseline (fp16) | — | — | — | 0.3902 |
| random | 0.3780 | 0.3415 | 0.3659 | 0.2134 |
| mixed | 0.3780 | 0.3171 | 0.3537 | 0.2134 |
| code | 0.3780 | 0.3415 | 0.3537 | 0.2073 |

Analisi completa, grafici e conclusioni nel notebook `06_conclusions.ipynb`.

## Come riprodurre

1. Attivare l'ambiente virtuale ed eseguire i notebook nell'ordine `01` → `06`
2. Il notebook `02` richiede `llama.cpp` compilato con supporto Metal
3. Il notebook `03` richiede `llama-cpp-python`
4. Il notebook `04` richiede la libreria `human-eval`
