# StringTie Assembly Pipeline

Snakemake pipeline to assemble transcriptomes from Illumina RNA-seq data. The pipeline trims reads, aligns them to the genome with multi-sample STAR 2-pass mapping, infers library strandedness, assembles transcripts per sample with StringTie guided by the GENCODE annotation, and builds per-sample transcript FASTA files and BLAST databases.

## Pipeline steps

| Step | Rule | Tool | Description |
|---|---|---|---|
| 1 | `trim` | cutadapt | Trim adapters, Ns, and low-quality bases (parameters from [RENEE](https://github.com/CCBR/RENEE)) |
| 2 | `fastqc` | FastQC | QC of the trimmed reads |
| 3 | `star_index` | STAR | Build a genome index with the reference annotation (`--sjdbOverhang` = `read_length` − 1) |
| 4 | `star1p` | STAR | First-pass alignment of each sample to collect splice junctions |
| 5 | `sjdb` | `awk` | Pool junctions across samples; keep canonical junctions with ≥3 uniquely mapped reads on `chr*` contigs (excluding chrM) |
| 6 | `star2p` | STAR | Second-pass alignment with the pooled junctions; per-gene read counts |
| 7 | `sort_index` | samtools | Sort, index, and `flagstat` the alignments |
| 8 | `gtf_to_bed12`, `strandedness` | UCSC tools, RSeQC `infer_experiment.py` | Infer library strandedness; sets StringTie `--fr`/`--rf` when >80% of reads agree, otherwise unstranded |
| 9 | `stringtie` | StringTie | Reference-guided transcript assembly (minimum length 200 bp) and gene abundance per sample |
| 10 | `gtf_to_fasta` | gffread | Transcript sequences from each assembly |
| 11 | `fasta_to_blastdb` | BLAST+ `makeblastdb` | Nucleotide BLAST database per sample |
| 12 | `multiqc` | MultiQC | Combined cutadapt, FastQC, STAR, samtools, and RSeQC report |

## Requirements

- Runs on NIH [Biowulf](https://hpc.nih.gov/) (SLURM and environment modules).
- Must have access to dependencies listed in [config/config.yaml](config/config.yaml).

## Installation

```bash
git clone https://github.com/CCRGeneticsBranch/assemble_stringtie.git
```

## Quick start

The pipeline is designed to be run in a separate directory (/path/to/my_run).

```bash
# 1. Create a work folder with template config files
bash assemble_stringtie/run_assemble_stringtie --runmode=init --workdir=/path/to/my_run [--samplesheet=/path/to/samples.tsv]

# 2. Edit /path/to/my_run/config/config.yaml (set rawdir) and config/samples.tsv (see Inputs)

# 3. Dry run (writes /path/to/my_run/dryrun.log)
bash assemble_stringtie/run_assemble_stringtie --runmode=dryrun --workdir=/path/to/my_run

# 4. Submit to SLURM
bash assemble_stringtie/run_assemble_stringtie --runmode=run --workdir=/path/to/my_run

# Unlock the work folder after an interrupted run
bash assemble_stringtie/run_assemble_stringtie --runmode=unlock --workdir=/path/to/my_run
```

`-h` shows all options; `-v` prints the pipeline and launcher versions. Per-rule SLURM resources are in `WORKDIR/config/cluster.yaml` (copied from [resources/cluster.yaml](resources/cluster.yaml) at `init`).

## Inputs

### Sample sheet (`config/samples.tsv`)

Tab-separated, one row per sample.

| Column | Description | Example |
|---|---|---|
| `Sample` | Sample name | `sample1` |
| `R1` | Read 1 FASTQ, relative to `rawdir` | `sample1_R1.fastq.gz` |
| `R2` | Read 2 FASTQ, relative to `rawdir` | `sample1_R2.fastq.gz` |

### Configuration (`config/config.yaml`)

| Key | Default | Description (source) | Edit? |
|---|---|---|---|
| `rawdir` | `/path/to/fastq/` | Folder containing the FASTQ files | Yes |
| `reference_fasta` | `/data/khanlab3/gb_omics/resources/genomes/hg38/hg38_main.fa` | Main-chromosome hg38 FASTA ([UCSC hg38](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/)) | No |
| `reference_gtf` | `/data/khanlab3/gb_omics/resources/gencode/gencode.v41.annotation.gtf` | GENCODE v41 annotation ([GENCODE human releases](https://www.gencodegenes.org/human/releases.html)) | As needed |
| `read_length` | `101` | Read length, used to set STAR `--sjdbOverhang` | As needed |
| `workdir`, `sample_sheet`, `tools`, `cluster` | `WORKDIR/...`, `PIPELINE_HOME/...` | Work folder and launcher/repository paths (filled by `init`) | No |

## Outputs

All paths are relative to the work folder.

| Path | Description |
|---|---|
| `results/star_index/` | STAR genome index built from the reference genome and GTF |
| `results/star_pass1/uniq.filtered.SJ.out.tab` | Splice junctions pooled across samples from the first pass |
| `results/bam/<sample>/<sample>.sorted.bam` | Coordinate-sorted, indexed alignments |
| `results/bam/<sample>/<sample>.ReadsPerGene.out.tab`, `.Log.final.out` | STAR gene counts and alignment summary |
| `results/stringtie/<sample>/<sample>.gtf` | Per-sample transcript assembly |
| `results/stringtie/<sample>/<sample>.gene_abundance.tsv` | Per-sample gene abundance estimates |
| `results/stringtie/<sample>/<sample>.fa` | Per-sample transcript sequences |
| `results/blastdb/<sample>/` | Per-sample nucleotide BLAST database |
| `results/qc/strandedness/<sample>.infer_experiment.txt` | RSeQC strandedness |
| `results/qc/multiqc_report.html` | Combined QC report |
| `results/pipeline_version.txt` | Pipeline version used for the run |
| `runinfo.yaml`, `snakemake.log`, `logs/` | Run information, Snakemake log, and per-rule SLURM logs |

Trimmed FASTQ files and unsorted BAM files are temporary and removed once no longer needed.

## Tools and references

Modules are listed in [resources/tools.yaml](resources/tools.yaml).

| Tool | Module / source | Used in | Reference |
|---|---|---|---|
| Snakemake | `snakemake/7.19.1` | workflow | Mölder et al. 2021, *F1000Research* [doi:10.12688/f1000research.29032.2](https://doi.org/10.12688/f1000research.29032.2) |
| cutadapt | `cutadapt` | `trim` | Martin 2011, *EMBnet.journal* [doi:10.14806/ej.17.1.200](https://doi.org/10.14806/ej.17.1.200) |
| FastQC | `fastqc` | `fastqc` | [Babraham Bioinformatics FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) |
| STAR | `STAR/2.7.11b` | `star_index`, `star1p`, `star2p` | Dobin et al. 2013, *Bioinformatics* [doi:10.1093/bioinformatics/bts635](https://doi.org/10.1093/bioinformatics/bts635) |
| SAMtools | `samtools` | `sort_index` | Danecek et al. 2021, *GigaScience* [doi:10.1093/gigascience/giab008](https://doi.org/10.1093/gigascience/giab008) |
| UCSC tools | `ucsc` | `gtf_to_bed12` | [UCSC Genome Browser utilities](https://hgdownload.soe.ucsc.edu/admin/exe/) |
| RSeQC | `rseqc/5.0.4` | `strandedness` | Wang et al. 2012, *Bioinformatics* [doi:10.1093/bioinformatics/bts356](https://doi.org/10.1093/bioinformatics/bts356) |
| StringTie | `stringtie` | `stringtie` | Pertea et al. 2015, *Nat Biotechnol* [doi:10.1038/nbt.3122](https://doi.org/10.1038/nbt.3122) |
| gffread | `gffread/0.12.7` | `gtf_to_fasta` | Pertea & Pertea 2020, *F1000Research* [doi:10.12688/f1000research.23297.2](https://doi.org/10.12688/f1000research.23297.2) |
| BLAST+ | `blast` | `fasta_to_blastdb` | Camacho et al. 2009, *BMC Bioinformatics* [doi:10.1186/1471-2105-10-421](https://doi.org/10.1186/1471-2105-10-421) |
| MultiQC | `multiqc` | `multiqc` | Ewels et al. 2016, *Bioinformatics* [doi:10.1093/bioinformatics/btw354](https://doi.org/10.1093/bioinformatics/btw354) |

## License

Original pipeline code and documentation are licensed under MIT; see [LICENSE](LICENSE). Third-party software and reference data retain their own terms. The MIT license does not grant redistribution rights for external or bundled third-party resources.

## Versioning

The pipeline version is in [VERSION](VERSION) and changes are listed in [CHANGELOG.md](CHANGELOG.md). Each run records the pipeline version, git commit, and launcher version in `runinfo.yaml` and `results/pipeline_version.txt`. The launch script is a copy of [biowulf-snakemake-launcher](https://github.com/CCRGeneticsBranch/biowulf-snakemake-launcher) (`bin/launcher.sh`).

## Contact

Erica Pehrsson, Genetics Branch, Center for Cancer Research, National Cancer Institute. Please report problems through GitHub issues.
