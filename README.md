# Short-read assembly pipeline

Snakemake pipeline to trim, align and assemble transcripts from short-read Illumina RNA-seq data

## Pipeline Overview

1. Trim adapters and low quality bases with cutadapt
2. Assess read quality with FastQC and aggregate QC with MultiQC
3. Build a STAR genome index from the reference genome and GENCODE annotation
4. Align trimmed reads to the reference genome with STAR using multi-sample 2-pass mapping
5. Coordinate sort and index the alignments with samtools
6. Assemble transcripts from the sorted BAM files with StringTie

## Quick Start

### Initialize
```
bash [path to repo]/assemble_short_read/run_assemble_short_read --runmode=init --workdir=[directory] --samplesheet=[sample sheet]
```

### Dry Run
Perform a dry run to validate the pipeline:
```
bash [path to repo]/assemble_short_read/run_assemble_short_read --runmode=dryrun --workdir=[directory]
```

### Execute the Pipeline
Run the full pipeline (submit to Biowulf slurm):
```
bash [path to repo]/assemble_short_read/run_assemble_short_read --runmode=run --workdir=[directory]
```

## Configuration Files

- **config.yaml**: Main pipeline configuration specifying input/output directories and sample metadata
- **cluster.yaml**: SLURM resource specifications per rule
- **tools.yaml**: Software module specifications

## Input Requirements

- `sample_sheet` (CSV): Table with columns `Sample`, `R1`, `R2`, where the FASTQ paths are relative to `rawdir`
- Paired-end FASTQ files
- Reference genome FASTA file
- Reference annotation GTF file (GENCODE v41)
- `read_length` in config.yaml, used to set STAR `--sjdbOverhang`

## Output

Pipeline generates results in the `results/` directory:
- `trimmed/{sample}_R[12].trimmed.fastq.gz`: Adapter-trimmed reads
- `star_index/`: STAR genome index built from the reference genome and GTF
- `star_pass1/uniq.filtered.SJ.out.tab`: Splice junctions detected across all samples in the first pass
- `bam/{sample}/{sample}.sorted.bam`: Coordinate-sorted, indexed alignments
- `stringtie/{sample}/{sample}.gtf`: Per-sample transcript assembly
- `stringtie/{sample}/{sample}.gene_abundance.tsv`: Per-sample gene abundance estimates
- `qc/multiqc_report.html`: Aggregated QC report
