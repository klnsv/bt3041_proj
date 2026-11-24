# External data sources

Every file in `data/external/` gets one row here. These files are committed snapshots (each < 50 MB), so the pipeline never calls a live API. Owner: Eswar. Add a row in the same PR that adds the file.

| File | Source | Version / date | md5 | Added by | Notes |
|---|---|---|---|---|---|
| `interactions.tsv` | DGIdb downloads page (https://dgidb.org/downloads), *exact URL: TODO Eswar* | DGIdb v5.0.13, data version 2026-10 (from file header) | `3082ca323ce0e9f5e5338dfe8586fa54` | Sidharth, before 2026-10-30 | drug–gene interactions; 2 comment lines (`#`) before the header |
| `categories.tsv` | DGIdb downloads page, *exact URL: TODO Eswar* | DGIdb v5.0.13, data version 2026-10 | `890fa296d4a877508dc2526ef316e8c6` | Sidharth, before 2026-10-30 | gene druggability categories; 2 comment lines before the header |
| `proteinatlas.tsv.zip` | Human Protein Atlas downloads (https://www.proteinatlas.org/about/download), *exact URL + HPA version: TODO Eswar* | zip entry dated 2026-06-03 | `d96f6f3505024f27586ebdb8bb2bd337` | Sidharth, before 2026-10-30 | 119 columns incl. `Protein class`, `RNA tissue specificity` |
| `gene_sets/` | *to be added* (ISG set: Sidharth; enrichment GMTs: Sahaj) | | | | |

Raw scRNA-seq data is **not** here. See CONTEXT.md §3.
