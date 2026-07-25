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

# Step 2: (Optional) Densify with mini-metacells for the GRN branch
# Recommended when the input is very sparse (heavy dropout). Feed the metacell
# matrix into getGRN; keep the real single-cell matrix for the RAS branch (Step 9).
scRBP getMetacell \
  --input             sketch.feather \
  --output            metacell.feather \
  --metacell_size     10 \
  --method            knn \
  --within_celltype \
  --celltype_col      celltype \
  --min_metacell_size 5

# Step 3: Infer GRN (run 30 seeds for robustness; use metacells if produced)
for seed in {1..30}; do
  scRBP getGRN \
    --matrix metacell.feather \
    --rbp_list rbp_list.txt \
    --output grn_seed${seed}.tsv \
    --mode gene \
    --seed $seed
done

# Step 4: Merge consensus GRN from all seeds
scRBP getMerge_GRN \
  --pattern "grn_seed*.tsv" \
  --output merged_grn.tsv

# Step 5: Extract regulon candidate modules
scRBP getModule \
  --input merged_grn.tsv \
  --output_merged modules.tsv

# Step 6: Prune using motif enrichment (ctxcore)
scRBP getPrune \
  --rbp_targets modules.tsv \
  --motif_rbp_links motif_rbp_links.feather \
  --motif_target_ranks hg38_500bp_rankings.feather \
  --save_dir pruned_results/

# Step 7: Generate GMT files
scRBP getRegulon \
  --input pruned_results/ctx_scores.csv \
  --out-symbol regulons_symbol.gmt \
  --out-entrez regulons_entrez.gmt

# Step 8: Merge region-specific GMT files (3UTR / 5UTR / CDS / Intronic)
scRBP mergeRegulons \
  --base_dir results/ \
  --input regulons_symbol.gmt \
  --output merged_regulons.gmt

# Step 9: Compute Regulon Activity Scores + expr-stats
# RAS always uses REAL single cells (not metacells) so cell-type resolution is preserved.
# --emit-expr-stats writes <out>_expr_stats.tsv for reuse by rgs / rgs_rare.
scRBP ras \
  --mode sc \
  --matrix data_normalized.h5ad \
  --regulons merged_regulons.gmt \
  --out ras_out/scz \
  --emit-expr-stats
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

## 4. Genetic Enrichment — Common **and** Rare Variants (Optional)

scRBP maps two complementary lines of genetic evidence onto each regulon.
The same downstream `trs` command consumes either RGS file.

```bash
# Step 10: Common-variant RGS via MAGMA (GWAS gene-level results)
scRBP rgs \
  --mode ct \
  --magma /path/to/magma \
  --genes-raw gwas.genes.raw \
  --sets merged_regulons_entrez.gmt \
  --id-type entrez \
  --out rgs_out/scz \
  --expr-stats ras_out/scz_expr_stats.tsv \
  --n-null 1000

# Step 11: Rare-variant RGS via competitive gene-set regression
# (TADA / logBF, SAIGE-GENE+ / burden P-values, STAAR-O, ...)
scRBP rgs_rare \
  --mode          ct \
  --rare-summary  tada_gene_scores.tsv \
  --rare-gene-col gene_symbol \
  --score-mode    logbf --logbf-col logBF \
  --sets          merged_regulons.gmt --id-type symbol \
  --gene-loc      NCBI38.gene.loc \
  --expr-stats    ras_out/scz_expr_stats.tsv \
  --n-null 1000 --q-bins 10 \
  --out           rgs_rare_out/scz

# Step 12: Trait Relevance Score — one command, two flavours
# 12a) Common-variant TRS
scRBP trs \
  --mode ct \
  --ras ras_out/scz.loom \
  --rgs-csv rgs_out/scz_real.csv \
  --out-prefix trs_out/scz_common \
  --lambda-penalty 1.0 \
  --celltypes-csv celltypes.csv

# 12b) Rare-variant TRS — same command, point at the rare RGS CSV
scRBP trs \
  --mode ct \
  --ras ras_out/scz.loom \
  --rgs-csv rgs_rare_out/scz.gsa_RGS.csv \
  --out-prefix trs_out/scz_rare \
  --lambda-penalty 1.0 \
  --celltypes-csv celltypes.csv
```

---

## 5. Output Files

| File | Description |
|------|-------------|
| `metacell.feather` | *(optional)* Densified gene × metacell matrix from `getMetacell` |
| `merged_regulons.gmt` | RBP regulon gene sets (symbol format) |
| `merged_regulons_entrez.gmt` | RBP regulon gene sets (Entrez format) |
| `ras_out/scz.loom` / `_regulons_by_cells.csv` | Regulon Activity Scores (RAS) per cell / cell type |
| `ras_out/scz_expr_stats.tsv` | Per-gene `mean_expr`, `pct_detected` — reused by `rgs` / `rgs_rare` for null matching |
| `rgs_out/scz_real.csv` | Common-variant Regulon-level Genetic Association Scores |
| `rgs_rare_out/scz.gsa_RGS.csv` | Rare-variant RGS (competitive regression on TADA / burden / etc.) |
| `trs_out/scz_common*.csv` | Trait Relevance Score using common-variant RGS |
| `trs_out/scz_rare*.csv` | Trait Relevance Score using rare-variant RGS |

---

