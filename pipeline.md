---
title: "Pipeline"
permalink: /pipeline/
toc: true
toc_sticky: true
---

scRBP implements a **10-step command-line pipeline** that takes raw single-cell RNA-seq data
and produces disease-relevant RBP regulon rankings.

```
Raw scRNA-seq (.h5ad)
        ↓
  [1] getSketch       → Stratified subsampling (Optional)
  [2] getGRN          → GRN inference (GRNBoost2/GENIE3)
  [3] getMerge_GRN    → Consensus merging (N seeds, default 30)
  [4] getModule       → Regulon candidate extraction
  [5] getPrune        → Motif enrichment pruning
  [6] getRegulon      → GMT file generation
  [7] mergeRegulons   → Region consolidation
        ↓
  [8] ras             → Regulon Activity Score (RAS)
  [9] rgs             → Regulon-level Genetic association Score (RGS)
  [10] trs            → Trait Relevance Score (TRS)
        ↓
   Disease-relevant RBP rankings
```

---

## Step 1: getSketch {#step-1-getSketch}

Stratified cell downsampling using **GeoSketch** to make large datasets tractable (Optional step).

```bash
scRBP getSketch \
  --input data.h5ad \
  --output sketch.feather \
  --n_cells 50000
```

| Parameter | Description |
|-----------|-------------|
| `--input` | Input `.h5ad` file |
| `--output` | Output downsampled `.feather` |
| `--n_cells` | Number of cells to retain |

---

## Step 2: getGRN {#step-2-getGRN}

Gene regulatory network inference using **GRNBoost2** or **GENIE3**.

```bash
scRBP getGRN \
  --input sketch.feather \
  --output grn.tsv \
  --method grnboost2 \
  --mode sc
```

| Parameter | Description |
|-----------|-------------|
| `--method` | `grnboost2` (default) or `genie3` |
| `--mode` | `sc` (single-cell) or `ct` (cell-type) |

---

## Step 3: getMerge_GRN {#step-3-getMerge_GRN}

Merge GRN results across **N seeds** for robust consensus networks (default: 30 seeds).
Running multiple seeds and taking the consensus reduces noise from stochastic GRN inference.

```bash
scRBP getMerge_GRN \
  --input_dir grn_seeds/ \
  --output merged_grn.tsv \
  --n_seeds 30
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input_dir` | path | required | Directory containing per-seed GRN `.tsv` files |
| `--output` | path | required | Output consensus GRN `.tsv` file |
| `--n_seeds` | int | 30 | Number of seed GRNs to merge |

**Input:** Directory of `grn_seed*.tsv` files (TF, target, importance columns)
**Output:** `merged_grn.tsv` — consensus edge list with aggregated importance scores

---

## Step 4: getModule {#step-4-getModule}

Extract regulon candidate modules from the merged GRN by grouping target genes per RBP.

```bash
scRBP getModule \
  --input merged_grn.tsv \
  --output modules/ \
  --top_targets 500
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | Consensus GRN `.tsv` from `getMerge_GRN` |
| `--output` | path | required | Output directory for module files |
| `--top_targets` | int | 500 | Max number of top target genes per RBP |

**Input:** `merged_grn.tsv` (RBP, target, importance)
**Output:** Per-RBP module files in `modules/` — one file per RBP containing its candidate target genes

---

## Step 5: getPrune {#step-5-getPrune}

Filter regulon candidates using **motif-binding evidence** via ctxcore for high-confidence regulons.
Requires pre-built motif annotation databases for your genome of interest (hg38 or mm10).

```bash
scRBP getPrune \
  --input modules/ \
  --output pruned_modules/ \
  --motif_db motif_db.feather \
  --rankings_db rankings.feather
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | Module directory from `getModule` |
| `--output` | path | required | Output directory for pruned regulons |
| `--motif_db` | path | required | Motif annotation `.feather` file (e.g. `hg38_motifs.feather`) |
| `--rankings_db` | path | required | Genome rankings `.feather` file (e.g. `hg38_500bp_rankings.feather`) |

**Input:** Module files + motif/rankings databases (download from [cisTarget resources](https://resources.aertslab.org/cistarget/))
**Output:** High-confidence regulons in `pruned_modules/` with motif enrichment scores

> Motif databases can be downloaded from the [pySCENIC resources page](https://pyscenic.readthedocs.io/en/latest/installation.html#auxiliary-datasets).

---

## Step 6: getRegulon {#step-6-getRegulon}

Generate **GMT files** (gene symbol and Entrez ID formats) from pruned regulons for downstream analysis.

```bash
scRBP getRegulon \
  --input pruned_modules/ \
  --output regulons.gmt \
  --format symbol
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input` | path | required | Pruned regulon directory from `getPrune` |
| `--output` | path | required | Output GMT file |
| `--format` | str | `symbol` | Gene ID format: `symbol` or `entrez` |

**Input:** Pruned regulon files from `getPrune`
**Output:** `regulons.gmt` — standard GMT format: `RBP_name \t description \t gene1 \t gene2 \t ...`

---

## Step 7: mergeRegulons {#step-7-mergeRegulons}

Consolidate region-specific GMT files (e.g. from different chromosomal regions or batches) into a single unified regulon set.

```bash
scRBP mergeRegulons \
  --input_dir regulon_regions/ \
  --output final_regulons.gmt
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--input_dir` | path | required | Directory containing region-specific `.gmt` files |
| `--output` | path | required | Merged output GMT file |

**Input:** Multiple `.gmt` files in `regulon_regions/`
**Output:** `final_regulons.gmt` — unified regulon set ready for `ras` scoring

---

## Step 8: ras {#step-8-ras}

Compute **Regulon Activity Scores (RAS)** using the AUCell algorithm.

```bash
scRBP ras \
  --input data.h5ad \
  --regulons final_regulons.gmt \
  --output ras.csv \
  --mode sc
```

| Parameter | Description |
|-----------|-------------|
| `--mode` | `sc` (per cell) or `ct` (per cell type) |

---

## Step 9: rgs {#step-9-rgs}

**MAGMA-based GWAS enrichment** analysis. Generates matched null distributions
controlling for 4 confounders: number of SNPs, parameters, mean expression, and percent detected.

```bash
scRBP rgs \
  --regulons final_regulons.gmt \
  --gwas_dir gwas_sumstats/ \
  --output rgs.csv \
  --magma_path /path/to/magma
```

> Requires MAGMA binary. See [GWASTutorial](https://cloufield.github.io/GWASTutorial/09_Gene_based_analysis/) and download MAGMA from [CNCR](https://cncr.nl/research/magma/).

---

## Step 10: trs {#step-10-trs}

Integrate RAS and RGS into a unified **Trait Relevance Score (TRS)**:


$$
\text{TRS} = \text{norm(RAS)} + \text{norm(RGS)} - \lambda \times |\text{norm(RAS)} - \text{norm(RGS)}|
$$


```bash
scRBP trs \
  --ras ras.csv \
  --rgs rgs.csv \
  --output trs.csv \
  --lambda 0.5
```

| Parameter | Description |
|-----------|-------------|
| `--lambda` | Penalty for RAS–RGS divergence (default: 0.5) |
