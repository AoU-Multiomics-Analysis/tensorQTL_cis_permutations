# tensorQTL cis Permutations

This repository contains WDL workflows for running **cis-QTL mapping with permutations** using [tensorQTL](https://github.com/broadinstitute/tensorqtl). Two workflows are provided:

| Workflow | WDL file | Use case |
|---|---|---|
| `tensorqtl_cis_permutations_workflow` | `workflows/tensorqtl_cis_permutations.wdl` | Standard cis-eQTL mapping (expression QTLs) |
| `tensorqtl_cis_permutations_sqtl_workflow` | `workflows/tensorqtl_cis_permutations_sQTL.wdl` | cis-sQTL mapping (splicing QTLs) – requires a phenotype groups file |

The workflows are registered on [Dockstore](https://dockstore.org/) and intended to be executed on a platform that supports WDL (e.g. Terra / Cromwell). Each task runs tensorQTL in `cis` permutation mode and produces a genome-wide cis-QTL summary table.

---

## Workflows

### `tensorqtl_cis_permutations.wdl` – eQTL workflow

Runs tensorQTL cis permutation testing for expression QTLs. The `phenotype_groups` file is optional and may be omitted when all phenotypes are independent.

### `tensorqtl_cis_permutations_sQTL.wdl` – sQTL workflow

Runs tensorQTL cis permutation testing for splicing QTLs. A `phenotype_groups` file is **required** because sQTL analyses group multiple intron clusters (or splice junctions) by gene.

---

## Inputs

### Required inputs

| Input | Type | Description |
|---|---|---|
| `plink_pgen` | `File` | PLINK2 `.pgen` file containing genotype data |
| `plink_pvar` | `File` | PLINK2 `.pvar` file containing variant information |
| `plink_psam` | `File` | PLINK2 `.psam` file containing sample information |
| `phenotype_bed` | `File` | bgzipped and tabix-indexed BED file of phenotype values (samples as columns, one phenotype per row) |
| `covariates` | `File` | Tab-separated covariates file (samples as columns, one covariate per row; first column is the covariate name) |
| `prefix` | `String` | Output file prefix (e.g. `my_cohort.chr1`) |
| `memory` | `Int` | Memory to allocate in GB |
| `disk_space` | `Int` | Disk space to allocate in GB |
| `num_threads` | `Int` | Number of CPU threads |
| `num_gpus` | `Int` | Number of GPUs (nvidia-tesla-p100) |
| `num_preempt` | `Int` | Number of preemptible retries |

### Optional inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `phenotype_groups` | `File` | – | Two-column file grouping phenotypes by gene; **required for sQTL**, optional for eQTL (needed when multiple phenotypes share a gene) |
| `fdr` | `Float` | `0.05` | FDR threshold for significant associations |
| `qvalue_lambda` | `Float` | – | Lambda parameter for q-value calculation |
| `pval_threshold` | `Float` | – | p-value threshold to filter nominal results |
| `seed` | `Int` | – | Random seed for reproducibility |
| `flags` | `String` | – | Additional flags to pass directly to tensorQTL |

---

## Outputs

| Output | Description |
|---|---|
| `cis_qtl` | `{prefix}.cis_qtl.txt.gz` – Compressed table of cis-QTL results (one row per phenotype, top variant and permutation-derived p-values) |
| `log` | `{prefix}.tensorQTL.cis.log` – tensorQTL run log |

---

## Data preparation

### Genotype data (PLINK2 format)

tensorQTL expects genotypes in **PLINK2 binary format** (`.pgen` / `.pvar` / `.psam`). To convert from VCF:

```bash
plink2 --vcf input.vcf.gz \
       --make-pgen \
       --out genotypes
```

Recommended pre-processing steps:
- Filter to samples present in your phenotype and covariate files.
- Remove variants with missing genotype rate > 5% and MAF < 0.01.
- Split multi-allelic sites and left-align indels.
- Optionally split by chromosome for parallelism.

### Phenotype file (BED format)

The phenotype file must be a **bgzipped, tabix-indexed BED file** where:
- The first four columns are `#chr`, `start`, `end`, `phenotype_id`.
- Subsequent columns are sample IDs matching those in the PLINK `.psam` file.
- Rows are phenotypes (genes for eQTL; splice junctions / intron clusters for sQTL).
- Values should be normalized (e.g. inverse-normal transformed).

Create and index the file:

```bash
bgzip phenotypes.bed
tabix -p bed phenotypes.bed.gz
```

### Covariates file

A tab-separated text file with:
- First column: covariate name.
- Remaining columns: one column per sample (in the same order as the phenotype BED file).
- Typical covariates include genotype PCs, expression/splicing PCs, and known technical factors.

Example:
```
ID    SAMPLE1  SAMPLE2  ...
PC1   0.021    -0.013   ...
PC2   0.004     0.031   ...
```

### Phenotype groups file (sQTL / optional eQTL)

A two-column, tab-separated file (no header) mapping each phenotype ID to its group (gene):

```
ENSG00000000003:1:100  ENSG00000000003
ENSG00000000003:2:200  ENSG00000000003
ENSG00000000005:1:300  ENSG00000000005
```

This file ensures that multiple phenotypes belonging to the same gene share a single permutation test.

---

## Scripts

The `scripts/` directory is reserved for helper scripts used to prepare or post-process data for these workflows. Currently no scripts are included; contributions are welcome.

---

## Runtime environment

Tasks run inside a Docker container:

```
gcr.io/broad-cga-francois-gtex/tensorqtl:latest
```

GPU acceleration is enabled by default (`nvidia-tesla-p100`). Workflows are designed to run on Google Cloud Platform (zone `us-central1-c`) via Terra / Cromwell, but any WDL-compatible runner with GPU support can be used.

A GitHub Actions workflow (`.github/workflows/docker-image.yml`) is included for building and pushing a custom Docker image to `ghcr.io` if needed.

---

## Dockstore registration

Both WDL workflows are registered on Dockstore via `.dockstore.yml`:

| Name | Primary descriptor |
|---|---|
| `tensorQTL_permutation` | `workflows/tensorqtl_cis_permutations.wdl` |
| `tensorQTL_permutation_sQTL` | `workflows/tensorqtl_cis_permutations_sQTL.wdl` |

