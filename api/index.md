---
title: "API Reference"
permalink: /api/
toc: true
toc_sticky: true
---

All scRBP commands follow the pattern:

```bash
scRBP <command> [options]
```

Run `scRBP --help` or `scRBP <command> --help` for full option listings.

---

## Commands Overview

| Command | Description |
|---------|-------------|
| `getSketch` | Stratified cell downsampling via GeoSketch |
| `getGRN` | GRN inference using GRNBoost2 or GENIE3 |
| `getMerge_GRN` | Consensus network merging across seeds |
| `getModule` | Regulon candidate extraction |
| `getPrune` | Motif enrichment filtering via ctxcore |
| `getRegulon` | GMT file generation (symbol + Entrez) |
| `mergeRegulons` | Region-specific GMT consolidation |
| `ras` | Regulon Activity Score (AUCell) |
| `rgs` | MAGMA gene-set GWAS enrichment |
| `trs` | Trait Relevance Score integration |

---

## getSketch

```
scRBP getSketch --input INPUT --output OUTPUT [--n_cells N]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Input `.h5ad` file |
| `--output` | path | required | Output `.h5ad` file |
| `--n_cells` | int | 5000 | Number of cells to sample |

---

## getGRN

```
scRBP getGRN --input INPUT --output OUTPUT [--method METHOD] [--mode MODE] [--seed SEED]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Input `.h5ad` file |
| `--output` | path | required | Output GRN `.tsv` file |
| `--method` | str | `grnboost2` | `grnboost2` or `genie3` |
| `--mode` | str | `sc` | `sc` (single-cell) or `ct` (cell-type) |
| `--seed` | int | 42 | Random seed |

---

## ras

```
scRBP ras --input INPUT --regulons REGULONS --output OUTPUT [--mode MODE]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Input `.h5ad` file |
| `--regulons` | path | required | Regulon GMT file |
| `--output` | path | required | Output RAS `.csv` file |
| `--mode` | str | `sc` | `sc` or `ct` |

---

## rgs

```
scRBP rgs --regulons REGULONS --gwas_dir DIR --output OUTPUT [--magma_path PATH]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--regulons` | path | required | Regulon GMT file |
| `--gwas_dir` | path | required | Directory of GWAS summary stats |
| `--output` | path | required | Output RGS `.csv` file |
| `--magma_path` | path | `magma` | Path to MAGMA binary |

---

## trs

```
scRBP trs --ras RAS --rgs RGS --output OUTPUT [--lambda LAMBDA]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--ras` | path | required | RAS `.csv` from `ras` step |
| `--rgs` | path | required | RGS `.csv` from `rgs` step |
| `--output` | path | required | Output TRS `.csv` file |
| `--lambda` | float | 0.5 | Penalty for RAS–RGS divergence |
