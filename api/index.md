---
title: "API Reference"
permalink: /api/
toc: true
toc_sticky: true
---

All scRBP commands follow the pattern:

```bash
scRBP <command> [options]
```

Run `scRBP --help` or `scRBP <command> --help` for full option listings.

---

## Commands Overview

| Command | Description |
|---------|-------------|
| `getSketch` | Stratified cell downsampling via GeoSketch (optional) |
| `getMetacell` | Aggregate similar cells into mini-metacells for GRN inference (optional) |
| `getGRN` | GRN inference using GRNBoost2 or GENIE3 (`--mode gene/isoform`) |
| `getMerge_GRN` | Consensus network merging across N seeds |
| `getModule` | Regulon candidate extraction (Top-N / Percentile strategies) |
| `getPrune` | Motif enrichment filtering via ctxcore (NES scoring) |
| `getRegulon` | GMT file generation (gene symbol + Entrez ID) |
| `mergeRegulons` | Merge 4 region-specific GMT files (3UTR / 5UTR / CDS / Introns) |
| `ras` | Regulon Activity Score via AUCell (`--mode sc/ct`); emits per-gene expr-stats for `rgs` / `rgs_rare` |
| `rgs` | **Common-variant** Regulon-level Genetic Association Score via MAGMA (`--mode sc/ct`) |
| `rgs_rare` | **Rare-variant** RGS via competitive gene-set regression on TADA / logBF / burden inputs (`--mode sc/ct`) |
| `trs` | Trait-Relevance Score integrating RAS with common- **or** rare-variant RGS (`--mode sc/ct`) |

---

## getSketch

```
scRBP getSketch --input INPUT --output OUTPUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Input `.h5ad` or `.feather` file |
| `--output` | path | required | Output file (`.h5ad` / `.feather` / `.csv` / `.npz`) |
| `--n_cells` | int | 250000 | Target total number of cells to sketch |
| `--n_pca` | int | 100 | Number of PCA components for GeoSketch |
| `--celltype_col` | str | `celltype` | Cell-type column in `adata.obs` (`.h5ad` input only) |
| `--min_cells_per_type` | int | 50 | Minimum cells per cell type (`.h5ad` input only) |
| `--seed` | int | 42 | Random seed |

---

## getMetacell

```
scRBP getMetacell --input INPUT --output OUTPUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | `.h5ad` (with cell-type column) or `.feather` (gene × cell) |
| `--output` | path | required | Output matrix (gene × metacell): `.h5ad` / `.csv` / `.feather` / `.npz` |
| `--metacell_size` | int | 10 | Target cells per metacell (atlas typical 10–15) |
| `--method` | str | `knn` | `knn`, `kmeans`, or `random` cell-grouping strategy |
| `--agg` | str | `sum` | Aggregate pooled cells by `sum` (default) or `mean` of original counts |
| `--within_celltype` | flag | on | Pool only within each cell type (disable via `--global_pooling`) |
| `--global_pooling` | flag | off | Pool across all cells (not recommended when a cell-type annotation is available) |
| `--celltype_col` | str | `celltype` | Column in `adata.obs` holding cell-type labels |
| `--min_metacell_size` | int | 1 | Merge metacells smaller than this into the nearest one |
| `--n_pca` | int | 50 | PCA components for the similarity embedding (`knn`/`kmeans`) |
| `--seed` | int | 42 | Random seed |
| `--save_members` | flag | False | Also write `<out>_metacell_members.csv` mapping metacells to source cells |

---

## getGRN

```
scRBP getGRN --matrix MATRIX --rbp_list RBP_LIST --output OUTPUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--matrix` | path | required | Expression matrix (`.csv` / `.csv.gz` / `.feather` / `.loom`) |
| `--rbp_list` | path | required | RBP list file (gene symbols, one per line) |
| `--output` | path | required | Output GRN `.tsv` file |
| `--method` | str | `grnboost2` | Algorithm: `grnboost2` or `genie3` |
| `--mode` | str | `gene` | Inference mode: `gene` (RBP→gene) or `isoform` (RBP→isoform) |
| `--isoform_annotation` | path | None | Isoform→gene annotation (required for `--mode isoform`) |
| `--n_workers` | int | all CPUs | Number of parallel workers |
| `--batch_size` | int | 10 | Number of outer batches |
| `--threshold` | float | 0.03 | Absolute Spearman correlation threshold |
| `--correlation` | bool | True | Compute Spearman correlation and Mode columns |
| `--seed` | int | 1234 | Random seed |

---

## getMerge_GRN

```
scRBP getMerge_GRN --pattern "GLOB" --output OUTPUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--pattern` | str | required | Glob pattern matching all seed GRN `.tsv` files |
| `--output` | path | required | Output merged GRN `.tsv` |
| `--corr-threshold` | float | 0.0 | Filter edges with `abs(mean Correlation)` ≤ threshold |
| `--n_present` | int | 0 | Min number of seed runs an edge must appear in |
| `--present_rate` | float | 0.0 | Min presence rate (n_present / N_runs) to keep an edge |

---

## getModule

```
scRBP getModule --input INPUT --output_merged OUTPUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Merged GRN `.tsv` from `getMerge_GRN` |
| `--output_merged` | path | required | Output merged modules `.tsv` |
| `--importance_threshold` | float | 0.005 | Minimum importance score to retain an edge |
| `--top_n_list` | str | `"5,10,50"` | Comma-separated Top-N values for selection |
| `--target_top_n` | str | `"50"` | Target Top-N for merged output |
| `--percentile` | str | `"0.75,0.9"` | Comma-separated percentile thresholds |
| `--verbose` | flag | False | Enable verbose logging |

---

## getPrune

```
scRBP getPrune --rbp_targets RBP_TARGETS --motif_rbp_links MOTIF_RBP_LINKS \
               --motif_target_ranks RANKINGS --save_dir SAVE_DIR [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--rbp_targets` | path | required | Module `.tsv` from `getModule` |
| `--motif_rbp_links` | path | required | Motif–RBP annotation `.feather` |
| `--motif_target_ranks` | path | required | Genome rankings `.feather` (e.g. `hg38_500bp_rankings.feather`) |
| `--save_dir` | path | required | Output directory for pruned scores (Parquet) |
| `--rank_threshold` | int | 1500 | Top-N rank cutoff for motif enrichment |
| `--auc_threshold` | float | 0.05 | AUC threshold for enrichment significance |
| `--nes_threshold` | float | 3.0 | Normalized Enrichment Score threshold |
| `--min_genes` | int | 20 | Minimum target genes to retain a regulon |
| `--n_jobs` | int | all CPUs | Number of parallel processes |
| `--chunksize` | int | 4 | Multiprocessing chunk size |
| `--only_rbp` | str | None | Restrict to a specific RBP (debugging) |
| `--only_strategy` | str | None | Restrict to a specific selection strategy |

---

## getRegulon

```
scRBP getRegulon --input INPUT --out-symbol OUT_SYMBOL --out-entrez OUT_ENTREZ [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Pruned ctxcore `.csv` from `getPrune` |
| `--out-symbol` | path | required | Output GMT file (gene symbols) |
| `--out-entrez` | path | required | Output GMT file (Entrez IDs) |
| `--rbp_col` | str | `RBP` | Column name for RBP in input CSV |
| `--genes_col` | str | auto | Column name for target genes (auto-detected) |
| `--min_genes` | int | 1 | Minimum targets to retain a regulon |
| `--taxid` | int | 9606 | NCBI Taxonomy ID (9606=human, 10090=mouse) |
| `--map-hgnc` | path | None | HGNC table for symbol→Entrez mapping |
| `--map-ncbi` | path | None | NCBI gene info file for symbol→Entrez mapping |
| `--drop-unmapped-genes` | flag | False | Drop genes that cannot be mapped to Entrez IDs |
| `--drop-empty-sets` | flag | False | Drop regulons with zero genes after mapping |

---

## mergeRegulons

```
scRBP mergeRegulons --base_dir BASE_DIR --input GMT_FILENAME --output GMT_FILENAME [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--base_dir` | path | required | Base directory containing region subdirectories |
| `--input` | str | required | Input GMT filename to find in each region directory |
| `--output` | str | required | Output merged GMT filename |
| `--region_order` | list | `3UTR 5UTR CDS Introns` | Order in which regions are merged |
| `--region_glob` | str | `Results_final_*_RBP_top1500_*` | Glob for region-specific subdirectories |
| `--tissue_glob` | str | `z_GRNBoost2_*_30times` | Glob for parent tissue dirs (with `--recursive`) |
| `--recursive` | flag | False | Recursively process multiple parent directories |
| `--dedup_lines` | flag | False | Deduplicate identical GMT lines |
| `--overwrite` | flag | False | Overwrite existing output files |
| `--summary_out` | path | None | Optional output `.tsv` for region-level summary |

---

## ras

```
scRBP ras --mode MODE --matrix MATRIX --regulons REGULONS --out OUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--mode` | str | `ct` | Scoring mode: `sc` (per cell) or `ct` (per cell type) |
| `--matrix` | path | required | Expression matrix (`.h5ad` / `.feather` / `.loom` / `.csv`) |
| `--regulons` | path | required | Regulon GMT file from `mergeRegulons` |
| `--out` | path | required | Output RAS file / prefix |
| `--out_format` | str | `loom` | Output format: `csv`, `loom`, or `both` |
| `--no-csv` | flag | False | Force disable writing CSV even if `--out_format` includes csv |
| `--no-loom` | flag | False | Force disable writing LOOM even if `--out_format` includes loom |
| `--csv_layout` | str | `regulons_by_cells` | CSV orientation: `regulons_by_cells`, `cells_by_regulons`, or `both` |
| `--celltypes-csv` | path | None | CSV with `cell_id`, `cell_type` columns (required for `--mode ct`) |
| `--cell-col` | str | auto | Cell ID column in `--celltypes-csv` |
| `--ctype-col` | str | auto | Cell-type column in `--celltypes-csv` |
| `--n_workers` | int | 4 | Workers for AUCell computation |
| `--min_genes` | int | 1 | Drop regulons with fewer than N targets |
| `--to_upper` | flag | False | Uppercase gene symbols when matching regulons |
| `--emit-expr-stats` | flag | on | Emit `<out>_expr_stats.tsv` (per-gene `mean_expr`, `pct_detected`) reused by `rgs` / `rgs_rare` |
| `--no-expr-stats` | flag | off | Disable expr-stats output |
| `--expr-stats-out` | path | auto | Custom path for expr-stats TSV |

---

## rgs

```
scRBP rgs --mode MODE --magma MAGMA --genes-raw GENES_RAW \
          --sets SETS --out OUT [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--mode` | str | required | `sc` (single-cell) or `ct` (cell-type) |
| `--magma` | path | required | Path to MAGMA binary |
| `--genes-raw` | path | required | MAGMA `<prefix>.genes.raw` from prior gene analysis |
| `--sets` | path | required | Regulon GMT file (Entrez or symbol format) |
| `--id-type` | str | `entrez` | Gene ID format in GMT: `entrez` or `symbol` |
| `--out` | str | required | Output file prefix |
| `--n-null` | int | 1000 | Number of matched null regulons |
| `--seed` | int | 2025 | Random seed for null sampling |
| `--q-bins` | int | 10 | Quantile bins for null matching |
| `--threads` | int | auto | CPU threads for MAGMA |
| `--gene-loc` | path | None | MAGMA `NCBI*.gene.loc` for Entrez↔Symbol mapping |
| `--min_genes` | int | 0 | Minimum regulon size |
| `--cleanup-out` | bool | True | Remove intermediate MAGMA output files |
| `--expr-stats` | path | None | Precomputed expression stats TSV |

---

## rgs_rare

Rare-variant analogue of `rgs` — competitive gene-set regression on external gene-level rare summaries (TADA / logBF / burden / STAAR-O / SAIGE-GENE+). Uses a purely genetic covariate (`z_log_union_CDS_length`) in the primary regression; expression stats are used only for matched-null construction in `--mode ct`.

```
scRBP rgs_rare --mode MODE --rare-summary FILE --rare-gene-col COL \
               --score-mode {pvalue,logbf,direct} --sets GMT --out PREFIX [options]
```

### Core

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--mode` | str | required | `sc` (real regulons only) or `ct` (REAL + matched NULLs) |
| `--rare-summary` | path | required | Gene-level rare-variant summary CSV/TSV |
| `--rare-gene-col` | str | required | Gene column in `--rare-summary` |
| `--rare-id-type` | str | `symbol` | Gene ID type in the rare summary: `symbol` / `entrez` |
| `--score-mode` | str | required | `pvalue` / `logbf` / `direct` |
| `--p-col` / `--logbf-col` / `--score-col` | str | — | Column matching the chosen `--score-mode` |
| `--top-winsor` | float | 0.01 | Upper-tail winsorisation fraction for gene-level rare scores |
| `--sets` | path | required | Regulon GMT (Symbol or Entrez) |
| `--id-type` | str | `symbol` | Gene ID type in `--sets` |
| `--out` | str | required | Output file prefix |
| `--gene-loc` | path | — | MAGMA `NCBI*.gene.loc` for Symbol ↔ Entrez mapping |
| `--min_genes` | int | 0 | Minimum overlap size for a regulon to be tested |
| `--max-regulon-frac` | float | 0.5 | Skip regulons covering more than this fraction of the gene universe |

### CDS covariate

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--cds-length` | path | — | Optional CDS length table; else the CDS column is looked up in `--rare-summary` |
| `--cds-gene-col` | str | `symbol` | Gene column in `--cds-length` |
| `--cds-id-type` | str | `symbol` | Gene ID type in `--cds-length` |
| `--cds-col` | str | `union_cds_length` | CDS covariate column name |
| `--cds-scale` | str | `raw` | `raw` — internally `log1p` + z-scored; `z_log` — already z-standardised, used as-is |

### Matched-null construction (`--mode ct`)

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--expr-stats` | path | — | Precomputed expression stats TSV; reuses the file emitted by `ras --emit-expr-stats` |
| `--emit-expr-stats` | bool | False | Compute expr-stats from `--matrix-stats` on the fly |
| `--matrix-stats` | path | — | Expression matrix used only to compute expr-stats |
| `--n-null` / `--null` | int | 1000 | Number of matched null regulons per real regulon |
| `--seed` | int | 2025 | Seed for null sampling |
| `--q-bins` | int | 5 | Quantile bins for matched-null construction |
| `--exclude-self` | bool | True | Exclude real-regulon genes when sampling nulls |
| `--min-bucket-size` | int | 5 | Minimum matched-bucket size before falling back to a coarser stratum |
| `--fdr-method` | str | `BH` | FDR method for the empirical audit table (`BH` / `BY`) |
| `--save-null-gmt` | path | — | Save REAL+NULL GMT in the working ID type |
| `--save-null-gmt-symbol` / `--save-null-gmt-entrez` | path | — | Also save REAL+NULL GMT in Symbol / Entrez |

---

## trs

Integrates RAS with either the **common-variant** RGS from `rgs` or the
**rare-variant** RGS from `rgs_rare`. The same command handles both — just
point `--rgs-csv` at the CSV you want to combine with RAS.

```
scRBP trs --mode MODE --ras RAS --rgs-csv RGS_CSV --out-prefix PREFIX [options]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--mode` | str | required | `sc` (single-cell) or `ct` (cell-type) |
| `--ras` | path | required | RAS matrix from `ras` (`.csv` / `.loom`) |
| `--rgs-csv` | path | required | RGS `.csv` — common-variant `<out>_real.csv` from `rgs` **or** rare-variant `<out>.gsa_RGS.csv` from `rgs_rare` |
| `--out-prefix` | str | required | Output file prefix |
| `--rgs-score` | str | `mlog10p` | RGS score to use: `mlog10p` or `z` |
| `--lambda-penalty` | float | 1.0 | Penalty for RAS–RGS divergence (λ) |
| `--q-hi-ras` | float | 0.99 | Upper quantile cap for RAS normalization |
| `--q-hi-rgs` | float | 0.99 | Upper quantile cap for RGS normalization |
| `--do-fdr` | int | 1 | Apply BH-FDR correction (1=yes, 0=no; CT mode only) |
| `--celltypes-csv` | path | None | CSV with `cell_id`, `cell_type` columns (CT mode) |
| `--min_cells_pert_ct` | int | 25 | Minimum cells per cell type (CT mode) |
