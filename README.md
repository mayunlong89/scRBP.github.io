<p align="center">
  <img src="https://raw.githubusercontent.com/mayunlong89/scRBP/main/Examples/scRBP_logo.png" width="250">
</p>



# scRBP

**A scalable framework for inferring RNA-binding protein regulons from single-cell transcriptomic data**

![pypi](https://img.shields.io/badge/pypi-0.1.3-green)
![python](https://img.shields.io/badge/python-3.9--3.11-blue)
![license](https://img.shields.io/badge/license-MIT-yellow)

**scRBP (Single-cell RNA Binding Protein Regulon Inference)** is a command-line toolkit for comprehensive analysis of RNA-binding proteins (RBPs) in single-cell RNA-seq data. scRBP provides a systematic, scalable and integrative framework to infer RBP-mediated gene and isoform regulatory networks ("regulons") from single-cell transcriptomes and prioritize networks underlying complex genetic traits and disorders. scRBP is comprised of six main modules: (i) developing a comprehensive compendium of RBPs and their associated motif clusters from diverse public resources; (ii) systematic, motif-guided transcriptome-wide inference of RBP targets at both gene- and isoform-level resolution; (iii) construction of RBP-gene and/or RBP-isoform co-expression networks from short- or long-read single-cell transcriptomic data, respectively; (iv) defining high-fidelity regulons by integrating RBP-target interactions, and quantifying cell type-specific regulon activity scores (RAS); (v) integrating GWAS results to compute regulon-level genetic association scores (RGS); and (vi) constructing a unified trait-relevance score (TRS) by combining RAS and RGS for each regulon in a given cellular context, with statistical significance assessed using Monte Carlo (MC) sampling.

---

## What scRBP Does

RBPs are key post-transcriptional regulators that control mRNA splicing, stability, and translation. scRBP enables you to:

- **Construct** which RBPs regulate which genes or isoforms in your single-cell data
- **Prune** raw RBP–gene associations using motif-binding evidence to obtain high-confidence regulons
- **Score** each cell or cell type for regulon activity score (RAS) using the AUCell algorithm
- **Link** RBP regulons to human disease through GWAS genetic enrichment (RGS via MAGMA)
- **Integrate** RAS and RGS into a unified Trait Relevance Score (TRS) that ranks disease-relevant RBPs

---

## Pipeline at a Glance

```
Raw single-cell data (.h5ad / .feather)
          │
          ▼
[Step 1]  scRBP getSketch        ── Stratified GeoSketch cell downsampling
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
[Step 9]  scRBP rgs              ── Regulon Gene-Set analysis (RGS)
          │                          (--mode sc | --mode ct)
          ▼
[Step 10] scRBP trs              ── Trait Relevance Score (TRS, integrating RAS with RGS)
                                     (--mode sc | --mode ct)
```

---

## Command Reference

| Step | Command                       | Key Inputs                    | Key Output                                            |
| ---- | ----------------------------- | ----------------------------- | ----------------------------------------------------- |
| 1    | `scRBP getSketch`             | `.h5ad` / `.feather`          | Downsampled cells                                     |
| 2    | `scRBP getGRN`                | Expression matrix, RBP list   | `*_scRBP_gene_GRNs.tsv` or `*_scRBP_isoform_GRNs.tsv` |
| 3    | `scRBP getMerge_GRN`          | Multiple GRN TSV files (glob) | Consensus GRN TSV                                     |
| 4    | `scRBP getModule`             | Consensus GRN TSV             | Modules TSV                                           |
| 5    | `scRBP getPrune`              | Modules TSV, motif files      | Pruned scores (Parquet)                               |
| 6    | `scRBP getRegulon`            | Pruned scores                 | Regulons GMT (symbol + Entrez)                        |
| 7    | `scRBP mergeRegulons`         | Multiple GMT files            | Merged GMT                                            |
| 8    | `scRBP ras` (`--mode sc\|ct`) | Expression matrix, GMT        | single-cell RAS matrix, cell-type RAS matrix          |
| 9    | `scRBP rgs` (`--mode sc\|ct`) | MAGMA `.genes.raw`, GMT       | RGS scores CSV                                        |
| 10   | `scRBP trs` (`--mode sc\|ct`) | RAS CSV, RGS CSV              | TRS scores CSV                                        |

Use `scRBP <command> --help` to see all parameters for any step.

## Links

- **GitHub**: https://github.com/mayunlong89/scRBP
- **Issues**: https://github.com/mayunlong89/scRBP/issues



## Citation

If you use scRBP in your research, please cite:

> Ma Y. *et al.* *Decoding disease-associated RNA-binding protein-mediated regulatory networks through polygenic enrichment across diverse cellular contexts.* (2026)

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

