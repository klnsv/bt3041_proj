## What changed
<!-- 1–3 lines. Workstream code + what this PR adds. -->

## How to regenerate
<!-- Exact commands from a clean clone + data/raw + venv. Write "n/a (docs only)" if nothing to run. -->
```bash
python scripts/NN_xxx.py
```

## Results (if any)
<!-- Key numbers, copied from the output files listed here. -->

## Session log(s)
<!-- logs/YYYY-MM-DD_HHMM_<name>_<ws>.md and logs/transcripts/<same stem>.txt -->

## Checklist (CONVENTIONS.md)
- [ ] Branch is `<name>/<task>`, not `main`; only my workstream's files changed (or shared-file change in its own PR)
- [ ] No data or large files committed (`data/raw`, `data/processed`, `*.h5ad`, anything > 10 MB in `results/`)
- [ ] Contract files are produced by a numbered script; schema matches CONTEXT.md §6 (or this PR updates §6 and consumers were asked)
- [ ] Relative paths via `scripts/common.py`; seeds fixed
- [ ] Notebooks run top-to-bottom on a fresh kernel
- [ ] Patient-level stats: patient-level CV/tests, BH-FDR reported, n (patients) on every plot
- [ ] Every number in this PR/log comes from an actual output
- [ ] Session log + transcript added; my row in CONTEXT.md §8 updated
- [ ] Reviewer reran the regenerate command and got the same numbers
