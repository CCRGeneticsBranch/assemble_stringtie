# Changelog

## 1.0.0 (unreleased)
- Moved from the ccrgb_96 repository (`scripts/illumina/assemble_short_read`) to a standalone repository; wrapper renamed to `run_assemble_stringtie`.
- Launch script replaced by the shared biowulf-snakemake-launcher (`bin/launcher.sh`); run modes: init, dryrun, unlock, run.
- Standardized repository layout (VERSION, CHANGELOG.md, .gitignore, config/samples.tsv template).
- `rawdir` in config.yaml is now a placeholder; the cluster config is copied to `WORKDIR/config/cluster.yaml`.
- Pipeline version printed at start and written to results/pipeline_version.txt.
