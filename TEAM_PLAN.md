# TEAM PLAN: who does what, by when

For the five of us. Skim §2–§4 first.
Project state, dataset facts and the exact file formats between workstreams are in [CONTEXT.md](CONTEXT.md). Rules are in [CONVENTIONS.md](CONVENTIONS.md). `HUMAN_README.md` is ours alone; agents don't touch it.

---

## 1. The project in five lines

- **Data:** nasal-swab single-cell RNA-seq from Ziegler et al. 2021. 58 donors, of which 35 are COVID-19 patients: **14 mild/moderate vs 21 severe**. 32,588 cells.
- **Goal 1 (before the presentation):** can we classify a patient as mild or severe from their nasal cells, honestly cross-validated **at patient level** and beating a permutation null?
- **Goal 2 (after):** which cell types carry the signal → which genes → what they do → which are **drug targets**. Plus a **perturbation analysis** (shift key genes in silico and watch predicted severity move) and a **sensitivity analysis** (does it hold when we change definitions?).
- **Big constraint:** with only 35 patients, the **patient is the sample**, never the cell.
- **Deliverables:** 6-min deck (submit **Mon 2 Nov, 1:00 PM IST**), final report ≤ 8 pages + code + AI logs (**Wed 25 Nov**).

## 2. Timeline

| When | Milestone | Owner |
|---|---|---|
| **Fri 30 Oct, night** | Teammates added as collaborators; scaffolding PR merged; raw h5ad uploaded as GitHub Release `data-v1` | Sidharth |
| **Sat 31 Oct 10:00** | Everyone: clone, venv, data downloaded, md5 checked, branch created | all |
| **Sat 31 Oct 13:00** | **C1 QC'd object script merged** · first patient features (C2/C3) | Vijay · Sahaj |
| **Sat 31 Oct 20:00** | Pseudobulk (C4) · UMAP/cluster figures | Sahaj · Vijay |
| **Sat 31 Oct 23:00** | First classifier results on composition features | Ruchir |
| **Sun 1 Nov 13:00** | Composition tests (C6) + figure · ISG baseline | Sahaj · Sidharth |
| **Sun 1 Nov 18:00** | Final sprint classifier results (C5) incl. pseudobulk features | Ruchir |
| **Sun 1 Nov 20:00** | **Clean-clone rebuild** of the full sprint pipeline on a second machine | Vijay |
| **Sun 1 Nov 21:00** | **Results freeze**: all sprint PRs merged. Deck owner/presenter decided. | all |
| **Sun 1 Nov 23:00** | Deck draft with every figure in place | Sidharth |
| **Mon 2 Nov 09:00–12:30** | Polish, 2 timed rehearsals (≤ 6:00), **submit by 12:30** (30 min buffer) | deck owner (TBD, §11) |
| 3–6 Nov | Presentations | presenter(s) |
| Tue 10 Nov | Pseudobulk DE · classifier v2 · external target tables | Sahaj · Ruchir · Eswar |
| Thu 12 – Sat 14 Nov | Cell-type attribution · enrichment · **key-gene shortlist** | Ruchir · Sahaj |
| Mon 16 Nov | Validation vs Ziegler | Vijay |
| Wed 18 Nov | **Perturbation** and **sensitivity** analyses final | Sidharth · Vijay |
| Thu 19 Nov | Drug-target ranking | Sidharth |
| **Fri 20 Nov** | **Results freeze** (no new analyses after this) | all |
| Sat 21 Nov | Each owner drafts their report sections | all |
| Mon 23 Nov | Report assembled (≤ 8 pages) | Sidharth |
| Tue 24 Nov | Formatting (Arial 11, single spacing), references, supplement (code + AI logs) | Eswar |
| **Wed 25 Nov** | **Final report submitted** | Sidharth (backup Ruchir) |

## 3. Work assignment

Each of the four engineers owns one sprint workstream and one or two post-presentation workstreams, all computational. Eswar's work is self-contained, and nothing waits on it.

| Code | Workstream | Owner | Deliverables (files) | Depends on | Due | Phase |
|---|---|---|---|---|---|---|
| `qc` | QC, normalisation, PCA/UMAP, Leiden vs author labels | **Vijay** (`klnsv`) | `scripts/01_qc.py` → **C1** + donor overview and QC violin figures; `scripts/02_embed_cluster.py` → `results/qc/figures/` (PCA variance, UMAPs by cell type / group / donor, ARI table) | raw h5ad | C1 Sat 31 Oct 13:00; figures Sat 20:00 | **sprint** |
| `composition` | Patient features + composition tests | **Sahaj** (`SSahaj24`) | `scripts/03_patient_features.py` → **C2, C3, C4**; `scripts/04_composition_tests.py` → **C6** + `results/composition/figures/` | C1 (develop on raw first) | C2/C3 Sat 13:00; C4 Sat 20:00; C6 Sun 1 Nov 13:00 | **sprint** |
| `classify` | Patient-level classifiers, CV, permutation test | **Ruchir** (`Ruchir-r`) | `scripts/classify_utils.py`, `scripts/05_classify.py` → **C5** (`cv_metrics.tsv`, `oof_predictions.tsv`, `perm_null.tsv`) + ROC / permutation figures | C3 (Sat 13:00), C4 (Sat 20:00) | first Sat 23:00; final Sun 1 Nov 18:00 | **sprint** |
| `isg` + `deck` | Interferon-signature baseline + sprint integration | **Sidharth** (`sid-biowiz`) | `scripts/06_isg_baseline.py` → `results/isg/`; ISG gene-set file in `data/external/gene_sets/`; workflow diagram; deck draft | C1, C2 | ISG Sun 1 Nov 13:00; deck draft Sun 23:00 | **sprint** |
| `lit` | Ziegler findings table + references | **Eswar** (`eswardattasai`) | `docs/literature/ziegler_findings.md`, `docs/literature/references.md` | none | Sun 1 Nov 18:00 (optional, not blocking) | sprint (light) |
| `classify` v2 + `attribution` | Per-cell-type features, combined model; which cell types carry the signal; gene importance (coef/SHAP) | **Ruchir** | `scripts/09_attribution.py` → **C8** | C4, C5 | v2 Tue 10 Nov; C8 Thu 12 Nov | post |
| `de` | Pseudobulk DE, enrichment, key-gene shortlist + gene roles | **Sahaj** | `07_pseudobulk_de.py` → **C7**; `08_enrichment.py`; `10_key_genes.py` → **C9** | C4; C8 for C9 | C7 Tue 10 Nov; enrichment Fri 13 Nov; C9 Sat 14 Nov | post |
| `perturb` | In-silico perturbation (CONTEXT.md §7.2) | **Sidharth** | `12_perturbation.py` → **C11** | C4, `classify_utils`, C9 | prototype on ISG genes Fri 13 Nov; final Wed 18 Nov | post |
| `sensitivity` | Sensitivity analysis, axes A–G (CONTEXT.md §7.3) | **Vijay** | `11_sensitivity.py` → **C12** | C1–C4, `classify_utils` | Wed 18 Nov | post |
| `validation` | Ziegler claims check; Chua et al. cohort (stretch) | **Vijay** | `14_validation_ziegler.py`, `15_external_cohort.py` → `results/validation/` | C6, C7, C9, Eswar's table | Ziegler Mon 16 Nov; Chua Fri 20 Nov | post |
| `targets` | Drug-target prioritisation (CONTEXT.md §7.4) | **Sidharth** | `13_targets.py` → **C13** | C9, C10, C11 | Thu 19 Nov | post |
| `lit` (post) | External-table sources + tidy table; literature evidence for shortlist genes; report formatting + AI-log supplement | **Eswar** | `data/external/SOURCES.md`, `00_tidy_external.py` → **C10**; `docs/literature/key_gene_evidence.md`; formatted report | C9 for the gene table only | C10 Tue 10 Nov; evidence Fri 20 Nov; formatting Tue 24 Nov | post |
| `report` | Report integration | **Sidharth** (lead), each owner writes their sections | report draft | everything | sections Sat 21 Nov; assembled Mon 23 Nov | post |

**Reviewers** (each PR is reviewed by whoever consumes it): Vijay's QC → Sahaj · Sahaj's features → Ruchir · Ruchir's classifier → Vijay · Sidharth's ISG/deck → Ruchir · Eswar's → Sidharth.

## 4. The sprint, in half-day blocks (Sat 31 Oct – Sun 1 Nov)

Handoffs are in **bold**. A handoff counts as done when the script is **merged into `main`**. Consumers then run it locally (`python scripts/NN_*.py`); nobody sends files around.

| Block | Vijay (`qc`) | Sahaj (`composition`) | Ruchir (`classify`) | Sidharth (`isg`/`deck`) | Eswar (`lit`) |
|---|---|---|---|---|---|
| **Sat AM** 09:00–13:00 | `01_qc.py`: counts layer, QC metrics, standard obs columns, log-norm, HVGs → **C1 merged by 13:00** | `03_patient_features.py` built **on the raw h5ad** (same obs logic) → **C2 + C3 first version by 13:00** | `classify_utils.py` + `05_classify.py` skeleton: models, repeated stratified 5-fold, LOPO, permutation; test on a throwaway proportions table in `notebooks/scratch/` | Release `data-v1` up (Fri night); ISG gene set chosen + committed; `06_isg_baseline.py` on raw | (optional) start Ziegler findings table |
| **Sat PM** 14:00–20:00 | `02_embed_cluster.py`: PCA, UMAP, Leiden, ARI, marker dotplot → **figures by 20:00** | switch 03 input to C1; add pseudobulk → **C4 by 20:00**; start `04_composition_tests.py` | run `comp` features: 3 models × 2 CV schemes + permutations; COVID-vs-control sanity → **first C5 by 23:00** | review Vijay's PR; workflow diagram; deck skeleton (≤ 10 slides) | |
| **Sun AM** 09:00–13:00 | review Ruchir's PR (rerun it); polish slide 2–3 figures | **C6 + composition figure by 13:00** | add `pb_all` features (needs C4) and `comb`; ROC + permutation-null figures | ISG results by 13:00; future-plans slide from CONTEXT.md §7 | |
| **Sun PM** 14:00–21:00 | **18:00–20:00 clean-clone rebuild** of steps 01–06 on your machine; report any mismatch | review Sidharth's PR; slide 6 | **final C5 by 18:00**; slides 7–8 | collect figures; **results freeze 21:00**; deck draft by 23:00 | Ziegler table by 18:00 (optional) |

**If someone is late:** Ruchir can build the classifier on his own proportions from the raw h5ad until C3 lands. Sahaj can run everything on the raw h5ad until C1 lands, because default QC keeps all cells and the swap is one path. Nothing waits on Eswar.

**Honest scope:** cell-type attribution, DE, enrichment, perturbation, sensitivity, targets and validation are **not** attempted before the presentation. The deck presents them as planned steps with precise definitions.

## 5. Monday 2 Nov (before 1:00 PM)

| Time | What | Who |
|---|---|---|
| 09:00–10:30 | Fix figures/wording from the Sunday-night draft; check every number against `results/` | deck owner + slide owners |
| 10:30–11:30 | Two timed run-throughs (target 5:30, hard cap 6:00) | presenter(s) |
| 11:30–12:15 | Final edits; export PDF | deck owner |
| **12:30** | **Submit** (30 min buffer before 1:00 PM) | deck owner, backup on standby |

Deck owner + backup + presenter(s): **to be decided by Sun 21:00** (proposal: owner Sidharth, backup Ruchir).

## 6. Per-person guide

### Vijay (`klnsv`): `qc` → `sensitivity`, `validation`
**You own:** the QC'd object everyone builds on (C1), the exploratory figures for slides 2–3, the reproducibility check of the sprint, and after the presentation the sensitivity analysis and validation.
**First steps (Sat AM):**
1. `git switch -c vijay/qc`. Write `scripts/01_qc.py` to the C1 spec (CONTEXT.md §6): `layers['counts']` from `raw.X`, recomputed `n_counts/n_genes/pct_mito`, required obs columns via `common.GROUP_MAP`, normalise + log1p, HVGs on host genes. Default: keep all author-QC'd cells (CONTEXT.md §5.3).
2. Two figures from step 01: `01_qc_violins.png` (genes/UMIs/mito per group) and `01_donor_overview.png` (cells per donor coloured by group, n on plot). **Open the PR by 12:00** so Sahaj can review and it's merged by 13:00.
3. `scripts/02_embed_cluster.py`: PCA variance, UMAP coloured by `ct_coarse` / `group` / `donor_id`, Leiden at 2–3 resolutions with ARI vs author labels.

**Hand off:** C1 (h5ad, via script) → Sahaj, Sidharth, Ruchir. Figures (`results/qc/figures/*.png`) → deck slides 2–3.

### Sahaj (`SSahaj24`): `composition` → `de`
**You own:** the patient-level tables everyone models on (C2, C3, C4), the composition tests, and after the presentation pseudobulk DE, enrichment and the key-gene shortlist.
**First steps (Sat AM):**
1. `git switch -c sahaj/patient-features`. Write `scripts/03_patient_features.py` reading the **raw** h5ad for now (apply `common.GROUP_MAP`). Output C2 (`patient_meta.tsv`) and C3 (proportions + CLR, Erythroblasts excluded). **PR by 12:30.**
2. After C1 merges, switch the input to `common.QC_H5AD` and add C4 (pseudobulk counts per donor × coarse type, ≥ 10 cells, plus `all`).
3. `scripts/04_composition_tests.py`: Kruskal–Wallis and Mann–Whitney U, BH across cell types → C6 + box/strip plots with q-values and n.

**Hand off:** C2/C3/C4 → Ruchir (classifier), Sidharth (ISG), Vijay (sensitivity). C6 + figure → slide 6.

### Ruchir (`Ruchir-r`): `classify` → `attribution`
**You own:** the "initial results" of the presentation and the CV machinery everyone reuses (`classify_utils.py`). After the presentation: classifier v2 and cell-type attribution.
**First steps (Sat AM):**
1. `git switch -c ruchir/classifier-v1`. Write `scripts/classify_utils.py` with plain functions `make_model(name, seed)` and `run_cv(X, y, model_name, cv_scheme, seed)`. CV schemes: repeated stratified 5-fold (×10, seeds 0–9) and LOPO. All preprocessing inside folds.
2. `scripts/05_classify.py`: mild vs severe (and control vs COVID as a sanity check) on `comp`, then `pb_all` (HVG + PCA inside the fold), then `comb`; LogReg / linear SVM / RF; AUROC, balanced accuracy; permutation test (1,000 / 200). Spec: CONTEXT.md §5.6.
3. Until C3 lands at 13:00, test on a throwaway proportions table made in `notebooks/scratch/` (not committed).

**Hand off:** C5 + `results/classify/figures/` → slides 7–8. `classify_utils.py` → Vijay (sensitivity) and Sidharth (perturbation). Its function signatures are a contract.

### Sidharth (`sid-biowiz`): `isg` + `deck` → `perturb`, `targets`, `report`
**You own:** tonight's setup, a knowledge-driven baseline (does Ziegler's interferon signature alone separate mild from severe?), the sprint deck draft, and after the presentation the perturbation analysis, drug-target ranking and report assembly.
**First steps:**
1. **Fri night:** make sure all four teammates are repo collaborators (the repo is private; Settings → Collaborators). Review and merge the scaffolding PR. Create Release `data-v1` with the h5ad (§9.2). Message the team when it's up.
2. Sat AM: `git switch -c sidharth/isg-baseline`. Choose an ISG gene set (e.g. MSigDB Hallmark interferon-α response). Commit it to `data/external/gene_sets/` with its source in `data/external/SOURCES.md`. Write `scripts/06_isg_baseline.py`: per-cell ISG score → per-donor mean in epithelial types → mild vs severe (MWU, n). The score's AUROC needs no training, so it's an honest single-feature baseline for slide 8.
3. Sat PM: workflow diagram + deck skeleton; review Vijay's PR.

**Hand off:** ISG figure → slide 8. Deck draft → team, Sun 23:00.

### Eswar (`eswardattasai`): `lit` (light, independent)
**You own:** the literature backbone and the external annotation tables. Nobody waits on you. Sidharth can fall back to the raw tables if needed.
**First steps:**
1. (optional, Sun 1 Nov 18:00) `git switch -c eswar/ziegler-findings`. Write `docs/literature/ziegler_findings.md`: a table of Ziegler et al.'s main claims (claim · figure/panel · cell type · genes · how we could check it in our data). Add `docs/literature/references.md` with the core references.
2. (Tue 10 Nov) Complete `data/external/SOURCES.md` (URLs + versions for DGIdb and HPA); download Open Targets COVID-19 associations and GTEx median TPM (each < 50 MB). Write `scripts/00_tidy_external.py` → C10 (one tidy table, schema in CONTEXT.md §6).
3. (Fri 20 Nov) Literature evidence per shortlist gene. (Tue 24 Nov) Report formatting + supplement (code link + `logs/` index).

**Hand off:** tables and markdown files in the repo. Nothing by hand.

## 7. Presentation plan (6 min, ≤ 10 slides)

The mentor's required flow: (i) nature of the data → (ii) objectives → (iii) workflow → (iv) initial results (**classification**) → (v) future plans.

| # | Part | Slide | Content / figure | Built by | ~sec |
|---|---|---|---|---|---|
| 1 | — | Title | Title, team, one-line question | Sidharth | 10 |
| 2 | (i) | The data | Seq-Well nasal swabs; 58 donors (15 healthy / 14 mild / 21 severe / 6 ICU non-COVID / 2 convalescent); cells per donor; "patient = sample" → `results/qc/figures/01_donor_overview.png` | Vijay | 40 |
| 3 | (i) | Cell landscape | UMAP by cell type and by group, n per group → `results/qc/figures/02_umap_celltype_group.png` | Vijay | 30 |
| 4 | (ii) | Objectives | RQ1–RQ7 condensed: classify → cell types → genes → targets; mentor's cost argument | Sidharth | 35 |
| 5 | (iii) | Workflow | Pipeline diagram (from README §4, updated) with sprint vs planned steps marked | Sidharth | 40 |
| 6 | (iv) | Composition | Cell types that shift with severity, q-values, n → `results/composition/figures/04_composition_box.png` | Sahaj | 35 |
| 7 | (iv) | Classifiers | AUROC + balanced accuracy table (3 models × feature sets), ROC curves → `results/classify/figures/05_roc.png` | Ruchir | 55 |
| 8 | (iv) | Is it real? | Permutation null vs observed AUROC; ISG single-score baseline → `05_permutation_null.png`, `results/isg/figures/06_isg_by_group.png` | Ruchir + Sidharth | 35 |
| 9 | (v) | Future plans | Key genes (pseudobulk DE + importance) → roles → drug targets; perturbation (§7.2) and sensitivity (§7.3), one line each with what result would matter | Sidharth (inputs from all) | 50 |
| 10 | (v) | Timeline + limitations | n = 35, cohort heterogeneity, ICU confound; dates to 25 Nov | Sidharth | 10 |

Total ≈ 5:40. Every number on a slide must come from a file in `results/`.

## 8. After the presentation (3 – 25 Nov)

See the assignment table (§3) for owners and dates and CONTEXT.md §7 for exact definitions. Critical path: **C4 → DE (10 Nov) + attribution (12 Nov) → key-gene shortlist (14 Nov) → perturbation (18 Nov) → targets (19 Nov) → freeze (20 Nov) → report (25 Nov).** Sensitivity and validation run in parallel and only need the sprint outputs. Sidharth starts the perturbation framework on ISG genes on 3 Nov, so he isn't idle waiting for the shortlist.

## 9. How to work

### 9.1 Setup (once)
```bash
git clone https://github.com/sid-biowiz/covid19-scrna.git
cd covid19-scrna
# Python 3.12
python -m venv .venv
# Windows:      .venv\Scripts\activate
# macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt
```

### 9.2 Get the data
- **Sidharth, once:** GitHub → repo → Releases → "Draft a new release" → tag `data-v1` → attach `data/raw/zeigler2021_nasal.h5ad` → Publish.
- **Everyone:** Releases → `data-v1` → download `zeigler2021_nasal.h5ad` into `data/raw/`. Fallback: the CELLxGENE link in CONTEXT.md §3. Check the md5 (command in CONTEXT.md §3) equals `2fcf7a8a3a52ec91e824f2041679195f`.
- Then build everything with `python scripts/01_qc.py`, `python scripts/03_patient_features.py`, … in numeric order.

### 9.3 Every session
```bash
git switch main && git pull
git switch -c <name>/<short-task>      # e.g. vijay/qc
# ... work (with Claude) ...
git status && git diff                 # YOU read what changed
git add <files> && git commit -m "..."
git push -u origin <name>/<short-task>
# open a PR on GitHub, fill the template, ask your reviewer (§3)
```

### 9.4 Starting a Claude session
Open a terminal in the repo, activate the venv, make your branch, run `claude`. Claude auto-loads `CLAUDE.md`, which makes it read the rules and contracts. Example first prompt:
> *"I'm Vijay (klnsv), workstream `qc`, on branch `vijay/qc`. Read CLAUDE.md, CONVENTIONS.md and CONTEXT.md. Task: write `scripts/01_qc.py` that produces contract C1 exactly as specified in CONTEXT.md §6, plus the two step-01 figures named in TEAM_PLAN.md §6. Keep the code simple. Ask before installing anything. Run it, show me the output summary, then write the session log."*

At the end, run `/export` and save the transcript to `logs/transcripts/<same name as the log>.txt`.

### 9.5 Where to find the latest state
- **CONTEXT.md §8:** status per workstream. **§6:** file contracts.
- **`logs/`:** what each session did.
- **GitHub PRs:** what's in flight.

## 10. Who to ask

| Topic | Person |
|---|---|
| QC object, UMAP, reproducibility | Vijay |
| Patient tables, pseudobulk, DE | Sahaj |
| CV, models, `classify_utils.py` | Ruchir |
| Repo, contracts, deck, perturbation, targets | Sidharth |
| References, external tables, formatting | Eswar |

## 11. Open questions / pending decisions

1. **Deck owner, backup and presenter(s)** for Mon 2 Nov. Proposal: owner Sidharth, backup Ruchir. Decide by **Sun 1 Nov 21:00**.
2. **Weekend availability** wasn't collected. The plan assumes all four engineers can do roughly the blocks in §4 on both days. If not, tell Sidharth by Sat 10:00 and blocks get swapped.
3. **Role fit:** roles were assigned without stated preferences. Swapping whole workstreams before Sat 10:00 is fine; update §3 and CONTEXT.md §6/§8 in one PR.
4. **Where the deck lives** (proposal: one shared Google Slides deck; Sidharth posts the link here).
5. **ISG gene set** choice (Sidharth, Sat AM; record it in the CONTEXT.md decisions log).
6. Mentor questions still open (from README §8): is grouped CV enough vs an external cohort; mild-vs-severe only or ordinal; epithelial vs immune focus for targets.
7. Source URLs/versions of `data/external/` files not yet recorded (Eswar, see `data/external/SOURCES.md`).
