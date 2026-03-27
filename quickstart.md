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

scRBP accepts single-cell expression data in `.h5ad` or `.feather` format:

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

### Gene-level mode (`--mode gene`)

```bash
# Step 1: Downsample large datasets (optional, recommended for >300,000 cells)
scRBP getSketch \
  --input data_normalized.h5ad \
  --output sketch.feather \
  --n_cells 50000 \
  --celltype_col celltype \
  --min_cells_per_type 50

# Step 2: Infer GRN (run 30 seeds for robustness)
for seed in {1..30}; do
  scRBP getGRN \
    --matrix sketch.feather \
    --rbp_list rbp_list.txt \
    --output grn_seed${seed}.tsv \
    --mode gene \
    --seed $seed
done

# Step 3: Merge consensus GRN from all seeds
scRBP getMerge_GRN \
  --pattern "grn_seed*.tsv" \
  --output merged_grn.tsv

# Step 4: Extract regulon candidate modules
scRBP getModule \
  --input merged_grn.tsv \
  --output_merged modules.tsv

# Step 5: Prune using motif enrichment (ctxcore)
scRBP getPrune \
  --rbp_targets modules.tsv \
  --motif_rbp_links motif_rbp_links.feather \
  --motif_target_ranks hg38_500bp_rankings.feather \
  --save_dir pruned_results/

# Step 6: Generate GMT files
scRBP getRegulon \
  --input pruned_results/ctx_scores.csv \
  --out-symbol regulons_symbol.gmt \
  --out-entrez regulons_entrez.gmt

# Step 7: Merge region-specific GMT files (3UTR / 5UTR / CDS / Introns)
scRBP mergeRegulons \
  --base_dir results/ \
  --input regulons_symbol.gmt \
  --output merged_regulons.gmt

# Step 8: Compute Regulon Activity Scores (single-cell mode)
scRBP ras \
  --mode sc \
  --matrix data_normalized.h5ad \
  --regulons merged_regulons.gmt \
  --out ras_sc.csv
```

### Isoform-level mode (`--mode isoform`)

```bash
scRBP getGRN \
  --matrix sketch.feather \
  --rbp_list rbp_list.txt \
  --output grn_isoform_seed1.tsv \
  --mode isoform \
  --isoform_annotation isoform2gene.txt \
  --seed 1
```

### Cell-type aggregated mode (`--mode ct` for ras/rgs/trs)

```bash
scRBP ras \
  --mode ct \
  --matrix data_normalized.h5ad \
  --regulons merged_regulons.gmt \
  --celltypes-csv celltypes.csv \
  --out ras_ct.csv
```

---

## 4. GWAS Disease Enrichment (Optional)

```bash
# Step 9: GWAS enrichment via MAGMA
scRBP rgs \
  --mode ct \
  --magma /path/to/magma \
  --genes-raw gwas.genes.raw \
  --sets merged_regulons_entrez.gmt \
  --id-type entrez \
  --out rgs_output \
  --n-null 1000

# Step 10: Compute Trait Relevance Score
scRBP trs \
  --mode ct \
  --ras ras_ct.csv \
  --rgs-csv rgs_output.csv \
  --out-prefix trs_results \
  --lambda-penalty 1.0
```

---

## 5. Output Files

| File | Description |
|------|-------------|
| `merged_regulons.gmt` | RBP regulon gene sets (symbol format) |
| `merged_regulons_entrez.gmt` | RBP regulon gene sets (Entrez format) |
| `ras_sc.csv` | Regulon Activity Scores (RAS) per cell |
| `ras_ct.csv` | Regulon Activity Scores (RAS) per cell type |
| `rgs_output.csv` | Regulon-level Genetic association Scores (RGS) |
| `trs_results.csv` | Integrated Trait Relevance Score (TRS) per regulon |

---

## Next Steps

- [Full Pipeline Documentation](/pipeline/)
- [API Reference](/api/)
- [Tutorials](/tutorials/)
