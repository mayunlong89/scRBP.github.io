---
title: "scRBP"
layout: splash
permalink: /
header:
  overlay_color: "#1a6496"
  overlay_filter: 0.6
  actions:
    - label: "<i class='fas fa-download'></i> pip install scRBP"
      url: "/installation/"
    - label: "<i class='fab fa-github'></i> GitHub"
      url: "https://github.com/mayunlong89/scRBP"
    - label: "<i class='fas fa-box'></i> PyPI"
      url: "https://pypi.org/project/scRBP"
excerpt: >
  **Single-Cell RNA-Binding Protein Regulon Inference** <br>
  A systematic, scalable and integrative framework to infer RBP-mediated
  gene and isoform regulatory networks from single-cell transcriptomes.

intro:
  - excerpt: >
      scRBP is a command-line toolkit that integrates GRN inference, motif-based pruning,
      regulon activity quantification, and parallel common- and rare-variant genetic
      association models into a unified 12-step pipeline — from raw scRNA-seq data to
      trait-relevant RBP-regulon rankings.

feature_row:
  - title: "<i class='fas fa-project-diagram'></i> RBP Regulon Inference"
    excerpt: >
      Construct high-confidence RBP–gene and RBP–isoform regulatory networks
      using GRNBoost2/GENIE3 with motif-binding evidence pruning via ctxcore.
      Supports gene-level (`--mode gene`) and isoform-level (`--mode isoform`) inference,
      plus optional mini-metacell densification for sparse inputs.
    url: "/pipeline/"
    btn_label: "View Pipeline"
    btn_class: "btn--primary"
  - title: "<i class='fas fa-chart-bar'></i> Regulon Activity Scoring"
    excerpt: >
      Quantify RBP regulon activity per cell or per cell type using the AUCell algorithm (RAS).
      Cell type-level activity is aggregated via a Jensen–Shannon Regulon Specificity Score
      (RSS) that highlights context-specific regulons.
    url: "/pipeline/#step-9-ras"
    btn_label: "Learn More"
    btn_class: "btn--primary"
  - title: "<i class='fas fa-dna'></i> Common- and Rare-Variant Regulons"
    excerpt: >
      Link both common-variant GWAS signals (MAGMA) and rare-variant burden / TADA / logBF
      evidence to RBP regulons (RGS), then integrate with RAS to produce a unified
      Trait Relevance Score (TRS) with matched-null empirical significance.
    url: "/pipeline/#step-10-rgs"
    btn_label: "Learn More"
    btn_class: "btn--primary"
---

<p align="center">
  <img src="https://raw.githubusercontent.com/mayunlong89/scRBP/main/Examples/scRBP_logo.png" width="220" alt="scRBP Logo">
</p>

<p align="center">
  <a href="https://pypi.org/project/scRBP"><img src="https://img.shields.io/badge/pypi-0.1.4.1-green" alt="PyPI"></a>
  <a href="https://pypi.org/project/scRBP"><img src="https://img.shields.io/badge/python-3.9--3.11-blue" alt="Python"></a>
  <a href="https://github.com/mayunlong89/scRBP/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-yellow" alt="License"></a>
</p>

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

---

## What scRBP Does

**scRBP** (*single-cell RNA-binding protein regulon inference*) is a command-line toolkit for reconstructing RBP-mediated regulatory programs from single-cell transcriptomic data and prioritizing regulons associated with complex traits and disorders. It supports both **gene-level** networks from short-read data and **isoform-level** networks from long-read data.

The framework integrates six major analytical components:

- **RBP compendium** — A curated database of RBPs and clustered RBP-binding motifs assembled from public resources.
- **Target inference** — Motif-guided transcriptome-wide rankings of candidate RBP targets at gene and isoform resolution.
- **Network construction** — Inference of RBP–gene or RBP–isoform association networks from single-cell expression data (with optional mini-metacell densification for sparse inputs).
- **Regulon scoring** — Motif-based refinement of candidate edges and quantification of Regulon Activity Scores (RAS) at single-cell or cell type resolution.
- **Common- and rare-variant genetics** — Parallel models that map common-variant GWAS signals (**MAGMA**) *and* gene-level rare-variant evidence (**TADA / SAIGE-GENE+ / burden / STAAR-O**) to regulons, yielding Regulon-level Genetic Association Scores (RGS).
- **Trait relevance** — Integration of RAS with either common- or rare-variant RGS into a Trait-Relevance Score (TRS), with significance evaluated against matched-null regulons via Monte Carlo sampling.

---

## Pipeline at a Glance

```
Raw single-cell data (.h5ad / .feather)
          │
          ▼
[Step 1]  scRBP getSketch        ── Stratified GeoSketch downsampling (optional)
          │
          ▼
[Step 2]  scRBP getMetacell      ── Aggregate similar cells into mini-metacells (optional;
          │                          for the GRN branch when dropout is heavy)
          ▼
[Step 3]  scRBP getGRN           ── GRNBoost2/GENIE3 RBP→gene or RBP→isoform inference
          │                          (run multiple random seeds; 30 runs recommended)
          ▼
[Step 4]  scRBP getMerge_GRN     ── Merge multi-seed GRNs into a consensus network
          │
          ▼
[Step 5]  scRBP getModule        ── Extract regulon candidates (Top-N / percentile)
          │
          ▼
[Step 6]  scRBP getPrune         ── Motif-enrichment pruning via ctxcore
          │
          ▼
[Step 7]  scRBP getRegulon       ── Export pruned regulons to GMT format
          │
          ▼
[Step 8]  scRBP mergeRegulons    ── Merge region-specific GMT files
          │                          (3'UTR / 5'UTR / CDS / Introns)
          ▼
[Step 9]  scRBP ras              ── Regulon Activity Score (RAS) per cell / cell type
          │                          (--mode sc | --mode ct)
          ▼
[Step 10] scRBP rgs              ── Common-variant RGS via MAGMA
          │                          (--mode sc | --mode ct)
          ▼
[Step 11] scRBP rgs_rare         ── Rare-variant RGS via competitive gene-set regression
          │                          (TADA / logBF / burden; --mode sc | --mode ct)
          ▼
[Step 12] scRBP trs              ── Trait-Relevance Score (RAS × RGS integration)
                                     (common- OR rare-variant; --mode sc | --mode ct)
```

> Steps 2 (**getMetacell**) and 11 (**rgs_rare**) are new in v0.1.4.1.
> `getMetacell` is optional and only used for the GRN branch on sparse inputs
> — `getSketch` remains the recommended thinning strategy for the RAS branch.
> `rgs_rare` runs in parallel to `rgs`; feed its output into `scRBP trs` to
> obtain a **rare-variant** TRS.

| Step | Command | Key Input | Key Output |
|------|---------|-----------|------------|
| 1 | `getSketch` | `.h5ad` / `.feather` | Downsampled cells |
| 2 | `getMetacell` *(optional, GRN branch)* | `.h5ad` (with cell-type col) / `.feather` | Metacell matrix (gene × metacell) + metacell→cell-type map |
| 3 | `getGRN` | Expression / metacell matrix, RBP list | `*_scRBP_gene_GRNs.tsv` |
| 4 | `getMerge_GRN` | Multiple GRN `.tsv` files | Consensus GRN `.tsv` |
| 5 | `getModule` | Consensus GRN `.tsv` | Modules `.tsv` |
| 6 | `getPrune` | Modules, motif databases | Pruned scores (Parquet) |
| 7 | `getRegulon` | Pruned scores | Regulons `.gmt` (symbol + Entrez) |
| 8 | `mergeRegulons` | Multiple `.gmt` files | Merged `.gmt` |
| 9 | `ras` | Expression matrix, `.gmt` | RAS matrix (`.csv` / `.loom`) + expr-stats TSV |
| 10 | `rgs` | MAGMA `.genes.raw`, `.gmt`, expr-stats | Common-variant RGS `.csv` |
| 11 | `rgs_rare` | Rare gene summary (TADA / burden / …), `.gmt`, expr-stats | Rare-variant RGS `.csv` (+ REAL/NULL GMT) |
| 12 | `trs` | RAS matrix, RGS `.csv` (common **or** rare) | TRS scores `.csv` |

[View Full Pipeline]({{ '/pipeline/' | relative_url }}){: .btn .btn--primary}
[API Reference]({{ '/api/' | relative_url }}){: .btn .btn--info}

---

## Quick Start

```bash
# Install
pip install scRBP

# 1) Downsample large datasets (recommended for >300,000 cells)
scRBP getSketch --input data.h5ad --output sketch.feather --n_cells 50000

# 2) (Optional) Densify the GRN branch with mini-metacells
scRBP getMetacell --input sketch.feather --output metacell.feather \
                  --metacell_size 10 --within_celltype --celltype_col celltype

# 3) Infer GRN (30 seeds for robustness; use the metacell matrix if produced)
for seed in {1..30}; do
  scRBP getGRN --matrix metacell.feather --rbp_list rbp_list.txt \
               --output grn_seed${seed}.tsv --seed $seed
done

# 4-8) Build consensus regulons
scRBP getMerge_GRN  --pattern "grn_seed*.tsv" --output merged_grn.tsv
scRBP getModule     --input merged_grn.tsv --output_merged modules.tsv
scRBP getPrune      --rbp_targets modules.tsv --motif_rbp_links motif_rbp_links.feather \
                    --motif_target_ranks rankings.feather --save_dir pruned/
scRBP getRegulon    --input pruned/ctx_scores.csv \
                    --out-symbol regulons_symbol.gmt --out-entrez regulons_entrez.gmt

# 9) Activity — always uses real single cells, not metacells
scRBP ras           --mode ct --matrix sketch.feather --regulons regulons_symbol.gmt \
                    --out ras_out/scz --celltypes-csv celltypes.csv --emit-expr-stats

# 10-11) Common- AND rare-variant regulon genetic association
scRBP rgs           --mode ct --magma /tools/magma --genes-raw gwas.genes.raw \
                    --sets regulons_entrez.gmt --id-type entrez \
                    --out rgs_out/scz --expr-stats ras_out/scz_expr_stats.tsv --n-null 1000
scRBP rgs_rare      --mode ct --rare-summary tada_gene_scores.tsv \
                    --rare-gene-col gene_symbol --score-mode logbf --logbf-col logBF \
                    --sets regulons_symbol.gmt --id-type symbol \
                    --expr-stats ras_out/scz_expr_stats.tsv \
                    --out rgs_rare_out/scz

# 12) Integrate into TRS — same command for common OR rare RGS
scRBP trs           --mode ct --ras ras_out/scz.loom \
                    --rgs-csv rgs_out/scz_real.csv \
                    --out-prefix trs_out/scz_common --celltypes-csv celltypes.csv
```

[Full Quick Start Guide]({{ '/quickstart/' | relative_url }}){: .btn .btn--primary .btn--large}
[Installation Guide]({{ '/installation/' | relative_url }}){: .btn .btn--info .btn--large}

---

## Citation

If you use scRBP in your research, please cite:

> Ma Y. *et al.* ***Decoding cell-specific RNA-binding protein regulatory networks across development and disease***. (2026)

---

## Links

- **GitHub**: [https://github.com/mayunlong89/scRBP](https://github.com/mayunlong89/scRBP)
- **PyPI**: [https://pypi.org/project/scRBP](https://pypi.org/project/scRBP)
- **Issues**: [https://github.com/mayunlong89/scRBP/issues](https://github.com/mayunlong89/scRBP/issues)

---

## License

scRBP is released under the [MIT License](https://github.com/mayunlong89/scRBP/blob/main/LICENSE).
