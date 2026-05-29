# Microbial DNA/RNAseq using `megahit`, `metaphlan4`, `kraken` and `humann3` pipeline

## Overview of the workflow

1. Create a samplesheet file to execute the pipeline which should be a csv file following the format below:

| sample  | fq1                 | fq2                 |
| ------- | ------------------- | ------------------- |
| sampleA | sampleA_R1.fastq.gz | sampleA_R2.fastq.gz |

2. Run QC tools
   1. `fastqc` on each sample - raw fastq files
   2. `fastp` to trim adapter sequences and low quality reads
      1. below options used for `fastp`
         - `--qualified_quality_phred 20`
         - `--adapter_fasta $adapter`
   3. `fastq_screen` on R1 fastq files to detect possible contaminants.
3. `multiqc` for summarizing the output files of the qc tools
4. Removal of host genome reads (set `params.host_removal` in `run.config`)
   - **DNA samples** (`params.host_removal = "DNA"`): align with `bowtie2`, extract both-unmapped reads
     1. `bowtie2 -x [index_file] -1 R1 -2 R2 | samtools sort -o bam_file`
     2. `samtools view -b -f 12 -F 256 bam_file > both_unmapped.bam`
     3. `samtools fastq both_unmapped.bam -1 sample_host_remove.R1.fastq.gz -2 sample_host_remove.R2.fastq.gz`
     4. [reference](https://www.metagenomics.wiki/tools/short-read/remove-host-sequences)
   - **RNA samples** (`params.host_removal = "RNA"`): align with `STAR`, extract unmapped reads directly as FASTQ
     1. `STAR --outSAMtype None --outReadsUnmapped Fastx ...`
     2. Unmapped reads are gzipped and used as `sample_host_remove.R[12].fastq.gz`
5. Obtain host genome filtered reads (`host_remove.R[12].fastq.gz`) / unfiltered reads (fastp output)
6. Genome assembly using `megahit` on host-filtered reads with `--k-min 27 --k-max 47` option.
7. Microbial classification using `metaphlan4` on host-filtered reads with `-t rel_ab_w_read_stats` option.
8. Concatenate host-filtered reads for `humann3`
   - i.e. `cat sample1_R1.fastq.gz sample1_R2.fastq.gz > sample1.concat.fastq.gz`
   - see [reference](https://forum.biobakery.org/t/humann3-paired-end-reads/862)
9. Run `humann3` on concatenated fastq files


## How to execute the pipeline

Adjust the configuration files such as `metagenomics_conf/run.config` and `cluster.config`. After that:
```
sbatch run_metagenomics.slurm
```

## Dependency

| Category | Tool |
| -------- | ---- |
| Workflow / cluster | `Nextflow`, `SLURM`, `Singularity` |
| QC | `FastQC`, `fastp`, `FastQ Screen`, `MultiQC` |
| Host removal | `bowtie2` (DNA), `STAR` (RNA), `samtools` |
| Assembly | `MEGAHIT` |
| Classification | `MetaPhlAn4`, `Kraken2`, `kraken-biom` |
| Functional profiling | `HUMAnN3` |