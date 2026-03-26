---
title: "Quick Start"
permalink: /quickstart/
toc: true
toc_sticky: true
---

## 1. Install scRBP

```bash
pip install scRBP
```

---

## 2. Prepare Input Data

scRBP accepts single-cell expression data in `.h5ad` format (AnnData):

```python
import scanpy as sc

# Load your scRNA-seq data
adata = sc.read_h5ad("your_data.h5ad")

# Recommended: log-normalize before running scRBP
sc.pp.normalize_total(adata, target_sum=1e4)
sc.pp.log1p(adata)

adata.write("data_normalized.h5ad")
```

---

## 3. Run the Core Pipeline

### Single-cell mode (`--mode sc`)

```bash
# Step 1: Downsample to ~50000 cells for GRN inference
scRBP getSketch --input data_normalized.h5ad --output sketch.h5ad --n_cells 50000

# Step 2: Infer GRN (run multiple seeds for robustness)
for seed in {1..30}; do
  scRBP getGRN --input sketch.h5ad --output grn_seed${seed}.tsv --seed $seed
done

# Step 3: Merge consensus GRN
scRBP getMerge_GRN --input_dir grn_seeds/ --output merged_grn.tsv

# Step 4-6: Extract and prune regulons
scRBP getModule --input merged_grn.tsv --output modules/
scRBP getPrune   --input modules/ --output pruned/ --motif_db motif.feather --rankings_db rankings.feather
scRBP getRegulon --input pruned/ --output regulons.gmt

# Step 8: Compute Regulon Activity Scores
scRBP ras --input data_normalized.h5ad --regulons regulons.gmt --output ras.csv
```

### Cell-type mode (`--mode ct`)

```bash
scRBP getGRN --input sketch.h5ad --output grn_ct.tsv --mode ct
scRBP ras    --input data_normalized.h5ad --regulons regulons.gmt --output ras_ct.csv --mode ct
```

---

## 4. GWAS Disease Enrichment (Optional)

```bash
# Step 9: GWAS enrichment via MAGMA
scRBP rgs \
  --regulons regulons.gmt \
  --gwas_dir /path/to/gwas_sumstats/ \
  --output rgs.csv

# Step 10: Compute Trait Relevance Score
scRBP trs \
  --ras ras.csv \
  --rgs rgs.csv \
  --output trs.csv
```

---

## 5. Output Files

| File | Description |
|------|-------------|
| `regulons.gmt` | RBP regulon gene sets |
| `ras.csv` | Regulon Activity Scores (RAS) per cell |
| `rgs.csv` | Regulon-level Genetic association Scores (RGS) per cell |
| `trs.csv` | Integrated Trait Relevance Score (TRS) per cell for each regulon |

---

## Next Steps

- [Full Pipeline Documentation](/pipeline/)
- [API Reference](/api/)
- [Tutorials](/tutorials/)
