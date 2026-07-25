---
title: "Pipeline"
permalink: /pipeline/
toc: true
toc_sticky: true
---

scRBP implements a **12-step command-line pipeline** that takes raw single-cell RNA-seq data
and produces trait-relevant RBP regulon rankings, using **parallel common- and rare-variant**
genetic association models.

```
Raw scRNA-seq (.h5ad / .feather)
        ↓
  [1]  getSketch       → Stratified GeoSketch downsampling (optional)
  [2]  getMetacell     → Mini-metacell aggregation for the GRN branch (optional)
  [3]  getGRN          → GRN inference (GRNBoost2/GENIE3, --mode gene/isoform)
  [4]  getMerge_GRN    → Consensus merging (N seeds, default 30)
  [5]  getModule       → Regulon candidate extraction
  [6]  getPrune        → Motif enrichment pruning
  [7]  getRegulon      → GMT generation (symbol + Entrez)
  [8]  mergeRegulons   → Merge 4 region GMT files (3UTR / 5UTR / CDS / Introns)
        ↓
  [9]  ras             → Regulon Activity Score (RAS, --mode sc/ct)
  [10] rgs             → Common-variant RGS via MAGMA (--mode sc/ct)
  [11] rgs_rare        → Rare-variant RGS via competitive regression (--mode sc/ct)
  [12] trs             → Trait-Relevance Score (RAS × RGS, common OR rare)
        ↓
   Trait-relevant RBP regulon rankings
```

> Steps 2 (**getMetacell**) and 11 (**rgs_rare**) are new in v0.1.4.1.
> `getMetacell` densifies the input for the GRN branch when dropout is heavy;
> the RAS branch always uses real single cells.
> `rgs_rare` runs in parallel to `rgs` and its output can be fed into `scRBP trs`
> to obtain a **rare-variant** TRS with the same command.

---

## Step 1: getSketch {#step-1-getSketch}

Stratified cell downsampling using **GeoSketch** (Optional). For `.h5ad` input, performs per-cell-type stratified sampling to ensure each cell type has at least `min_cells_per_type` cells while approaching the global `n_cells` target.

```bash
scRBP getSketch \
  --input data.h5ad \
  --output sketch.feather \
  --n_cells 50000 \
  --celltype_col celltype \
  --min_cells_per_type 50
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | Input `.h5ad` or `.feather` file |
| `--output` | path | required | Output file (`.h5ad` / `.feather` / `.csv` / `.npz`) |
| `--n_cells` | int | 50000 | Target total number of cells to sketch |
| `--n_pca` | int | 100 | Number of PCA components for GeoSketch |
| `--celltype_col` | str | `celltype` | Column in `adata.obs` for cell-type labels (`.h5ad` only) |
| `--min_cells_per_type` | int | 50 | Minimum cells per cell type to guarantee (`.h5ad` only) |
| `--seed` | int | 42 | Random seed |

> **Note:** `.feather` input lacks cell-type metadata; global GeoSketch is applied without per-type guarantees.

---

## Step 2: getMetacell {#step-2-getMetacell}

Aggregate transcriptomically similar single cells into **mini-metacells** (default 10 cells per unit) to densify the expression matrix for downstream GRN inference. Single-cell RNA-seq is sparse (high dropout), which is especially damaging for **RBP** regulators — their post-transcriptional co-regulation typically has a smaller dynamic range than transcription-factor networks. Pooling similar cells markedly improves signal-to-noise for `getGRN`.

**Relationship to `getSketch` (complementary, not alternatives):**

- **`getSketch`**   — *selects* a diversity-preserving subset of **real** single cells → feed the **activity (RAS) branch**, which needs real cells to keep cell-type resolution.
- **`getMetacell`** — *aggregates* similar cells into **dense** pseudo-cells → feed the **GRN branch** (`getGRN → getModule → getPrune`).

For large multi-tissue atlases the two are composable:
`getSketch` (pre-thin) → `getMetacell --within_celltype` → `getGRN`.
Metacells are built **within each cell type** by default, so cell-type boundaries are preserved.

> **Normalisation contract.** Cell *similarity* (which cells to pool) is computed on a
> library-size-normalised, log1p-transformed PCA embedding, but the metacell *profile*
> aggregates the **original raw counts** (sum by default). Feed `getMetacell` raw/linear
> counts, exactly as you would feed `getGRN` afterwards.

```bash
scRBP getMetacell \
  --input             lung_lineage.h5ad \
  --output            lung_metacell.feather \
  --metacell_size     10 \
  --method            knn \
  --within_celltype \
  --celltype_col      cell_type \
  --min_metacell_size 5
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | `.h5ad` (with cell-type column) or `.feather` (gene × cell). |
| `--output` | path | required | Output matrix (gene × metacell). `.h5ad` / `.csv` / `.feather` / `.npz`; directly consumable by `getGRN`. |
| `--metacell_size` | int | 10 | Target cells per metacell (atlas typical 10–15). |
| `--method` | str | `knn` | `knn` (greedy nearest-neighbour, most faithful), `kmeans` (MiniBatchKMeans, scalable), or `random` (baseline). |
| `--agg` | str | `sum` | Aggregate pooled cells by `sum` (default) or `mean` of original counts. |
| `--within_celltype` | flag | on | Pool only within each cell type. Disable via `--global_pooling`. |
| `--global_pooling` | flag | off | Pool across all cells (not recommended when a cell-type annotation is available). |
| `--celltype_col` | str | `celltype` | Column in `adata.obs` holding cell-type labels. |
| `--min_metacell_size` | int | 1 | Merge metacells smaller than this into the nearest one (recommended `~metacell_size // 2`). |
| `--n_pca` | int | 50 | PCA components for the similarity embedding (`knn`/`kmeans`). |
| `--seed` | int | 42 | Random seed. |
| `--save_members` | flag | off | Also write `<out>_metacell_members.csv` mapping metacells to source cells. |

**Outputs**

| File | Description |
|------|-------------|
| `<output>` | Metacell expression matrix (gene × metacell). |
| `<output_prefix>_metacell_to_celltype.csv` | `cell`, `cell_type`, `n_cells` — drop-in for `ras --celltypes-csv`. |
| `<output_prefix>_metacell_summary.csv` | Per-cell-type size statistics. |
| `<output_prefix>_metacell_members.csv` | *(optional, `--save_members`)* Full source-cell membership table. |

---

## Step 3: getGRN {#step-3-getGRN}

GRN inference using **GRNBoost2** or **GENIE3**. Supports two modes: **gene-level** (RBP→gene) and **isoform-level** (RBP→isoform). Run this command with multiple seeds (e.g., 30 times) for robust consensus networks.

```bash
# Gene-level inference (recommended)
scRBP getGRN \
  --matrix sketch.feather \
  --rbp_list rbp_list.txt \
  --output grn_seed1.tsv \
  --method grnboost2 \
  --mode gene \
  --seed 1

# Isoform-level inference
scRBP getGRN \
  --matrix sketch.feather \
  --rbp_list rbp_list.txt \
  --output grn_isoform_seed1.tsv \
  --mode isoform \
  --isoform_annotation isoform2gene.txt
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--matrix` | path | required | Expression matrix (`.csv` / `.csv.gz` / `.feather` / `.loom`); rows=features, cols=cells |
| `--rbp_list` | path | required | RBP list file (gene symbols, one per line) |
| `--output` | path | required | Output GRN `.tsv` file |
| `--method` | str | `grnboost2` | Inference algorithm: `grnboost2` or `genie3` |
| `--mode` | str | `gene` | Inference mode: `gene` (RBP→gene) or `isoform` (RBP→isoform) |
| `--isoform_annotation` | path | None | Isoform→gene annotation file (required when `--mode isoform`) |
| `--n_workers` | int | all CPUs | Number of parallel workers |
| `--batch_size` | int | 10 | Number of outer batches |
| `--threshold` | float | 0.03 | Absolute Spearman correlation threshold for filtering |
| `--correlation` | bool | True | Compute Spearman correlation and Mode columns |
| `--seed` | int | 1234 | Random seed |

**Output columns:** `RBP`, `Target`, `Importance`, `Correlation`, `Mode`

---

## Step 4: getMerge_GRN {#step-4-getMerge_GRN}

Merge GRN results across **N seeds** for a robust consensus network. Uses a glob pattern to match all seed output files.

```bash
scRBP getMerge_GRN \
  --pattern "grn_seed*.tsv" \
  --output merged_grn.tsv \
  --n_present 10 \
  --present_rate 0.3 \
  --corr-threshold 0.0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--pattern` | str | required | Glob pattern matching all seed GRN `.tsv` files (e.g. `"grn_seed*.tsv"`) |
| `--output` | path | required | Output merged consensus GRN `.tsv` |
| `--corr-threshold` | float | 0.0 | Filter edges with `abs(mean Correlation)` ≤ threshold |
| `--n_present` | int | 10 | Minimum number of seed runs in which an edge must appear |
| `--present_rate` | float | 0.3 | Minimum presence rate (n\_present / N\_runs) to keep an edge |

**Output columns:** `RBP`, `Target`, `mean_Importance`, `mean_Correlation`, `n_present`, `present_rate`, `Mode`

---

## Step 5: getModule {#step-5-getModule}

Extract regulon candidate modules from the merged GRN using multiple selection strategies (Top-N and percentile-based).

```bash
scRBP getModule \
  --input merged_grn.tsv \
  --output_merged modules.tsv \
  --importance_threshold 0.005 \
  --top_n_list "5,10,50" \
  --target_top_n "50" \
  --percentile "0.75,0.9"
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | Merged GRN `.tsv` from `getMerge_GRN` |
| `--output_merged` | path | required | Output merged modules `.tsv` |
| `--importance_threshold` | float | 0.005 | Minimum importance score to retain an edge |
| `--top_n_list` | str | `"5,10,50"` | Comma-separated Top-N values for target selection |
| `--target_top_n` | str | `"50"` | Target Top-N for merged module output |
| `--percentile` | str | `"0.75,0.9"` | Comma-separated percentile thresholds for importance |
| `--verbose` | flag | False | Enable verbose logging |

---

## Step 6: getPrune {#step-6-getPrune}

Filter regulon candidates using **motif-binding enrichment** via ctxcore (NES scoring). Requires pre-built motif annotation and genome ranking databases.

```bash
scRBP getPrune \
  --rbp_targets modules.tsv \
  --motif_rbp_links motif_rbp_links.feather \
  --motif_target_ranks hg38_500bp_rankings.feather \
  --save_dir pruned_results/ \
  --rank_threshold 1500 \
  --nes_threshold 3.0 \
  --min_genes 20
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `--rbp_targets` | path | required | Module `.tsv` from `getModule` |
| `--motif_rbp_links` | path | required | Motif-RBP annotation `.feather` (maps motifs to RBPs) |
| `--motif_target_ranks` | path | required | Rankings `.feather` (e.g. `homo_sapiens_616RBPs_20746motifs_gene_rank_3UTR.feather`) |
| `--save_dir` | path | required | Output directory for pruned scores (Parquet) |
| `--rank_threshold` | int | 1500 | Top-N rank cutoff for motif target enrichment |
| `--auc_threshold` | float | 0.05 | AUC threshold for enrichment significance |
| `--nes_threshold` | float | 3.0 | Normalized Enrichment Score (NES) threshold |
| `--min_genes` | int | 20 | Minimum number of target genes to retain a regulon |
| `--n_jobs` | int | all CPUs | Number of parallel processes |
| `--chunksize` | int | 4 | Chunk size for multiprocessing imap |
| `--only_rbp` | str | None | Restrict pruning to a specific RBP (for debugging) |
| `--only_strategy` | str | None | Restrict to a specific selection strategy |

> Download motif databases from [scRBP resources](https://pyscenic.readthedocs.io/en/latest/installation.html#auxiliary-datasets).

---

## Step 7: getRegulon {#step-7-getRegulon}

Convert pruned ctxcore scores to standard **GMT files** in both gene-symbol and Entrez-ID formats.

```bash
scRBP getRegulon \
  --input pruned_results/ctx_scores.csv \
  --out-symbol regulons_symbol.gmt \
  --out-entrez regulons_entrez.gmt \
  --min_genes 1 \
  --taxid 9606
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | Pruned ctxcore `.csv` file from `getPrune` |
| `--out-symbol` | path | required | Output GMT file (gene symbols) |
| `--out-entrez` | path | required | Output GMT file (Entrez IDs) |
| `--rbp_col` | str | `RBP` | Column name for RBP in the input CSV |
| `--genes_col` | str | auto | Column name for target genes (auto-detected if omitted) |
| `--min_genes` | int | 1 | Minimum number of targets to retain a regulon |
| `--taxid` | int | 9606 | NCBI Taxonomy ID (9606 = human, 10090 = mouse) |
| `--map-hgnc` | path | None | HGNC gene mapping table for symbol→Entrez conversion |
| `--map-ncbi` | path | None | NCBI gene info file for symbol→Entrez conversion |
| `--drop-unmapped-genes` | flag | False | Drop genes that cannot be mapped to Entrez IDs |
| `--drop-empty-sets` | flag | False | Drop regulons with zero genes after mapping |

**GMT format:** `RBP_name <TAB> description <TAB> gene1 <TAB> gene2 ...`

---

## Step 8: mergeRegulons {#step-8-mergeRegulons}

Merge region-specific GMT files (3'UTR, 5'UTR, CDS, Introns) from multiple run directories into a single unified regulon set.

```bash
scRBP mergeRegulons \
  --base_dir results/ \
  --input regulons_symbol.gmt \
  --output merged_regulons.gmt \
  --region_order 3UTR 5UTR CDS Introns \
  --dedup_lines
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--base_dir` | path | required | Base directory containing region subdirectories |
| `--input` | str | required | Input GMT filename to find in each region directory |
| `--output` | str | required | Output merged GMT filename |
| `--region_order` | list | `3UTR 5UTR CDS Introns` | Order in which regions are merged |
| `--region_glob` | str | `Results_final_*_RBP_top1500_*` | Glob pattern for region-specific subdirectories |
| `--tissue_glob` | str | `z_GRNBoost2_*_30times` | Glob pattern for parent tissue dirs (with `--recursive`) |
| `--recursive` | flag | False | Recursively process multiple parent directories |
| `--dedup_lines` | flag | False | Deduplicate identical GMT lines |
| `--overwrite` | flag | False | Overwrite existing output files |
| `--summary_out` | path | None | Optional output `.tsv` for region-level summary table |

---

## Step 9: ras {#step-9-ras}

Compute **Regulon Activity Scores (RAS)** using the AUCell algorithm. Supports single-cell (`--mode sc`) and cell-type aggregated (`--mode ct`) modes.

```bash
# Single-cell mode
scRBP ras \
  --mode sc \
  --matrix data.h5ad \
  --regulons merged_regulons.gmt \
  --out ras_sc.csv

# Cell-type mode (requires cell-type annotation)
scRBP ras \
  --mode ct \
  --matrix data.h5ad \
  --regulons merged_regulons.gmt \
  --celltypes-csv celltypes.csv \
  --out ras_ct.csv
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--mode` | str | `ct` | Scoring mode: `sc` (per cell) or `ct` (per cell type) |
| `--matrix` | path | required | Expression matrix (`.h5ad` / `.feather` / `.loom` / `.csv`) |
| `--regulons` | path | required | Regulon GMT file from `mergeRegulons` |
| `--out` | path | required | Output RAS file / prefix |
| `--out_format` | str | `loom` | Output format: `csv`, `loom`, or `both`. |
| `--no-csv` | flag | False | Force disable writing CSV even if `--out_format` includes csv. |
| `--no-loom` | flag | False | Force disable writing LOOM even if `--out_format` includes loom. |
| `--csv_layout` | str | `regulons_by_cells` | CSV orientation: `regulons_by_cells`, `cells_by_regulons`, or `both`. |
| `--celltypes-csv` | path | None | CSV with `cell_id`, `cell_type` columns (required for `--mode ct`). |
| `--n_workers` | int | 4 | Number of workers for AUCell. |
| `--min_genes` | int | 1 | Drop regulons with fewer than `min_genes` targets. |
| `--to_upper` | flag | False | Uppercase gene symbols when matching to regulons. |
| `--emit-expr-stats` | flag | on | Emit `<out>_expr_stats.tsv` (per-gene `mean_expr`, `pct_detected`) reused by `rgs` / `rgs_rare` for null matching. |
| `--no-expr-stats` | flag | off | Disable expr-stats output. |
| `--expr-stats-out` | path | auto | Custom path for the expr-stats TSV. |

---

## Step 10: rgs {#step-10-rgs}

**MAGMA gene-set analysis** for GWAS enrichment. Computes Regulon-level Genetic association Scores (RGS) with matched null regulons controlling for 4 confounders: number of SNPs, number of parameters, mean gene expression, and percent detected.

```bash
scRBP rgs \
  --mode ct \
  --magma /path/to/magma \
  --genes-raw gwas.genes.raw \
  --sets merged_regulons_entrez.gmt \
  --id-type entrez \
  --out rgs_output \
  --n-null 1000 \
  --seed 2025
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--mode` | str | required | `sc` (single-cell) or `ct` (cell-type) |
| `--magma` | path | required | Path to MAGMA binary |
| `--genes-raw` | path | required | MAGMA `<prefix>.genes.raw` from prior gene analysis |
| `--sets` | path | required | Regulon GMT file (symbol or Entrez format) |
| `--id-type` | str | `entrez` | Gene ID format in GMT: `entrez` or `symbol` |
| `--out` | str | required | Output file prefix |
| `--n-null` | int | 1000 | Number of matched null regulons to generate |
| `--seed` | int | 2025 | Random seed for null sampling |
| `--q-bins` | int | 10 | Number of quantile bins for null matching |
| `--threads` | int | auto | CPU threads for MAGMA |
| `--gene-loc` | path | None | MAGMA `NCBI*.gene.loc` file (for symbol→Entrez mapping) |
| `--min_genes` | int | 0 | Minimum regulon size for inclusion |
| `--cleanup-out` | bool | True | Remove intermediate MAGMA output files |
| `--expr-stats` | path | None | Precomputed expression stats TSV (`symbol`, `mean_expr`, `pct_detected`) |

> Requires MAGMA binary. See [GWASTutorial](https://cloufield.github.io/GWASTutorial/09_Gene_based_analysis/) and download from [CNCR](https://cncr.nl/research/magma/).

---

## Step 11: rgs_rare {#step-11-rgs-rare}

**Rare-variant** analogue of `rgs`. Instead of MAGMA common-variant gene results, `rgs_rare` accepts gene-level rare-variant evidence from external frameworks — **TADA / extTADA**, SCHEMA-style burden, SAIGE-GENE+, REGENIE, STAAR-O — and fits a competitive gene-set regression:

$$
\text{rare\_score}_g = \beta_0 + \beta_s \cdot I(g \in \text{regulon}_s) + \gamma \cdot C_g + \varepsilon_g
$$

`rare_score_g` is a winsorised and z-standardised gene-level rare score. For frequentist inputs it is `-log10(P)`; for TADA-like Bayesian inputs it is `logBF`. The regression covariate `C_g` is kept **purely genetic** (the coding-opportunity term `z_log_union_CDS_length`), so the primary regression is not contaminated by expression signal. Expression covariates (`mean_expr`, `pct_detected`) are used **only for covariate-matched null regulon construction** in `--mode ct`, mirroring `rgs`.

`RGS_z = beta_s / SE(beta_s)`. Output columns and file naming mirror `rgs`, so downstream `scRBP trs` consumes the rare RGS with **no additional plumbing** — a **rare-variant TRS** is obtained by pointing `trs --rgs-csv` at the rare `.gsa_RGS.csv`.

**Two modes:**

- **`--mode sc`** — test real regulons only; write a MAGMA-like RGS CSV.
- **`--mode ct`** — additionally build covariate-matched null regulons (3-D matching on CDS length × mean expression × detection rate) and emit `RBP__REAL` / `RBP__NULL_XXXX` rows directly compatible with `scRBP trs`.

> `rgs_rare` does **not** perform primary rare-variant association testing — it consumes gene-level rare summary statistics produced upstream (TADA, burden, SAIGE-GENE+, STAAR-O, …).

```bash
# Single-cell with TADA-like logBF input
scRBP rgs_rare \
  --mode          sc \
  --rare-summary  tada_asd_gene_scores.tsv \
  --rare-gene-col gene_symbol \
  --rare-id-type  symbol \
  --score-mode    logbf --logbf-col logBF \
  --sets          regulons_symbol.gmt --id-type symbol \
  --out           rgs_rare_out/asd_sc

# Cell-type with SAIGE-GENE+ / burden P-values, reusing ras --emit-expr-stats
scRBP rgs_rare \
  --mode          ct \
  --rare-summary  saige_gene_burden.tsv \
  --rare-gene-col gene \
  --score-mode    pvalue --p-col P \
  --cds-length    gene_union_cds.tsv --cds-col union_cds_length --cds-scale raw \
  --sets          regulons_symbol.gmt --id-type symbol \
  --gene-loc      NCBI38.gene.loc \
  --expr-stats    ras_ct_output/scz_ct_expr_stats.tsv \
  --n-null 1000 --q-bins 10 \
  --out           rgs_rare_out/scz_ct
```

### Parameters — core

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--mode` | str | required | `sc` = real regulons only; `ct` = REAL + matched NULLs (feeds `trs`). |
| `--rare-summary` | path | required | Gene-level rare-variant summary CSV/TSV. |
| `--rare-gene-col` | str | required | Gene column in `--rare-summary`. |
| `--rare-id-type` | str | `symbol` | Gene ID type in the rare summary: `symbol` or `entrez`. |
| `--score-mode` | str | required | `pvalue` (uses `--p-col` → `-log10(P)`), `logbf` (uses `--logbf-col`), or `direct` (uses `--score-col` verbatim). |
| `--p-col` / `--logbf-col` / `--score-col` | str | — | Column matching the chosen `--score-mode`. |
| `--top-winsor` | float | 0.01 | Upper-tail winsorisation fraction for gene-level rare scores. |
| `--sets` | path | required | Regulon GMT (Symbol or Entrez). |
| `--id-type` | str | `symbol` | Gene ID type in `--sets`. |
| `--out` | str | required | Output file prefix. |
| `--gene-loc` | path | — | MAGMA `NCBI*.gene.loc` for Symbol ↔ Entrez mapping (required when input ID types differ). |
| `--min_genes` | int | 0 | Minimum overlap size for a regulon to be tested. |
| `--max-regulon-frac` | float | 0.5 | Skip regulons covering more than this fraction of the gene universe. |

### Parameters — CDS covariate

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--cds-length` | path | — | Optional CDS length table. If omitted, the CDS column is looked up in `--rare-summary`. |
| `--cds-gene-col` | str | `symbol` | Gene column in `--cds-length`. |
| `--cds-id-type` | str | `symbol` | Gene ID type in `--cds-length`. |
| `--cds-col` | str | `union_cds_length` | CDS covariate column name. |
| `--cds-scale` | str | `raw` | `raw` — raw union CDS length, internally `log1p` + z-scored. `z_log` — already a z-standardised log-CDS covariate; used as-is. |

### Parameters — matched-null construction (`--mode ct`)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--expr-stats` | path | — | Precomputed expression stats TSV. Reuses the file emitted by `ras --emit-expr-stats`. |
| `--emit-expr-stats` | bool | False | If True and `--expr-stats` is missing, compute from `--matrix-stats` and save. |
| `--matrix-stats` | path | — | Expression matrix used only when computing expr-stats on the fly. |
| `--n-null` / `--null` | int | 1000 | Number of matched null regulons per real regulon. |
| `--seed` | int | 2025 | Seed for null sampling. |
| `--q-bins` | int | 5 | Quantile bins for matched-null construction (use `10` for stricter matching). |
| `--exclude-self` | bool | True | Exclude real-regulon genes when sampling nulls. |
| `--min-bucket-size` | int | 5 | Minimum matched-bucket size before falling back to a coarser stratum. |
| `--fdr-method` | str | `BH` | FDR method for the empirical audit table (`BH` or `BY`). |
| `--save-null-gmt` | path | — | Path to save REAL+NULL GMT (working ID type). |
| `--save-null-gmt-symbol` / `--save-null-gmt-entrez` | path | — | Also save REAL+NULL GMT in Symbol / Entrez format. |

**Outputs**

| File | Description |
|------|-------------|
| `<out>.gsa_RGS.csv` | Regulon-level RGS table. `sc`: `Regulon, GeneSet, NGENES, BETA, BETA_STD, SE, P, RGS_z, RGS_mlog10P`. `ct`: additionally `RBP, SET_KIND, NULL_ID`. |
| `<out>_gene_scores.tsv` | Per-gene rare score with winsorised / z-scored columns and CDS covariate. |
| `<out>_REAL_PLUS_NULLS.<idtype>.gmt` | *(ct only)* REAL + NULL regulons; drop-in for `ras.py` if you wish to recompute RAS on the null sets. |
| `<out>.null_index.tsv` | *(ct only)* Index of every REAL/NULL set with sizes. |
| `<out>_empirical.csv` | *(ct only)* Audit table: `P_empirical`, `z_empirical`, `FDR_empirical`, `P_param`, `FDR_param`. |

---

## Step 12: trs {#step-12-trs}

Integrate RAS and RGS into a unified **Trait Relevance Score (TRS)**:

$$
\text{TRS} = \text{norm(RAS)} + \text{norm(RGS)} - \lambda \times |\text{norm(RAS)} - \text{norm(RGS)}|
$$

The same `trs` command consumes either the **common-variant** RGS from `rgs`
or the **rare-variant** RGS from `rgs_rare` — simply point `--rgs-csv` at the
CSV you want to integrate.

```bash
# Common-variant TRS
scRBP trs \
  --mode ct \
  --ras ras_out/scz.loom \
  --rgs-csv rgs_out/scz_real.csv \
  --out-prefix trs_out/scz_common \
  --lambda-penalty 1.0 --rgs-score mlog10p \
  --celltypes-csv celltypes.csv

# Rare-variant TRS — same command, different RGS file
scRBP trs \
  --mode ct \
  --ras ras_out/scz.loom \
  --rgs-csv rgs_rare_out/scz_ct.gsa_RGS.csv \
  --out-prefix trs_out/scz_rare \
  --lambda-penalty 1.0 --rgs-score mlog10p \
  --celltypes-csv celltypes.csv
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--mode` | str | required | `sc` (single-cell) or `ct` (cell-type) |
| `--ras` | path | required | RAS matrix from `ras` (`.csv` / `.loom`) |
| `--rgs-csv` | path | required | RGS `.csv` — common-variant from `rgs` **or** rare-variant `<out>.gsa_RGS.csv` from `rgs_rare` |
| `--out-prefix` | str | required | Output file prefix |
| `--rgs-score` | str | `mlog10p` | RGS score column to use: `mlog10p` or `z` |
| `--lambda-penalty` | float | 1.0 | Penalty for RAS–RGS divergence (λ in TRS formula) |
| `--q-hi-ras` | float | 0.99 | Upper quantile cap for RAS normalization |
| `--q-hi-rgs` | float | 0.99 | Upper quantile cap for RGS normalization |
| `--do-fdr` | int | 1 | Apply BH-FDR correction (`1`=yes, `0`=no; CT mode only) |
| `--celltypes-csv` | path | None | CSV with `cell_id`, `cell_type` columns (CT mode) |
| `--min_cells_pert_ct` | int | 25 | Minimum cells per cell type for CT mode |
