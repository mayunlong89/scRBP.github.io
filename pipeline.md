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

## Step 1: getSketch

Stratified cell downsampling using **GeoSketch** to make large datasets tractable.

```bash
scRBP getSketch \
  --input data.h5ad \
  --output sketch.h5ad \
  --n_cells 50000
```

| Parameter | Description |
|-----------|-------------|
| `--input` | Input `.h5ad` file |
| `--output` | Output downsampled `.h5ad` |
| `--n_cells` | Number of cells to retain |

---

## Step 2: getGRN

Gene regulatory network inference using **GRNBoost2** or **GENIE3**.

```bash
scRBP getGRN \
  --input sketch.h5ad \
  --output grn.tsv \
  --method grnboost2 \
  --mode sc
```

| Parameter | Description |
|-----------|-------------|
| `--method` | `grnboost2` (default) or `genie3` |
| `--mode` | `sc` (single-cell) or `ct` (cell-type) |

---

## Step 3: getMerge_GRN

Merge GRN results across **N seeds** for robust consensus networks (default: 30 seeds).

```bash
scRBP getMerge_GRN \
  --input_dir grn_seeds/ \
  --output merged_grn.tsv \
  --n_seeds 30
```

---

## Step 4: getModule

Extract regulon candidate modules from the merged GRN.

```bash
scRBP getModule \
  --input merged_grn.tsv \
  --output modules/
```

---

## Step 5: getPrune

Filter regulon candidates using **motif-binding evidence** via ctxcore for high-confidence regulons.

```bash
scRBP getPrune \
  --input modules/ \
  --output pruned_modules/ \
  --motif_db motif_db.feather \
  --rankings_db rankings.feather
```

---

## Step 6: getRegulon

Generate **GMT files** (symbol and Entrez formats) from pruned regulons.

```bash
scRBP getRegulon \
  --input pruned_modules/ \
  --output regulons.gmt \
  --format symbol
```

---

## Step 7: mergeRegulons

Consolidate region-specific GMT files into a unified regulon set.

```bash
scRBP mergeRegulons \
  --input_dir regulon_regions/ \
  --output final_regulons.gmt
```

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

> Requires MAGMA binary. See [Installation](/installation/#optional-magma-for-gwas-enrichment).

---

## Step 10: trs

Integrate RAS and RGS into a unified **Trait Relevance Score (TRS)**:

$$\text{TRS} = \text{norm(RAS)} + \text{norm(RGS)} - \lambda \times |\text{norm(RAS)} - \text{norm(RGS)}|$$

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
