---
title: "scRBP"
layout: splash
permalink: /
header:
  overlay_color: "#1a6496"
  overlay_filter: "0.5"
  overlay_image: /assets/images/banner.jpg
  actions:
    - label: "<i class='fas fa-download'></i> Install via pip"
      url: "/installation/"
    - label: "<i class='fab fa-github'></i> GitHub"
      url: "https://github.com/mayunlong89/scRBP"
excerpt: >
  **Single-Cell RNA-Binding Protein Regulon Inference** <br>
  A systematic, scalable and integrative framework to infer RBP-mediated
  gene and isoform regulatory networks from single-cell transcriptomes.

intro:
  - excerpt: >
      scRBP combines GRN inference, motif enrichment, GWAS integration, and regulon activity scoring
      into a unified 10-step command-line pipeline for single-cell RBP analysis.

feature_row:
  - image_path: /assets/images/feature-grn.png
    alt: "GRN Inference"
    title: "RBP Regulon Inference"
    excerpt: >
      Construct high-confidence RBP–gene and RBP–isoform regulatory networks
      using GRNBoost2/GENIE3 with motif-binding evidence pruning via ctxcore.
    url: "/pipeline/"
    btn_label: "Learn More"
    btn_class: "btn--primary"
  - image_path: /assets/images/feature-score.png
    alt: "Activity Scoring"
    title: "Regulon Activity Scoring"
    excerpt: >
      Quantify RBP regulon activity at single-cell and cell-type resolution
      using the AUCell algorithm (RAS), enabling cross-condition comparisons.
    url: "/pipeline/#step-8-ras"
    btn_label: "Learn More"
    btn_class: "btn--primary"
  - image_path: /assets/images/feature-gwas.png
    alt: "GWAS Integration"
    title: "GWAS Disease Enrichment"
    excerpt: >
      Link RBP regulons to human disease traits via MAGMA-based GWAS enrichment (RGS),
      then integrate with RAS to compute unified Trait Relevance Scores (TRS).
    url: "/pipeline/#step-9-rgs"
    btn_label: "Learn More"
    btn_class: "btn--primary"

feature_row2:
  - image_path: /assets/images/pipeline-overview.png
    alt: "Pipeline Overview"
    title: "10-Step Integrated Pipeline"
    excerpt: |
      From raw single-cell expression data to disease-relevant RBP rankings:

      1. **getSketch** — Stratified cell downsampling
      2. **getGRN** — Network inference (GRNBoost2/GENIE3)
      3. **getMerge_GRN** — Consensus network merging
      4. **getModule** — Regulon candidate extraction
      5. **getPrune** — Motif enrichment filtering
      6. **getRegulon** — GMT file generation
      7. **mergeRegulons** — Region-specific consolidation
      8. **ras** — Regulon Activity Score computation
      9. **rgs** — MAGMA GWAS enrichment analysis
      10. **trs** — Trait Relevance Score integration
    url: "/pipeline/"
    btn_label: "View Full Pipeline"
    btn_class: "btn--primary"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

{% include feature_row id="feature_row2" type="left" %}

## Quick Start

```bash
# Install scRBP
pip install scRBP

# Step 1: Downsample cells
scRBP getSketch --input data.h5ad --output sketch.h5ad --n_cells 50000

# Step 2: Infer GRN
scRBP getGRN --input sketch.h5ad --output grn.tsv --method grnboost2

# Step 3: Compute Regulon Activity Score
scRBP ras --input data.h5ad --regulons regulons.gmt --output ras.csv

```

[Full Pipeline Guide](/pipeline/){: .btn .btn--primary .btn--large}
[Installation Guide](/installation/){: .btn .btn--info .btn--large}

---

## Citation

If you use scRBP in your research, please cite:

> Ma Y. *et al.* Decoding disease-associated RNA-binding protein-mediated regulatory networks through polygenic enrichment across diverse cellular contexts. (2026)
> GitHub: [https://github.com/mayunlong89/scRBP](https://github.com/mayunlong89/scRBP)

---

## License

scRBP is released under the [MIT License](https://github.com/mayunlong89/scRBP/blob/main/LICENSE).
