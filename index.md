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
      scRBP is a command-line toolkit that integrates GRN inference, motif enrichment,
      GWAS-based genetic scoring, and regulon activity quantification into a unified
      10-step pipeline — from raw scRNA-seq data to disease-relevant RBP rankings.

feature_row:
  - title: "<i class='fas fa-project-diagram'></i> RBP Regulon Inference"
    excerpt: >
      Construct high-confidence RBP–gene and RBP–isoform regulatory networks
      using GRNBoost2/GENIE3 with motif-binding evidence pruning via ctxcore.
      Supports both single-cell (`--mode sc`) and cell-type (`--mode ct`) modes.
    url: "/pipeline/"
    btn_label: "View Pipeline"
    btn_class: "btn--primary"
  - title: "<i class='fas fa-chart-bar'></i> Regulon Activity Scoring"
    excerpt: >
      Quantify RBP regulon activity per cell or per cell type using the AUCell algorithm (RAS).
      Statistical significance is assessed via Monte Carlo sampling with matched null regulons.
    url: "/pipeline/#step-8-ras"
    btn_label: "Learn More"
    btn_class: "btn--primary"
  - title: "<i class='fas fa-dna'></i> Disease-Associated Regulons"
    excerpt: >
      Link RBP regulons to human disease traits via MAGMA-based GWAS enrichment (RGS),
      then integrate with RAS to compute unified Trait Relevance Scores (TRS).
    url: "/pipeline/#step-9-rgs"
    btn_label: "Learn More"
    btn_class: "btn--primary"
---

<p align="center">
  <img src="https://raw.githubusercontent.com/mayunlong89/scRBP/main/Examples/scRBP_logo.png" width="220" alt="scRBP Logo">
</p>

<p align="center">
  <a href="https://pypi.org/project/scRBP"><img src="https://img.shields.io/badge/pypi-0.1.3-green" alt="PyPI"></a>
  <a href="https://pypi.org/project/scRBP"><img src="https://img.shields.io/badge/python-3.9--3.11-blue" alt="Python"></a>
  <a href="https://github.com/mayunlong89/scRBP/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-yellow" alt="License"></a>
</p>

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

---

## What scRBP Does

**scRBP** (*Single-cell RNA-Binding Protein Regulon Inference*) is a command-line toolkit for comprehensive analysis of RNA-binding proteins (RBPs) in single-cell RNA-seq data. It provides a systematic, scalable and integrative framework to infer RBP-mediated gene and isoform regulatory networks ("regulons") from single-cell transcriptomes and prioritize networks underlying complex genetic traits and disorders.

scRBP is comprised of six main modules:

- **RBP compendium** — A comprehensive database of RBPs and associated motif clusters from diverse public resources
- **Target inference** — Motif-guided transcriptome-wide inference of RBP targets at gene- and isoform-level resolution
- **Network construction** — RBP–gene and RBP–isoform co-expression networks from short- or long-read scRNA-seq data
- **Regulon scoring** — High-fidelity regulon definition and cell type-specific Regulon Activity Scores (RAS)
- **GWAS integration** — GWAS enrichment to compute Regulon-level Genetic association Scores (RGS)
- **Trait relevance** — Unified Trait Relevance Score (TRS) combining RAS and RGS per regulon per cellular context

---

## Pipeline at a Glance

```
Raw single-cell data (.h5ad / .feather)
          │
          ▼
[Step 1]  scRBP getSketch        ── Stratified GeoSketch cell downsampling (Optional)
          │
          ▼
[Step 2]  scRBP getGRN           ── GRNBoost2/GENIE3 RBP→Gene/Isoform inference
          │                          (run N seeds for robustness, default 30 times)
          ▼
[Step 3]  scRBP getMerge_GRN     ── Merge N-seed GRNs → consensus network
          │
          ▼
[Step 4]  scRBP getModule        ── Extract regulon candidates (Top-N / percentile)
          │
          ▼
[Step 5]  scRBP getPrune         ── Motif-enrichment pruning via ctxcore
          │
          ▼
[Step 6]  scRBP getRegulon       ── Export pruned regulons to GMT format
          │
          ▼
[Step 7]  scRBP mergeRegulons    ── Merge region-specific GMT files
          │                          (3'UTR / 5'UTR / CDS / Introns)
          ▼
[Step 8]  scRBP ras              ── Regulon Activity Score (RAS) per cell / cell type
          │                          (--mode sc | --mode ct)
          ▼
[Step 9]  scRBP rgs              ── Regulon-level Genetic association Score (RGS)
          │                          (--mode sc | --mode ct)
          ▼
[Step 10] scRBP trs              ── Trait Relevance Score (TRS = RAS + RGS)
                                     (--mode sc | --mode ct)
```

| Step | Command | Key Input | Key Output |
|------|---------|-----------|------------|
| 1 | `getSketch` | `.h5ad` / `.feather` | Downsampled cells |
| 2 | `getGRN` | Expression matrix, RBP list | `*_scRBP_gene_GRNs.tsv` |
| 3 | `getMerge_GRN` | Multiple GRN `.tsv` files | Consensus GRN `.tsv` |
| 4 | `getModule` | Consensus GRN `.tsv` | Modules `.tsv` |
| 5 | `getPrune` | Modules, motif databases | Pruned scores (Parquet) |
| 6 | `getRegulon` | Pruned scores | Regulons `.gmt` (symbol + Entrez) |
| 7 | `mergeRegulons` | Multiple `.gmt` files | Merged `.gmt` |
| 8 | `ras` | Expression matrix, `.gmt` | RAS matrix `.csv` |
| 9 | `rgs` | MAGMA `.genes.raw`, `.gmt` | RGS scores `.csv` |
| 10 | `trs` | RAS `.csv`, RGS `.csv` | TRS scores `.csv` |

[View Full Pipeline](/pipeline/){: .btn .btn--primary}
[API Reference](/api/){: .btn .btn--info}

---

## Quick Start

```bash
# Install
pip install scRBP

# Downsample large datasets (recommended for >300,000 cells)
scRBP getSketch --input data.h5ad --output sketch.feather --n_cells 50000

# Infer GRN (run 30 seeds for robustness)
for seed in {1..30}; do
  scRBP getGRN --input sketch.feather --output grn_seed${seed}.tsv --seed $seed
done

# Build regulons and score activity
scRBP getMerge_GRN --input_dir grn_seeds/ --output merged_grn.tsv
scRBP getModule    --input merged_grn.tsv  --output modules/
scRBP getPrune     --input modules/ --output pruned/ \
                   --motif_db motif_db.feather --rankings_db rankings.feather
scRBP getRegulon   --input pruned/ --output regulons.gmt
scRBP ras          --input data.h5ad --regulons regulons.gmt --output ras.csv
```

[Full Quick Start Guide](/quickstart/){: .btn .btn--primary .btn--large}
[Installation Guide](/installation/){: .btn .btn--info .btn--large}

---

## Citation

If you use scRBP in your research, please cite:

> Ma Y. *et al.* Decoding disease-associated RNA-binding protein-mediated regulatory networks through polygenic enrichment across diverse cellular contexts. (2026)

---

## Links

- **GitHub**: [https://github.com/mayunlong89/scRBP](https://github.com/mayunlong89/scRBP)
- **PyPI**: [https://pypi.org/project/scRBP](https://pypi.org/project/scRBP)
- **Issues**: [https://github.com/mayunlong89/scRBP/issues](https://github.com/mayunlong89/scRBP/issues)

---

## License

scRBP is released under the [MIT License](https://github.com/mayunlong89/scRBP/blob/main/LICENSE).
