## If you're using an external HDD (especially exFAT/NTFS type), you must customize your `nextflow.config` file

### This document explains how to modify the pipeline using **profiles** and **overrides** in `nextflow.config`, including setting CPU/RAM per process, changing the `workDir`, and tips for handling common errors (OOM, `No space left on device`).
---

## 1) How Nextflow Reads Configuration
Priority order (highest priority at the bottom):
1. Default `nextflow.config` from the pipeline
2. Additional config file via `-c my.config`
3. Profiles enabled with `-profile`
4. CLI variables/parameters (`--param value`)

> In practice: keep your custom settings in a separate file, e.g. `conf/custom.config`, then call it with `-c conf/custom.config`.
---

## 2) Minimal Custom Config Structure
Create a file: `conf/custom.config`
```groovy
profiles {
  // Profile for running on laptop / single-node VM
  local {
    process {
      executor = 'local'
      withLabel: 'light' { cpus = 2; memory = 4.GB; time = '2h' }
      withLabel: 'heavy' { cpus = 8; memory = 32.GB; time = '24h' }
    }
    workDir = '/path/to/ssd_ext4/work'   // Avoid exFAT/NTFS
    scratch = true
  }

  // Example profile for SLURM cluster
  slurm {
    process {
      executor = 'slurm'
      queue = 'general'
      withLabel: 'light' { cpus = 2; memory = 6.GB; time = '3h' }
      withLabel: 'heavy' { cpus = 16; memory = 64.GB; time = '36h' }
    }
    workDir = '/scratch/$USER/wf/work'
    scratch = true
  }
}

// Commonly overridden pipeline parameters
params {
  outdir = 'results'
  // Example references
  genome_fasta = '/data/ref/GRCh38.fa'
  genome_gtf   = '/data/ref/GRCh38.gtf'
  // Example subset / target gene
  bed_regions  = 'assets/targets.bed'
}
---
## 3) Process Labels, Disk Issues, and OOM Handling
Nextflow allows customizing resources per process and also requires some adjustments to avoid disk errors or out-of-memory (OOM) problems.

### Process Labels & Resource Allocation
You can set **different CPU/RAM/time** for each process using `withLabel:` or `withName:`.

```groovy
process {
  // default for all processes
  cpus = 2
  memory = 4.GB
  time = '2h'

  // override based on label
  withLabel: 'map'      { cpus = 8;  memory = 32.GB; time = '24h' }
  withLabel: 'assemble' { cpus = 12; memory = 48.GB; time = '36h' }
}

process {
  // override based on process name
  withName: 'pipeline:reference_assembly:map_reads' { cpus = 8;  memory = 32.GB; time = '24h' }
  withName: 'pipeline:split_bam'                    { cpus = 4;  memory = 16.GB; time = '8h' }
  withName: 'pipeline:assemble_transcripts'         { cpus = 12; memory = 48.GB; time = '36h' }
}
---

## 4) Avoiding "No space left on device" Errors
This error often appears because:
1. `workDir` is pointed to an exFAT/NTFS disk (not compatible).
2. The disk is full (e.g. `/tmp` is too small).

**Solution**
Set `workDir` in `nextflow.config` or `custom.config`:
```groovy
workDir = '/mnt/nvme_ext4/work'   // gunakan filesystem EXT4/XFS
process {
  scratch = true                  // pakai scratch lokal
  env.TMPDIR = '/mnt/nvme_ext4/tmp'
}
---

## 5) Handling OOM (Out Of Memory)
Error `exit 137` or `Killed` ussually indicates insufficient RAM.

Solution
**1. Increase memory allocation for the specific process:**
```groovy
process {
  withName: 'pipeline:assemble_transcripts' {
    cpus = 16
    memory = 64.GB
    time = '48h'
  }
}
