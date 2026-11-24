# CONTEXT: project state (single source of truth)

> Read by humans and agents before every session. **Rules** live in [CONVENTIONS.md](CONVENTIONS.md), the **people/timeline plan** in [TEAM_PLAN.md](TEAM_PLAN.md).
> Sections 1–7 (problem, facts, labels, decisions, contracts, analysis definitions) are **shared**: change them only in their own small PR (CONVENTIONS.md §3).
> Last updated: 2026-10-30 by Sidharth (scaffolding session, see `logs/2026-10-30_1945_sidharth_setup.md`).

---

## 1. Problem and objectives

**Problem.** Ziegler et al. (2021) showed that patients who become severely ill with COVID-19 mount a weaker early interferon response in the nasal epithelium. We ask whether the nasal-swab single-cell transcriptome can **classify a patient as mild/moderate vs severe**. If it can, we ask **which cell types carry the signal**, **which genes drive it**, what those genes do, and which of them are plausible **drug targets**. The mentor asked us to go beyond diagnosis because scRNA-seq costs about $2,000 per sample.

| # | Objective | Phase |
|---|---|---|
| O1 | Map the cell landscape (QC, PCA/UMAP, clustering vs author annotations) | sprint |
| O2 | Test whether cell-type **composition** differs between control / mild / severe | sprint (first pass) |
| O3 | Build **patient-level classifiers** for mild vs severe with honest CV and a permutation baseline | sprint (first pass), refined after |
| O4 | Find **which cell types carry the predictive signal** | post |
| O5 | Find **key severity genes** (pseudobulk DE + model importance), their **roles** (enrichment, literature) and **drug targets** (HPA, DGIdb, Open Targets, GTEx) | post |
| O6 | **Perturbation analysis**: how predicted severity responds when key genes are shifted in silico (§7.2) | post |
| O7 | **Sensitivity analysis**: robustness to severity definition, features, QC, sampling (§7.3) | post |
| O8 | **Validate** against Ziegler et al. findings and, if time allows, an independent cohort (Chua et al. 2020) | post |

**Research questions**
- **RQ1** Do cell-type proportions differ between healthy controls, mild and severe COVID-19 patients?
- **RQ2** Can mild vs severe be predicted from patient-level features better than a label-permutation null? Which feature type works best: composition, pseudobulk expression, or both?
- **RQ3** Which cell types carry the predictive signal (single-cell-type models, ablation)?
- **RQ4** Which genes differ between mild and severe within those cell types (pseudobulk DE, BH-FDR), what pathways are they in, and what are their roles?
- **RQ5** If a key gene's expression is shifted toward mild levels, how much does predicted severity change, and is that change larger than for expression-matched random genes?
- **RQ6** Are the findings robust to how severity is defined and to analysis choices?
- **RQ7** Which key genes are druggable, have existing compounds, prior COVID-19 evidence, and acceptable tissue specificity?

---

## 2. Dataset facts (verified 2026-10-30, read-only, h5py 3.14 + anndata 0.13.4)

**File:** `data/raw/zeigler2021_nasal.h5ad`, 268,167,914 bytes, md5 `2fcf7a8a3a52ec91e824f2041679195f`. CELLxGENE schema 7.1.0, `uns['title']` = "Nasopharynx". The filename's "zeigler" spelling is intentional: it is the contract path, so keep it.

| Item | Value |
|---|---|
| Cells × genes | **32,588 × 31,738**. Earlier docs said ~32.9k genes; the file has 31,738. 10 are SARS-CoV-2 genes (`var['feature_reference'] == 'NCBITaxon:2697049'`: E, M, N, ORF10, ORF3a, ORF6, ORF7a, ORF7b, ORF8, S). |
| `var` index | Ensembl IDs; symbols in `var['feature_name']` (31,727 unique, so a few duplicates) |
| **Raw counts** | **`adata.raw.X`**: CSR, float32, integer-valued (max 3,601). Same shape as `X`. |
| `X` | log-normalised (approx. log1p CP10k but not exactly reproducible). **Don't use it for DE or as counts.** Re-normalise from `raw.X`. |
| `layers` | empty |
| `obsm` | `X_pca` (100 dims), `X_umap` (authors' embedding) |
| Author HVGs | `var['mvp.variable']`: 3,384 genes |
| Cell QC already applied by authors | genes/cell: min 113, 5th pct 233, median 1,146; UMIs/cell: min 184, median 1,967; `Percent_Mitochondrial` max 49.9 (an author cut-off of about 50% is likely) |
| Assay / tissue | Seq-Well S3 / nasopharynx; `is_primary_data` all True |

**Key `obs` columns**

| Column | Meaning | Values |
|---|---|---|
| `donor_id` | **patient ID, the sample unit** | 58 donors, e.g. `COVID19_Participant34`, `Control_Participant10` |
| **`Cohort_Disease_WHO_Score`** | **group label (primary)** | `COVID19_WHO_6-8` (21 donors, 12,909 cells), `Control_WHO_0` (15, 8,874), `COVID19_WHO_1-5` (14, 5,164), `Control_WHO_8` (6, 3,335), `ConvalescentCOVID19_WHO_0` (2, 2,306) |
| `Peak_Respiratory_Support_WHO_Score` | peak WHO ordinal score (stored as object; cast to int) | per donor: mild = WHO 1 (4), 3 (2), 4 (3), 5 (5); severe = WHO 6 (1), 7 (4), 8 (16); controls 0 or 8 |
| `Coarse_Cell_Annotations` | author cell types, 18 | identical to `Annotation`. Cells: Ciliated 10,059; Squamous 5,250; Developing Ciliated 3,854; Secretory 3,633; Goblet 2,807; Basal 1,691; T 1,475; Erythroblasts 986; Macrophages 903; Deuterosomal 583; Developing Secretory and Goblet 406; Ionocytes 399; Mitotic Basal 266; Dendritic 142; B 71; Enteroendocrine 41; Plasmacytoid DCs 13; Mast 9 |
| `Detailed_Cell_Annotations` | author subtypes, 39 | each maps to exactly one coarse type |
| `cell_type` | CELLxGENE ontology label (18) | **not** the same grouping as the author coarse labels; don't use for analysis |
| `disease` | CELLxGENE disease | COVID-19 / normal / respiratory failure (= `Control_WHO_8`) / long COVID-19 (= convalescent). **Don't use it as the severity label.** |
| `SARSCoV2_PCR_Status` | pos / neg | per donor |
| `SingleCell_SARSCoV2_RNA_Status` | per-cell viral RNA call | neg 29,684 / amb 2,489 / pos 415 |
| `SARSCoV2_Total_Corrected` | per-cell viral UMIs | 0 to 12,612 |
| `Bloody_Swab` | Yes/No per donor | 4 donors Yes: COVID19_Participant15 (severe), COVID19_Participant32 (severe), Control_Participant10, Control_Participant8 |
| `sex`, `development_stage` | per donor (age by decade) | COVID mild 6F/8M, severe 10F/11M, control 9F/6M |

**Facts that constrain the analysis**
1. **Small n.** 35 COVID patients (14 mild vs 21 severe). The patient is the sample unit.
2. **Unequal depth per donor.** Cells per donor range from 18 to 2,140. 8 of 35 COVID donors have fewer than 100 cells (COVID19_Participant10 has 18).
3. **Uneven cell-type coverage.** Among the 35 COVID donors, the number with **at least 10 cells** of a type is: Ciliated 30, Secretory 24, Squamous 21, Goblet 21, Developing Ciliated 18, T 17, Deuterosomal 11, Basal 10, Ionocytes 8, Macrophages 7, all others ≤3. Split by group (mild / severe donors with ≥ 10 cells): Ciliated 14/16, Secretory 9/15, Goblet 9/12, Squamous 9/12, T 9/8, Developing Ciliated 5/13, Deuterosomal 3/8, Macrophages **1/6**, Basal 5/5. Per-cell-type models and DE are therefore realistic only for about 5 types, and each must state its own n. **Macrophage/immune-specific claims about mild vs severe are not testable at donor level.**
4. **Erythroblasts are blood contamination.** 973 of 986 come from Control_Participant10 (577) and COVID19_Participant32 (396), both bloody swabs.
5. **The 23 "controls" are mixed:** 15 healthy (WHO 0) + 6 non-COVID ICU respiratory failure (WHO 8) + 2 convalescent.

---

## 3. Getting the data

The raw h5ad is **not in git** (GitHub's limit is 100 MB per file; this one is 268 MB). It is attached to the GitHub Release **`data-v1`** of this (private) repo.
1. Browser: GitHub repo → Releases → `data-v1` → download `zeigler2021_nasal.h5ad` into `data/raw/`.
2. Fallback (public): `curl -L -o data/raw/zeigler2021_nasal.h5ad https://datasets.cellxgene.cziscience.com/6a6db93e-901f-4054-90e0-14af5207978a.h5ad`
3. Check: `python -c "import hashlib;print(hashlib.md5(open('data/raw/zeigler2021_nasal.h5ad','rb').read()).hexdigest())"` must print `2fcf7a8a3a52ec91e824f2041679195f`. If the CELLxGENE copy gives a different md5, CELLxGENE has re-processed the file. Use the release copy.

`data/external/` (DGIdb 2026-10 v5.0.13 `interactions.tsv`, `categories.tsv`; HPA `proteinatlas.tsv.zip`) **is committed** to git. Sources and checksums are in [data/external/SOURCES.md](data/external/SOURCES.md).

---

## 4. Labels (who is mild, severe, control)

Defined once in `scripts/common.py` (`GROUP_MAP`). Everyone uses these names.

| `group` (ours) | Author `Cohort_Disease_WHO_Score` | Donors | Used for |
|---|---|---|---|
| `control` | `Control_WHO_0` | 15 | healthy reference: COVID-vs-control sanity check, composition tests, "shift toward control" |
| `mild` | `COVID19_WHO_1-5` | 14 | **primary task, class 0** |
| `severe` | `COVID19_WHO_6-8` | 21 | **primary task, class 1 (positive)** |
| `control_icu` | `Control_WHO_8` | 6 | **specificity check only**: non-COVID respiratory failure. Does the "severe" signature just mean "in the ICU"? |
| `convalescent` | `ConvalescentCOVID19_WHO_0` | 2 | **excluded** from all analyses (n=2) |

- `severe` (binary): 1 = severe, 0 = mild, NA for everyone else.
- `who_peak` (int): `Peak_Respiratory_Support_WHO_Score`. It is used for the sensitivity analysis (§7.3) and the ordinal check.

---

## 5. Methodological decisions (and why)

1. **The patient is the sample unit.** All tests, CV splits and p-values that support claims are at donor level. Cells from one donor are never split across train and test. Cell-level p-values are never reported as findings (Squair et al. 2021: they are badly inflated).
2. **Cell-type labels = the authors' annotations** (`Coarse_Cell_Annotations` for features and DE; `Detailed_` for exploration only). Our Leiden clustering is a check (agreement with author labels via ARI), not a re-annotation. *Why:* there's no time to re-annotate, and the authors' labels are validated in the paper.
3. **QC = keep the authors' QC'd cells by default** and add QC flags/metrics. Extra filtering must be justified and is tested as a sensitivity setting. *Why:* the authors already filtered the data, and stricter filtering removes cells from donors who already have few.
4. **Normalisation:** counts → `normalize_total(target_sum=1e4)` → `log1p`, recomputed from `raw.X`. SARS-CoV-2 genes are excluded from host features, HVGs and DE.
5. **Composition:** per-donor proportions over coarse types excluding Erythroblasts. CLR transform with pseudocount 0.5 on counts. Kruskal–Wallis (control/mild/severe) and Mann–Whitney U (mild vs severe; control vs COVID). BH-FDR across cell types within each comparison.
6. **Classification (sprint spec):**
   - Task: mild (0) vs severe (1), n = 14 + 21 = 35. Sanity task: control (15) vs COVID (35).
   - Feature sets: `comp` = CLR composition (17 types). `pb_all` = whole-sample pseudobulk logCPM, top HVGs chosen **inside each training fold**, then PCA **fit inside the fold**. Post-presentation adds `pb_<celltype>` and `comb`.
   - Models: L2 logistic regression, linear SVM, random forest (500 trees). `class_weight="balanced"`, **fixed hyperparameters** in the sprint, so no tuning and no nested CV is needed. Tuning later is allowed only with nested CV.
   - CV: **primary = repeated stratified 5-fold, 10 repeats (seeds 0–9)**, AUROC on pooled out-of-fold (OOF) predictions per repeat, reported as median [min–max]. **Secondary = leave-one-patient-out (LOPO).** *Why not LOPO as primary:* with n=35, LOO shifts the training-set class balance against the held-out patient and can bias AUROC downward.
   - Every preprocessing step (scaling, HVG selection, PCA, feature selection) is fit **inside the training fold only**.
   - Metrics: AUROC, balanced accuracy (threshold 0.5), confusion matrix.
   - **Permutation test:** shuffle labels across patients and rerun the whole CV. Use 1,000 permutations (LogReg/SVM) or 200 (RF). p = (1 + #null ≥ observed) / (1 + n_perm).
7. **DE = pseudobulk** (sum raw counts per donor × coarse cell type, min 10 cells) → `pydeseq2`, primary design `~ severe`; `~ sex + severe` as a check. BH-FDR within each cell type × contrast. Significant = padj < 0.05.
8. **Enrichment:** over-representation via Fisher's exact test (BH) on significant genes against an expressed-gene background, plus preranked GSEA (`gseapy.prerank`) on the DESeq2 Wald statistic. **Gene-set files (GMT) are downloaded once and committed** to `data/external/gene_sets/` so results don't depend on internet access or library updates.
9. **Seeds:** `common.SEED = 42` for anything random (UMAP, Leiden, RF, permutations), except CV repeats, which use seeds 0–9.
10. **Every plot states n (patients)** per group.

### Decisions log

| Date | Decision | Who | Why |
|---|---|---|---|
| 2026-10-30 | Primary label = `Cohort_Disease_WHO_Score` (mild 14 vs severe 21); `who_peak` for sensitivity | Sidharth (agreed with team) | Author-defined cohorts; WHO score enables threshold checks |
| 2026-10-30 | `control` = `Control_WHO_0` only; `Control_WHO_8` = specificity check; convalescent excluded | Claude, delegated by Sidharth | Healthy reference is unambiguous; ICU non-COVID controls test whether the signature just reflects "ICU"; n=2 is too small |
| 2026-10-30 | Erythroblasts excluded from composition features | Claude, delegated by Sidharth | 973/986 from 2 bloody-swab donors, so contamination |
| 2026-10-30 | Pseudobulk min 10 cells per donor × cell type | Claude, delegated by Sidharth | Fewer cells make the profile too noisy; tested in sensitivity (5/10/20) |
| 2026-10-30 | Raw h5ad shared through GitHub Release `data-v1`; `data/external/` committed; `data/processed/` never shared | Sidharth + Claude | GitHub's 100 MB file limit; external snapshots are small and dated |
| 2026-10-30 | Primary CV = repeated stratified 5-fold (10×); LOPO secondary | Claude, delegated by Sidharth | LOO AUROC bias at small n |
| 2026-10-30 | Author annotations used, Leiden only as a check | Claude, delegated by Sidharth | Time; authors' labels validated |
| 2026-10-30 | Python 3.12 + pinned `requirements.txt` | Sidharth + Claude | Reproducibility without a shared env |
| 2026-10-30 | Deck owner/presenter **not yet decided** | Team | Decide by Sun 1 Nov 21:00 (TEAM_PLAN.md) |

---

## 6. Data contracts (interfaces between workstreams)

Rule: **a contract file is produced only by its owner's numbered script** in `scripts/`. Consumers only read it. Changing a schema = PR that edits this section, and the consumers review it. All paths are relative to the repo root (constants in `scripts/common.py`). Tables are tab-separated `.tsv` with a header row. Cell-type names in column names and filenames are **slugs**: lowercase, spaces → `_` (e.g. `Developing Ciliated Cells` → `developing_ciliated_cells`).

### Pipeline order

| Step | Script | Owner | Reads | Writes (contract) |
|---|---|---|---|---|
| 00 | `scripts/00_tidy_external.py` | Eswar | `data/external/*` | C10 |
| 01 | `scripts/01_qc.py` | Vijay | raw h5ad | **C1** |
| 02 | `scripts/02_embed_cluster.py` | Vijay | C1 | `results/qc/` figures and tables, `data/processed/nasal_embedded.h5ad` (not a contract) |
| 03 | `scripts/03_patient_features.py` | Sahaj | C1 | **C2, C3, C4** |
| 04 | `scripts/04_composition_tests.py` | Sahaj | C2, C3 | **C6** |
| 05 | `scripts/05_classify.py` (+ `scripts/classify_utils.py`) | Ruchir | C2, C3, C4 | **C5** |
| 06 | `scripts/06_isg_baseline.py` | Sidharth | C1, C2 | `results/isg/` |
| 07 | `scripts/07_pseudobulk_de.py` | Sahaj | C2, C4 | **C7** |
| 08 | `scripts/08_enrichment.py` | Sahaj | C7 | `results/de/enrichment_*.tsv` |
| 09 | `scripts/09_attribution.py` | Ruchir | C2, C3, C4 | **C8** |
| 10 | `scripts/10_key_genes.py` | Sahaj | C7, C8 | **C9** |
| 11 | `scripts/11_sensitivity.py` | Vijay | C1–C4, `classify_utils.py` | **C12** |
| 12 | `scripts/12_perturbation.py` | Sidharth | C2, C4, C9, `classify_utils.py` | **C11** |
| 13 | `scripts/13_targets.py` | Sidharth | C9, C10, C11 | **C13** |
| 14 | `scripts/14_validation_ziegler.py` | Vijay | C1, C6, C7, C9 | `results/validation/` |
| 15 | `scripts/15_external_cohort.py` (stretch) | Vijay | Chua et al. data, C5 model | `results/validation/` |

To rebuild everything from a clean clone: run `python scripts/NN_*.py` in numeric order. Steps 00 and 06 are independent of the rest.

### C1 `data/processed/nasal_qc.h5ad` (Vijay, step 01, **due Sat 31 Oct 13:00**)
- AnnData of the cells kept after QC (by default all 32,588; see §5.3). All 58 donors kept. Downstream code filters by `group`.
- `layers['counts']`: raw integer counts copied from `raw.X`. `.X`: log1p(CP10k) from `counts`. `.raw`: not set.
- `var`: index = Ensembl ID. Columns: `gene_symbol` (from `feature_name`), `is_viral` (bool), `highly_variable` (computed on host genes, used for embedding only).
- `obs` **required columns** (all original columns are also kept):
  `donor_id` (str), `group` (§4), `severe` (float 1/0/NaN), `who_peak` (int), `sex` (str), `age_decade` (str, from `development_stage`), `bloody_swab` (bool), `ct_coarse` (= `Coarse_Cell_Annotations`), `ct_detailed` (= `Detailed_Cell_Annotations`), `n_counts`, `n_genes`, `pct_mito` (recomputed), `viral_umis` (= `SARSCoV2_Total_Corrected`).
- `uns['qc_params']`: dict of the thresholds used.

### C2 `data/processed/patient_meta.tsv` (Sahaj, step 03, **first version Sat 13:00**)
One row per donor (58). Columns: `donor_id, group, severe, who_peak, sex, age_decade, bloody_swab, pcr_status, n_cells` (n_cells after QC).

### C3 `data/processed/features_composition.tsv` (Sahaj, step 03, **first version Sat 13:00**)
One row per donor (58). Columns: `donor_id, n_cells_composition`, then `prop__<ct>` and `clr__<ct>` for the 17 coarse types (Erythroblasts excluded).

### C4 `data/processed/pseudobulk_coarse.h5ad` (Sahaj, step 03, **due Sat 31 Oct 20:00**)
- `X` = **summed raw counts** (integers) for each donor × cell type with `n_cells >= 10`. Rows below the threshold are absent, not zero. Also includes `ct = "all"`, the whole-sample pseudobulk per donor (all non-erythroblast cells).
- `obs` (index `<donor_id>__<ct>`): `donor_id, ct, n_cells, group, severe, who_peak, sex`.
- `var`: host genes only (viral removed). Index = Ensembl ID, `gene_symbol`.
- `uns['min_cells'] = 10`.

### C5 `results/classify/` (Ruchir, step 05, **first results Sat 31 Oct 23:00, final for deck Sun 1 Nov 18:00**)
- `oof_predictions.tsv`: `donor_id, task, feature_set, model, cv_scheme, repeat, y_true, score` (score = out-of-fold P(severe) or decision value).
- `cv_metrics.tsv`: `task, feature_set, model, cv_scheme, n_pos, n_neg, auroc_median, auroc_min, auroc_max, bal_acc_median, perm_p, n_perm`.
- `perm_null.tsv`: `task, feature_set, model, perm_id, auroc`.
- `scripts/classify_utils.py` exposes plain functions `make_model(name, seed)` and `run_cv(X, y, model_name, cv_scheme, seed)` → OOF scores. Steps 11 and 12 import these, so their signatures are part of the contract.

### C6 `results/composition/composition_tests.tsv` (Sahaj, step 04, **due Sun 1 Nov 13:00**)
`ct, comparison` (`kw_control_mild_severe` / `mild_vs_severe` / `control_vs_covid`), `test, statistic, pvalue, qvalue_bh, median_control, median_mild, median_severe, n_control, n_mild, n_severe, direction`.

### C7 `results/de/de_<ct>_<contrast>.tsv` + `results/de/de_all.tsv` (Sahaj, step 07, due Tue 10 Nov)
`gene_id, gene_symbol, ct, contrast` (`severe_vs_mild`, `covid_vs_control`, `severe_vs_control_icu`), `base_mean, log2fc, lfcse, stat, pvalue, padj, n_donors_a, n_donors_b`. log2fc > 0 = higher in the first group named.

### C8 `results/attribution/` (Ruchir, step 09, due Thu 12 Nov)
- `celltype_importance.tsv`: `ct, n_donors, n_pos, n_neg, auroc_single_median` (model on that cell type only), `auroc_drop_median` (all features minus that type), `delta_auroc, perm_p`.
- `gene_importance.tsv`: `gene_id, gene_symbol, ct, model, importance` (mean |coef| or mean |SHAP| over folds), `rank, stability` (fraction of CV fits where the gene is in the top 50).

### C9 `results/de/key_gene_shortlist.tsv` (Sahaj, step 10, due Sat 14 Nov)
≤ 50 rows. `gene_id, gene_symbol, ct, direction` (`up_in_severe` / `down_in_severe`), `log2fc, padj, importance_rank, evidence` (`de` / `model` / `both`), `in_isg_set` (bool), `role_summary` (one line).

### C10 `data/processed/external/target_annotations.tsv` (Eswar, step 00, due Tue 10 Nov)
One row per gene symbol. `gene_symbol, ensembl_id, hpa_protein_class, hpa_rna_tissue_specificity, dgidb_categories` (`;`-separated), `dgidb_n_drugs, dgidb_n_approved_drugs, dgidb_example_drugs` (≤5, `;`), `ot_covid19_score` (Open Targets, NaN if absent), `gtex_max_tissue, gtex_n_tissues_tpm_gt10`.
**Not on the critical path:** if it's late, step 13 reads the raw `data/external/` tables directly.

### C11 `results/perturb/perturbation_effects.tsv` (Sidharth, step 12, due Wed 18 Nov)
`gene_id, gene_symbol, ct, shift` (`severe_to_mild` / `mild_to_severe` / `severe_to_control`), `alpha, model, n_patients, mean_delta_p, median_delta_p, frac_flipped, null_p, null_n`.

### C12 `results/sensitivity/sensitivity_summary.tsv` (Vijay, step 11, due Wed 18 Nov)
`axis, setting, n_pos, n_neg, feature_set, model, auroc_median, auroc_min, auroc_max, bal_acc_median, perm_p, top3_celltypes, jaccard_top20_genes_vs_primary`.

### C13 `results/targets/target_ranking.tsv` (Sidharth, step 13, due Thu 19 Nov)
`gene_symbol, ct, direction, desired_drug_action` (`inhibit` / `activate`), `perturb_mean_delta_p, druggable` (bool), `n_approved_drugs, example_drugs, ot_covid19_score, tissue_specificity_flag, score, rank, rationale`.

---

## 7. Analysis definitions (post-presentation workstreams)

### 7.1 Cell-type attribution (Ruchir, step 09)
- **Single-cell-type models:** for each coarse type with at least 10 cells in at least 8 mild **and** 8 severe donors (on the raw data, verified 2026-10-30: Ciliated, Secretory, Goblet, Squamous, T cells), train the §5.6 models on `pb_<ct>` features. Use only donors that have that type, and report n. AUROC is compared with that type's own permutation null.
- **Ablation:** combined model (composition + pseudobulk of all eligible types). Drop one type's features at a time and record ΔAUROC = full − dropped. Donors missing a type get that block imputed with the training-fold median **inside the fold**.
- **Gene importance:** mean |coefficient| (L2 LogReg on standardised genes) and mean |SHAP| (RF), averaged over CV fits, with stability = how often the gene is in the top 50.
- *Meaningful result:* a cell type whose single-type AUROC beats its permutation null (p < 0.05) **and** whose removal lowers combined AUROC more than the spread across CV repeats.

### 7.2 Perturbation analysis (Sidharth, step 12)
*Question:* if a key gene's expression in a key cell type were moved toward its level in mild (or healthy) patients, how much would the model's predicted probability of severe disease fall? This is the mentor's "vary expression levels of key genes and see the effect".
- **Model:** L2 logistic regression on standardised pseudobulk logCPM genes of the key cell type(s) (smooth, so small shifts give graded responses). The RF is a secondary check; it gives step-wise responses.
- **Out-of-fold only:** each patient is perturbed using the model trained **without** that patient (5-fold, 10 repeats; average over repeats). We never perturb training data.
- **What is perturbed:** for gene *g* in cell type *c*, for patient *i*: x′ = x + α · (m_ref − x), where m_ref is the median logCPM of *g* in the **reference group of the training fold**. α ∈ {0, 0.25, 0.5, 0.75, 1.0}. A "dose–response" sweep α ∈ [−1, 1.5] in steps of 0.25 is used for plots. Only gene *g* changes; all other features stay fixed. The perturbed vector goes through the same fold's scaler (and PCA, if used).
- **Shifts:** `severe_to_mild` (severe patients, m_ref = mild median), `severe_to_control` (m_ref = healthy-control median), and the reverse control `mild_to_severe`.
- **Module version:** the same shift applied together to the gene's co-expression module (genes with Spearman ρ > 0.7 to *g* across training donors). This reduces off-manifold samples where one gene moves and its correlated partners don't.
- **What is measured:** ΔP_i = P(severe | x′) − P(severe | x). Summaries: mean/median ΔP over the group, `frac_flipped` (fraction of severe patients crossing below 0.5 at α = 1), and the slope of P vs α near α = 0 (local sensitivity).
- **Null:** repeat the procedure for 200 random genes matched to *g* on mean expression and variance (10 bins each). `null_p` = (1 + #random with |mean ΔP| ≥ |observed|) / (1 + 200).
- **Meaningful result:** a gene or module where (i) severe → mild shifts lower P(severe) beyond the matched-random null (null_p < 0.05), (ii) the direction agrees with its DE sign, (iii) the effect is consistent across CV repeats and survives the RF check. These genes go to target prioritisation with a **desired drug action** (gene up in severe → inhibit; down in severe, e.g. ISGs → activate/agonise).
- **Limits (say so in the report):** this measures the **model's** dependence on a gene, not biological causality. Shifting one gene ignores downstream regulation. Results are hypotheses for targets, not evidence of efficacy.
- **Sprint:** nothing is run. The deck presents this definition as a planned step.

### 7.3 Sensitivity analysis (Vijay, step 11)
*Question:* do the main conclusions (classifier AUROC > null, the top cell types, the top genes) survive reasonable changes to how we define severity and how we analyse? Every setting reruns the §5.6 pipeline with `classify_utils.run_cv` and reports AUROC median [min–max] over 10 repeats, the permutation p (200 perms), the top 3 cell types and the Jaccard overlap of the top 20 genes with the primary setting.

| Axis | Settings | What it tells us |
|---|---|---|
| A. Severity definition | primary (WHO 1–5 vs 6–8, 14 vs 21); WHO ≤5 vs ≥7 (drop the WHO-6 donor, 14 vs 20); extremes WHO ≤4 vs 8 (9 vs 16); ordinal: Spearman ρ between OOF score and `who_peak` across all 35 | Is the signal a sharp threshold artefact, or graded with severity? |
| B. Specificity | apply the trained mild-vs-severe model to the 6 `control_icu` donors and the 15 healthy controls | If non-COVID ICU patients score "severe", the signature reflects critical illness/ventilation rather than COVID-specific biology |
| C. Feature set | `comp` vs `pb_all` vs per-cell-type vs `comb`; number of HVGs 500/1000/2000; coarse vs detailed annotations for composition | Is the conclusion tied to one representation? |
| D. QC and inclusion | minimum cells per donor 0/50/100; pseudobulk min cells 5/10/20; mito cut-off 50 (author) / 30 / 20 %; exclude the 4 bloody-swab donors | Are results driven by low-quality donors or contamination? |
| E. Sampling depth | downsample every donor to 50, 100, 200 cells (10 draws each) before building features | How many cells per patient are needed? This answers the mentor's cost question directly ($2,000 per sample). |
| F. Influence | leave-one-donor-out influence on AUROC; 10 CV seeds | Does one patient carry the result? |
| G. Covariates | add sex and age decade as features; check whether OOF score correlates with them | Confounding check |

- **Meaningful result:** the AUROC stays above its permutation 95th percentile in most settings, and the top cell types/genes overlap (Jaccard ≥ 0.5). Equally valuable: identifying **where** it breaks (e.g. the signal disappears when controlled for ICU status). Cell-type ablation is in §7.1 and is cross-referenced here.

### 7.4 Drug-target prioritisation (Sidharth, step 13)
Candidates = C9 shortlist genes with a meaningful perturbation effect (C11). Each is scored on: druggability (HPA protein class: enzyme, receptor, transporter, ion channel, secreted; DGIdb "druggable genome" categories), existing compounds (DGIdb approved drugs), prior COVID-19 evidence (Open Targets score), safety (broad expression in normal tissues per HPA/GTEx flags a risk), and perturbation effect size. Use a simple transparent additive score (documented weights) plus a written rationale per gene. Output: top ~10.

### 7.5 Validation (Vijay, step 14–15)
- Ziegler claims (Eswar's table `docs/literature/ziegler_findings.md` lists them) checked one by one against our data: e.g. ISG expression in epithelial cells vs viral load, severe vs mild; secretory/goblet changes; inflammatory macrophages in severe.
- Stretch: Chua et al. 2020 nasopharyngeal scRNA-seq (moderate vs critical). Harmonise genes, rebuild patient features with the same code, apply the trained model unchanged and report AUROC.

---

## 8. Current status

| Workstream | Owner | Status | Last updated |
|---|---|---|---|
| Scaffolding (docs, env, skeleton) | Sidharth + Claude | done, pending PR merge | 2026-10-30 |
| qc (01–02) | Vijay | 01 merged (C1 + 2 figures); 02 done (PCA/UMAP/Leiden + ARI, 5 figures, branch `klnsv/pca-umap`, PR pending). Strong per-donor structure in UMAP | 2026-11-01 |
| composition / features (03–04) | Sahaj | not started | 2026-10-30 |
| classify (05) | Ruchir | not started | 2026-10-30 |
| isg baseline (06) + deck integration | Sidharth | not started | 2026-10-30 |
| lit / external tables (00) | Eswar | not started | 2026-10-30 |
| de, enrichment, key genes (07, 08, 10) | Sahaj | post-presentation | 2026-10-30 |
| attribution (09) | Ruchir | post-presentation | 2026-10-30 |
| sensitivity (11) | Vijay | post-presentation | 2026-10-30 |
| perturb (12), targets (13) | Sidharth | post-presentation | 2026-10-30 |
| validation (14–15) | Vijay | post-presentation | 2026-10-30 |

Update your own row in the same PR as your work (one-line edit; conflicts stay trivial).

---

## 9. Session logs

- Each agent session writes **one file**: `logs/YYYY-MM-DD_HHMM_<name>_<workstream>.md` using [logs/_TEMPLATE.md](logs/_TEMPLATE.md). `<name>` ∈ {sidharth, vijay, sahaj, ruchir, eswar}; `<workstream>` is the code from TEAM_PLAN.md.
- At the end of the session the human runs Claude Code's `/export` and saves the full transcript as `logs/transcripts/<same stem>.txt`.
- Both files go in the PR together with the work. One file per session means no merge conflicts. This section keeps only the condensed status (§8).
- Together these files are the **AI-usage supplement** the course requires (prompts + chat histories).
