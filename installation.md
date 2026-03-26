---
title: "Installation"
permalink: /installation/
toc: true
toc_sticky: true
---

## Requirements

| Requirement | Version |
|-------------|---------|
| Python | 3.9, 3.10, or 3.11 |
| pip | latest recommended |
| MAGMA (optional) | for GWAS enrichment (`rgs` step) |

> **Note:** Python 3.12+ is currently not supported due to upstream dependency constraints.

---

## Install via PyPI (Recommended)

```bash
pip install scRBP
```

Verify installation:

```bash
scRBP --help
```

---

## Install from Source

```bash
git clone https://github.com/mayunlong89/scRBP.git
cd scRBP
pip install -e .
```

---

## Install in a Conda Environment (Recommended)

```bash
# Create a dedicated environment
conda create -n scrbp python=3.10
conda activate scrbp

# Install scRBP
pip install scRBP
```

---

## Optional: MAGMA for GWAS Enrichment

The `rgs` step requires the [MAGMA binary](https://ctg.cncr.nl/software/magma) for gene-set GWAS enrichment analysis.

1. Download MAGMA from the [official website](https://ctg.cncr.nl/software/magma)
2. Add MAGMA to your system `PATH`:

```bash
export PATH="/path/to/magma:$PATH"
```

3. Verify:

```bash
magma --version
```

---

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `numpy`, `pandas`, `scipy` | Numerical computation |
| `anndata`, `scanpy` | Single-cell data I/O |
| `arboreto` | GRNBoost2 / GENIE3 inference |
| `pyscenic`, `ctxcore` | Motif enrichment pruning |
| `geosketch` | Stratified cell downsampling |
| `polars`, `pyarrow` | High-performance data processing |

---

## Supported File Formats

| Format | Extension | Used For |
|--------|-----------|---------|
| AnnData | `.h5ad` | Expression matrix (recommended) |
| Feather | `.feather` | Fast tabular exchange |
| Loom | `.loom` | Single-cell data |
| CSV | `.csv` | Expression or output scores |
| GMT | `.gmt` | Regulon gene sets |
| TSV | `.tsv` | Network edge lists |
