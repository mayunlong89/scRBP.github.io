---
title: "Pipeline"
permalink: /pipeline/
toc: true
toc_sticky: true
---

scRBP implements a **10-step command-line pipeline** that takes raw single-cell RNA-seq data
and produces disease-relevant RBP regulon rankings.

```
Raw scRNA-seq (.h5ad / .feather)
        ↓
  [1] getSketch       → Stratified subsampling (Optional)
  [2] getGRN          → GRN inference (GRNBoost2/GENIE3, --mode gene/isoform)
  [3] getMerge_GRN    → Consensus merging (N seeds, default 30)
  [4] getModule       → Regulon candidate extraction
  [5] getPrune        → Motif enrichment pruning
  [6] getRegulon      → GMT file generation (symbol + Entrez)
  [7] mergeRegulons   → Merge 4 region GMT files (3UTR/5UTR/CDS/Introns)
        ↓
  [8] ras             → Regulon Activity Score (RAS, --mode sc/ct)
  [9] rgs             → Regulon-level Genetic association Score (RGS, --mode sc/ct)
  [10] trs            → Trait Relevance Score (TRS, --mode sc/ct)
        ↓
   Disease-relevant RBP rankings
```

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

## Step 2: getGRN {#step-2-getGRN}

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

## Step 3: getMerge_GRN {#step-3-getMerge_GRN}

Merge GRN results across **N seeds** for a robust consensus network. Uses a glob pattern to match all seed output files.

```bash
scRBP getMerge_GRN \
  --pattern "grn_seed*.tsv" \
  --output merged_grn.tsv \
  --present_rate 0.0 \
  --corr-threshold 0.0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--pattern` | str | required | Glob pattern matching all seed GRN `.tsv` files (e.g. `"grn_seed*.tsv"`) |
| `--output` | path | required | Output merged consensus GRN `.tsv` |
| `--corr-threshold` | float | 0.0 | Filter edges with `abs(mean Correlation)` ≤ threshold |
| `--n_present` | int | 0 | Minimum number of seed runs in which an edge must appear |
| `--present_rate` | float | 0.0 | Minimum presence rate (n\_present / N\_runs) to keep an edge |

**Output columns:** `RBP`, `Target`, `mean_Importance`, `mean_Correlation`, `n_present`, `present_rate`, `Mode`

---

## Step 4: getModule {#step-4-getModule}

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

## Step 5: getPrune {#step-5-getPrune}

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
|-----------|------|---------|-------------|
| `--rbp_targets` | path | required | Module `.tsv` from `getModule` |
| `--motif_rbp_links` | path | required | Motif–RBP annotation `.feather` (maps motifs to RBPs) |
| `--motif_target_ranks` | path | required | Target gene/isoform rankings `.feather` (e.g. homo_sapiens_616RBPs_20746motifs_gene_rank_3UTR.feather`) |
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

## Step 6: getRegulon {#step-6-getRegulon}

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

## Step 7: mergeRegulons {#step-7-mergeRegulons}

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

## Step 8: ras {#step-8-ras}

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
| `--out` | path | required | Output RAS file |
| `--out_format` | str | `csv` | Output format: `csv`, `loom`, or `both` |
| `--celltypes-csv` | path | None | CSV with `cell_id`, `cell_type` columns (required for `--mode ct`) |
| `--n_workers` | int | 4 | Number of workers for AUCell |
| `--min_genes` | int | 1 | Drop regulons with fewer than `min_genes` targets |
| `--to_upper` | flag | False | Uppercase gene symbols when matching to regulons |

---

## Step 9: rgs {#step-9-rgs}

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

## Step 10: trs {#step-10-trs}

Integrate RAS and RGS into a unified **Trait Relevance Score (TRS)**:

$$
\text{TRS} = \text{norm(RAS)} + \text{norm(RGS)} - \lambda \times |\text{norm(RAS)} - \text{norm(RGS)}|
$$

```bash
scRBP trs \
  --mode ct \
  --ras ras_ct.csv \
  --rgs-csv rgs_output.csv \
  --out-prefix trs_results \
  --lambda-penalty 1.0 \
  --rgs-score mlog10p
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--mode` | str | required | `sc` (single-cell) or `ct` (cell-type) |
| `--ras` | path | required | RAS `.csv` from `ras` step |
| `--rgs-csv` | path | required | RGS `.csv` from `rgs` step |
| `--out-prefix` | str | required | Output file prefix |
| `--rgs-score` | str | `mlog10p` | RGS score column to use: `mlog10p` or `z` |
| `--lambda-penalty` | float | 1.0 | Penalty for RAS–RGS divergence (λ in TRS formula) |
| `--q-hi-ras` | float | 0.99 | Upper quantile cap for RAS normalization |
| `--q-hi-rgs` | float | 0.99 | Upper quantile cap for RGS normalization |
| `--do-fdr` | int | 1 | Apply BH-FDR correction (`1`=yes, `0`=no; CT mode only) |
| `--celltypes-csv` | path | None | CSV with `cell_id`, `cell_type` columns (CT mode) |
| `--min_cells_pert_ct` | int | 25 | Minimum cells per cell type for CT mode |
