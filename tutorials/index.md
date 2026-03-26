---
title: "Tutorials"
permalink: /tutorials/
toc: true
toc_sticky: true
---

Step-by-step tutorials for common scRBP use cases.

---

## Tutorial 1: End-to-End RBP Regulon Inference

A complete walkthrough from raw scRNA-seq data to final regulons.

**Data:** Human PBMC dataset (`.h5ad`)
**Goal:** Identify active RBP regulons across cell types

```bash
# 1. Preprocess and downsample
scRBP getSketch --input pbmc.h5ad --output pbmc_sketch.h5ad --n_cells 50000

# 2. Infer GRN with 30 seeds
for seed in $(seq 1 30); do
  scRBP getGRN --input pbmc_sketch.h5ad \
               --output seeds/grn_seed${seed}.tsv \
               --seed $seed
done

# 3. Merge
scRBP getMerge_GRN --input_dir seeds/ --output merged_grn.tsv

# 4. Build regulons
scRBP getModule  --input merged_grn.tsv --output modules/
scRBP getPrune   --input modules/ --output pruned/ \
                 --motif_db hg38_motifs.feather \
                 --rankings_db hg38_rankings.feather
scRBP getRegulon --input pruned/ --output pbmc_regulons.gmt

# 5. Score activity
scRBP ras --input pbmc.h5ad --regulons pbmc_regulons.gmt --output pbmc_ras.csv
```

---

## Tutorial 2: Disease Trait Relevance Scoring

Link RBP regulons to GWAS traits using MAGMA.

**Prerequisites:** Complete Tutorial 1 first.
**Required:** MAGMA binary installed.

```bash
# Compute GWAS enrichment scores
scRBP rgs \
  --regulons pbmc_regulons.gmt \
  --gwas_dir /data/gwas_sumstats/ \
  --output pbmc_rgs.csv

# Integrate into Trait Relevance Scores
scRBP trs \
  --ras pbmc_ras.csv \
  --rgs pbmc_rgs.csv \
  --output pbmc_trs.csv \
  --lambda 0.5
```

The output `pbmc_trs.csv` ranks each RBP regulon by its relevance to each GWAS trait.

---

## Tutorial 3: Cell-Type Mode Analysis

For bulk-like aggregated analysis per cell type.

```bash
scRBP getGRN --input sketch.h5ad --output grn_ct.tsv --mode ct
scRBP ras    --input data.h5ad   --regulons regulons.gmt \
             --output ras_ct.csv --mode ct
```

---

## More Tutorials

Additional tutorials will be added covering:
- Isoform-level regulatory network inference
- Visualization of RBP regulon activity in UMAP
- Integration with Scanpy/Seurat workflows
- Comparative analysis across datasets

[View on GitHub](https://github.com/mayunlong89/scRBP){: .btn .btn--primary}
