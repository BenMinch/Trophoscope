# TrophoScope

Predicts whether a giant virus infects an **autotroph** or a **heterotroph**.

```bash
source ../glm2_experiment/env.sh        # or any env with the deps below
$PY trophoscope2.py -d my_genomes/ -H ../resources/Pfam.hmm -o predictions.csv
```

`-m` defaults to the bundled `model/`. Nothing else is required: gene calling
(pyrodigal) and Pfam search (pyhmmer, GA cutoffs) run in-process, so `prodigal`
and `hmmsearch` need not be on PATH. No GPU, no language model.

Deps: `python>=3.11, lightgbm, scikit-learn, scipy, pandas, numpy, biopython,
pyrodigal, pyhmmer, joblib`.

## Accuracy

Measured on 44 cultured reference genomes (21 autotroph / 23 heterotroph hosts),
which are **not** in the training data — the model is trained purely on
endogenous viral regions (GEVEs).

| | accuracy | balanced | ROC-AUC | autotroph recall | heterotroph recall |
|---|---|---|---|---|---|
| **as shipped** | **43/44 = 97.7%** | 97.8% | 0.996 | 21/21 | 22/23 |
| honest estimate (threshold held out too) | 41/44 = 93.2% | 93.3% | 0.996 | 20/21 | 21/23 |
| previous model, for comparison | 34/44 = 77.3% | 76.2% | 0.979 | 11/21 | 23/23 |

**Which number to quote: 93.2%.** The 97.7% row uses a decision threshold
(0.1441) that was chosen with all 44 genomes visible. The 93.2% row re-chooses
the threshold per host genus from the *other* genera only, so nothing it reports
has seen the genome it scores. The gap is one genome either way — at n=44 a
single genome is 2.3%.

The one persistent error is *Cedratvirus kamchatka* (p=0.208, called Autotroph).

## What changed versus the original model

The original was not under-powered — on this cohort it already had ROC-AUC 0.979
while scoring 77.3%, and *every* error was an autotroph scored just below 0.5,
with no heterotroph ever wrong. The ranking was nearly right; the operating point
was wrong, because training and inference were in different feature domains:

| cohort | median Pfam domains per scored unit |
|---|---|
| GEVE training rows | 12 |
| **complete genome (old inference)** | **300** |
| 12 kb window (new inference) | 10 |

TF-IDF vectors are L2-normalised, so a 300-domain document is simply not the
regime the model was fitted in. TrophoScope 2 therefore scores a genome the way
training rows were built: cut it at gene boundaries into ~12 kb windows (GEVE
median; 5 kb floor, ≥3 genes), score each window, and average. The threshold is
0.1441, read from `model/model_metadata.json` — **not** 0.5.

Everything else — TNF features, Pfam TF-IDF, LightGBM, the 54,117 training
GEVEs — is unchanged from your pipeline. TNF is bit-identical to the original
implementation.

## Scope

**Protist-infecting giant viruses.** The training data is entirely protist-host
GEVEs across 289 host families. Vertebrate- and insect-host viruses
(poxviruses, ranaviruses, ascoviruses, iridoviruses) are out of scope; on those
the model is not merely uncertain, it is confidently wrong, calling 52 of 59
such genomes Autotroph.

Each prediction therefore carries `pfam_oov_frac` (fraction of Pfam domains
absent from the training vocabulary) and an `out_of_scope_warning` raised above
0.10. Measured on the 59 out-of-scope and 44 in-scope references, that catches
~93% of out-of-scope genomes and false-alarms on ~16% of valid ones
(ROC-AUC 0.931). The threshold was picked on those same genomes, so treat the
rates as indicative. `ood_tnf_z` is reported but does not raise the flag — it
discriminates poorly (AUC 0.651).

**This is a backstop, not a gate.** It will miss genuinely novel out-of-scope
input. Knowing your host lineage remains the real safeguard.

## Output

`predictions.csv` — one row per genome: `prob_autotroph`, `prediction`,
`n_windows`, `n_genes`, `frac_windows_autotroph`, `total_bp`, `ood_tnf_z`,
`pfam_oov_frac`, `out_of_scope_warning`.

`predictions.windows.csv` — per-window probabilities. Worth checking when
`frac_windows_autotroph` sits near 0.5: windows disagreeing means the genome is
genuinely borderline rather than confidently called.

## Caveats

- n=44 references; one genome is ±2.3%. Treat differences under ~3 points as noise.
- The 54k GEVE labels come from host taxonomy, not measured trophic mode.
- Host-genus-held-out splitting does not guarantee *viral* homology separation,
  so some optimism may remain from related viruses across folds.
- Mixotroph hosts are excluded from training and evaluation; the model emits a
  binary call and will force such genomes to one side.
- Two reference labels were corrected on host biology (`NC_038553`
  Heterosigma akashiwo virus 01, `NC_028094` Chrysochromulina ericina virus).
  See `../glm2_experiment/README.md` — the second is worth re-checking, as the
  model argues against it.

Full experimental record, ablations and the gLM2 evaluation:
`../glm2_experiment/README.md`.
