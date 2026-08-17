# Notes

## Alternative host removal tools

Introducing BBSplit: Read Binning Tool for Metagenomes and Contaminated Libraries
https://www.seqanswers.com/forum/bioinformatics/bioinformatics-aa/35881-introducing-bbsplit-read-binning-tool-for-metagenomes-and-contaminated-libraries?t=41288

Introducing RemoveHuman: Human Contaminant Removal
https://www.seqanswers.com/forum/bioinformatics/bioinformatics-aa/37175-introducing-removehuman-human-contaminant-removal

## Read counting channels in metagenomics.nf (lines 513–516)

These lines collect read counts at three pipeline stages per sample.

```nextflow
ch_raw_counts     = ch_reads.map { meta, reads -> [meta.sample_name, reads, "raw"] }
ch_fastp_counts   = fastp.out.trim_reads.map { sn, reads, se -> [sn, reads, "post_fastp"] }
ch_hostrem_counts = ch_host_removed.map { sn, reads, se -> [sn, reads, "host_removed"] }
count_reads( ch_raw_counts.mix(ch_fastp_counts, ch_hostrem_counts) )
```

**Why `.map` is needed on each channel:**

Each upstream channel has a different shape:

| Channel | Original shape | After `.map` |
|---|---|---|
| `ch_reads` | `[meta_map, reads]` | `[sample_name, reads, "raw"]` |
| `fastp.out.trim_reads` | `[sample_name, reads, is_SE]` | `[sample_name, reads, "post_fastp"]` |
| `ch_host_removed` | `[sample_name, reads, is_SE]` | `[sample_name, reads, "host_removed"]` |

- `ch_reads` carries a full `meta` map (`[sample_name:, single_end:]`), so `.map` extracts just `meta.sample_name` and adds the stage label.
- The other two already have `sample_name` as a plain string, so `.map` just replaces the third element (`is_SE` boolean) with the stage label string.

**Why `.mix()`:**

`.mix()` merges the three channels into one stream. All items now share the same shape `[sample_name, reads, stage]`, which matches the `count_reads` process input. The process runs once per item — so 3 times per sample (one per stage), producing files like:
- `SampleA.raw.count.csv`
- `SampleA.post_fastp.count.csv`
- `SampleA.host_removed.count.csv`

## References

Methods in Microbiomics
https://methods-in-microbiomics.readthedocs.io/en/latest/preprocessing/preprocessing.html
