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
| `getSketch` | Stratified cell downsampling via GeoSketch (Optional) |
| `getGRN` | GRN inference using GRNBoost2 or GENIE3 |
| `getMerge_GRN` | Consensus network merging across seeds |
| `getModule` | Regulon candidate extraction |
| `getPrune` | Motif enrichment filtering via ctxcore |
| `getRegulon` | GMT file generation (symbol + Entrez) |
| `mergeRegulons` | Region-specific GMT consolidation |
| `ras` | Regulon Activity Score (RAS) |
| `rgs` | Regulon-level Genetic association Score (RGS) |
| `trs` | Trait Relevance Score by integrating RAS and RGS |

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

## getMerge_GRN

```
scRBP getMerge_GRN --input_dir DIR --output OUTPUT [--n_seeds N]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input_dir` | path | required | Directory of per-seed GRN `.tsv` files |
| `--output` | path | required | Output merged GRN `.tsv` |
| `--n_seeds` | int | 30 | Number of seed GRNs to merge |

**Input format:** Each `.tsv` file has columns: `TF`, `target`, `importance`
**Output format:** `.tsv` with consensus importance scores across seeds

---

## getModule

```
scRBP getModule --input INPUT --output OUTPUT [--top_targets N]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Merged GRN `.tsv` from `getMerge_GRN` |
| `--output` | path | required | Output directory for module files |
| `--top_targets` | int | 500 | Max target genes per RBP |

**Output:** Per-RBP files listing candidate target genes ranked by importance

---

## getPrune

```
scRBP getPrune --input INPUT --output OUTPUT --motif_db MOTIF_DB --rankings_db RANKINGS_DB
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Module directory from `getModule` |
| `--output` | path | required | Output directory for pruned regulons |
| `--motif_db` | path | required | Motif annotation `.feather` (e.g. `hg38_motifs.feather`) |
| `--rankings_db` | path | required | Genome rankings `.feather` (e.g. `hg38_500bp_rankings.feather`) |

**Resources:** Download motif/rankings databases from [cisTarget resources](https://resources.aertslab.org/cistarget/) or [pySCENIC docs](https://pyscenic.readthedocs.io/en/latest/installation.html#auxiliary-datasets)

---

## getRegulon

```
scRBP getRegulon --input INPUT --output OUTPUT [--format FORMAT]
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input` | path | required | Pruned regulon directory from `getPrune` |
| `--output` | path | required | Output GMT file |
| `--format` | str | `symbol` | Gene ID format: `symbol` or `entrez` |

**Output GMT format:** `RBP_name <TAB> description <TAB> gene1 <TAB> gene2 ...`

---

## mergeRegulons

```
scRBP mergeRegulons --input_dir DIR --output OUTPUT
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--input_dir` | path | required | Directory of region-specific `.gmt` files |
| `--output` | path | required | Merged output GMT file |

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
