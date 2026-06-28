# Clinical Genomics Pipeline - Technical Guide

## What is This Pipeline?

This pipeline is an automated system that analyzes DNA sequencing data using Nextflow. It takes raw DNA reads, aligns them to a reference genome, indexes the alignment file, and calls genetic variants using deep learning.

---

## The Three Main Steps

### Step 1: **ALIGN_AND_SORT**

**Technical process:**
- **Tool:** `minimap2` (aligns reads to reference)
- **Input:** 
  - `reads.fastq` - Raw DNA sequences
  - `ref_genome.fa` - Reference genome
  - `ref_genome.fa.fai` - Reference genome index for faster processing
- **Output:** `aligned.bam` - Sorted alignment file
- **Command:** `minimap2 -a ref_genome.fa reads.fastq | samtools sort -o aligned.bam -`
- **Resources:** 8 CPUs, 24 GB memory
- **Container:** `minimap2.sif`

---

### Step 2: **INDEX_BAM**

**Technical process:**
- **Tool:** `samtools index`
- **Input:** `aligned.bam` from Step 1
- **Output:** `aligned.bam.bai` - Index file
- **Command:** `samtools index aligned.bam`
- **Resources:** 4 CPUs, 8 GB memory
- **Container:** `minimap2.sif`

---

### Step 3: **CALL_VARIANTS**

**Technical process:**
- **Tool:** `Clair3` (deep learning-based variant caller for long-read data)
- **Input:**
  - `aligned.bam` + `aligned.bam.bai` (from Steps 1 & 2)
  - `ref_genome.fa` - Reference genome
  - `ref_genome.fa.fai` - Reference genome index
  - Platform: "ont" (Oxford Nanopore Technology)
- **Output:** `output.vcf` - Variant call file
- **Command:** `run_clair3.sh --bam_fn=aligned.bam --ref_fn=ref_genome.fa --output=. --threads=8 --platform=ont`
- **Resources:** 8 CPUs, 24 GB memory
- **Container:** `clair3.sif`

---

## Workflow

```
reads.fastq + ref_genome.fa + ref_genome.fa.fai
    |
    v
ALIGN_AND_SORT → aligned.bam
    |
    v
INDEX_BAM → aligned.bam.bai
    |
    v
CALL_VARIANTS → output.vcf
```

---

## File Locations

**Data Folder** (`D:\clinical_genomics\data\`):
- `reads.fastq` - Raw DNA sequence reads
- `ref_genome.fa` - Reference genome sequence
- `ref_genome.fa.fai` - Reference genome index file

**Containers Folder** (`D:\clinical_genomics\containers\`):
- `minimap2.sif` - Container with minimap2 and samtools
- `minimap2.def` - Singularity definition for minimap2 container
- `clair3.sif` - Container with Clair3
- `clair3.def` - Singularity definition for clair3 container

**Results Folder** (`D:\clinical_genomics\results\`):
- Generated automatically when pipeline runs
- Contains `aligned.bam`, `aligned.bam.bai`, and `output.vcf`

---

## Configuration (`nextflow.config`)

**Parameters:**
- `reads` - Path to input FASTQ file
- `ref_genome` - Path to reference genome FASTA file
- `ref_genome_fai` - Path to reference genome index file
- `outdir` - Output directory for results

**Settings:**
- `errorStrategy = 'retry'` - Retries failed processes once
- `maxRetries = 1` - Maximum 1 retry per process
- `cleanup = false` - Keeps intermediate work files
- `apptainer.enabled = true` - Uses Apptainer/Singularity containers
- `apptainer.autoMounts = true` - Automatically mounts data folders in containers

---

## Process Configuration

| Process | Container | CPUs | Memory |
|---------|-----------|------|--------|
| ALIGN_AND_SORT | minimap2.sif | 8 | 24 GB |
| INDEX_BAM | minimap2.sif | 4 | 8 GB |
| CALL_VARIANTS | clair3.sif | 8 | 24 GB |

---

## Running the Pipeline

**Command:**
```bash
nextflow run main.nf
```

**Execution flow:**
1. ALIGN_AND_SORT runs with input FASTQ and reference genome
2. INDEX_BAM indexes the output BAM file
3. CALL_VARIANTS calls variants using BAM, index, and reference
4. Results saved to `results/` folder

Nextflow runs each process in its assigned container with specified CPU and memory resources.

---

## Output Files

After pipeline completion, results are in `results/`:
- `aligned.bam` - Aligned and sorted reads
- `aligned.bam.bai` - BAM index
- `output.vcf` - Variant call file with detected variants

---

## Main Workflow File (`main.nf`)

The workflow defines three processes that execute sequentially:

1. **ALIGN_AND_SORT:** Uses minimap2 to align FASTQ reads to reference, pipes output to samtools sort
2. **INDEX_BAM:** Creates BAM index file using samtools index
3. **CALL_VARIANTS:** Uses Clair3 to call variants from BAM and index

Data flows between processes using Nextflow channels. Each process has its own container and resource allocation defined in `nextflow.config`.

---

## Key Features

- **Automated workflow:** Three processes execute sequentially
- **Containerized:** Each process runs in Singularity/Apptainer container with all required tools
- **Reproducible:** Same configuration produces identical results
- **Error handling:** Automatic retry on process failure
- **Resource-aware:** Each process has specified CPU and memory allocation
- **Input flexibility:** Paths defined in config file, easy to change data sources
