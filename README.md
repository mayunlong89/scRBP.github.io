<p align="center">
  <img src="https://raw.githubusercontent.com/mayunlong89/scRBP/main/Examples/scRBP_logo.png" width="250">
</p>



# scRBP

**A scalable framework for inferring RNA-binding protein regulons from single-cell transcriptomic data**

![pypi](https://img.shields.io/badge/pypi-0.1.4.1-green)
![python](https://img.shields.io/badge/python-3.9--3.11-blue)
![license](https://img.shields.io/badge/license-MIT-yellow)

**scRBP (single-cell RNA-binding protein regulon inference)** is a command-line toolkit for reconstructing RNA-binding protein (RBP)-mediated regulatory programs from single-cell transcriptomic data and prioritizing regulons associated with complex traits and disorders. It supports both **gene-level** networks from short-read data and **isoform-level** networks from long-read data.

The framework integrates six major analytical components: (i) a curated compendium of RBPs and clustered RBP-binding motifs assembled from public resources; (ii) motif-guided transcriptome-wide rankings of candidate RBP targets at gene and isoform resolution; (iii) inference of RBP–gene or RBP–isoform association networks from single-cell expression data (with optional mini-metacell densification for sparse inputs); (iv) motif-based refinement of candidate edges to define high-confidence regulons and quantify Regulon Activity Scores (RAS); (v) **parallel common- and rare-variant models** that map common-variant GWAS signals (MAGMA) *and* gene-level rare-variant evidence (TADA / SAIGE-GENE+ / burden / STAAR-O) to regulons to derive Regulon-level Genetic Association Scores (RGS); and (vi) integration of RAS with common- or rare-variant RGS into a Trait-Relevance Score (TRS) for each regulon within each cellular context, with significance evaluated against matched-null regulons via Monte Carlo sampling.

---

## What scRBP Does

RBPs regulate multiple layers of post-transcriptional gene control, including RNA splicing, localization, stability, and translation. scRBP enables you to:

- **Infer** candidate RBP–gene or RBP–isoform association networks from single-cell transcriptomes
- **Refine** candidate RBP–target edges using sequence-motif evidence to define high-confidence regulons
- **Quantify** regulon activity at single-cell or cell-type resolution using AUCell (RAS)
- **Evaluate** both common-variant (MAGMA) **and** rare-variant (TADA / burden / logBF) enrichment per regulon (RGS)
- **Prioritize** trait-relevant RBP regulons via a unified Trait-Relevance Score (TRS) that consumes either RGS flavour

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
          │                          for the GRN branch on sparse inputs)
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
          │                          (3'UTR / 5'UTR / CDS / Intron)
          ▼
[Step 9]  scRBP ras              ── Regulon Activity Score (RAS) per cell / cell type
          │                          (--mode sc | --mode ct)
          ▼
[Step 10] scRBP rgs              ── Common-variant RGS via MAGMA (--mode sc | --mode ct)
          │
          ▼
[Step 11] scRBP rgs_rare         ── Rare-variant RGS via competitive gene-set regression
          │                          (TADA / logBF / burden; --mode sc | --mode ct)
          ▼
[Step 12] scRBP trs              ── Trait-Relevance Score (RAS × RGS integration)
                                     (common- OR rare-variant; --mode sc | --mode ct)
```

> Steps 2 (**getMetacell**) and 11 (**rgs_rare**) are new in v0.1.4.1.
> `getMetacell` is optional and only used for the GRN branch on sparse inputs;
> `rgs_rare` runs in parallel to `rgs` and feeds `scRBP trs` in the same way
> to produce a **rare-variant TRS**.

---

## Command Reference

| Step | Command                            | Key Inputs                                                   | Key Output                                                 |
| ---- | ---------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| 1    | `scRBP getSketch`                  | `.h5ad` / `.feather`                                         | Downsampled cells                                          |
| 2    | `scRBP getMetacell` *(optional)*   | `.h5ad` (with cell-type col) / `.feather`                    | Metacell matrix (gene × metacell) + metacell→cell-type map |
| 3    | `scRBP getGRN`                     | Expression / metacell matrix, RBP list                       | `*_scRBP_gene_GRNs.tsv` or `*_scRBP_isoform_GRNs.tsv`      |
| 4    | `scRBP getMerge_GRN`               | Multiple GRN TSV files (glob)                                | Consensus GRN TSV                                          |
| 5    | `scRBP getModule`                  | Consensus GRN TSV                                            | Modules TSV                                                |
| 6    | `scRBP getPrune`                   | Modules TSV, motif files                                     | Pruned scores (Parquet)                                    |
| 7    | `scRBP getRegulon`                 | Pruned scores                                                | Regulons GMT (symbol + Entrez)                             |
| 8    | `scRBP mergeRegulons`              | Multiple GMT files                                           | Merged GMT                                                 |
| 9    | `scRBP ras` (`--mode sc\|ct`)      | Expression matrix, GMT                                       | RAS matrix (`.csv` / `.loom`) + per-gene expr-stats TSV    |
| 10   | `scRBP rgs` (`--mode sc\|ct`)      | MAGMA `.genes.raw`, GMT, expr-stats                          | Common-variant RGS scores CSV                              |
| 11   | `scRBP rgs_rare` (`--mode sc\|ct`) | Gene-level rare variant summary (TADA / burden / …), GMT, expr-stats | Rare-variant RGS scores CSV (+ REAL/NULL GMT in ct mode)   |
| 12   | `scRBP trs` (`--mode sc\|ct`)      | RAS matrix, RGS CSV (common **or** rare variant)             | TRS scores CSV                                             |

Use `scRBP <command> --help` to see all parameters for any step.

## Links

- **GitHub**: https://github.com/mayunlong89/scRBP
- **Issues**: https://github.com/mayunlong89/scRBP/issues



## Citation

If you use scRBP in your research, please cite:

> Ma Y. *et al.* Single-cell maps of RNA-binding protein networks reveal post-transcriptional architecture across development and disease.* (2026)

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

