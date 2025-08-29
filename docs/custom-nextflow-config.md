## If you're using the HDD external (specific for exFAT/NTFS type) you must to customize your netxflow.config file

Dokumen ini menjelaskan cara mengubah pipeline lewat **profile** dan **override** di `nextflow.config`, termasuk set CPU/RAM per proses, ganti `workDir`, dan tips mengatasi error umum (OOM, `No space left on device`).
---

## 1) Cara Nextflow Membaca Konfigurasi
Urutan prioritas (yang paling tinggi di bawah):
1. `nextflow.config` bawaan pipeline
2. File config tambahan via `-c my.config`
3. Profile yang diaktifkan via `-profile`
4. Variabel/parameter CLI (`--param value`)

> Praktiknya: simpan customize di file terpisah, mis. `conf/custom.config`, lalu panggil dengan `-c conf/custom.config`.
---

## 2) Struktur Minimal File Kustom
Buat file: `conf/custom.config`
```groovy
profiles {
  // Profile untuk kerja di laptop/VM single node
  local {
    process {
      executor = 'local'
      withLabel: 'light' { cpus = 2; memory = 4.GB; time = '2h' }
      withLabel: 'heavy' { cpus = 8; memory = 32.GB; time = '24h' }
    }
    workDir = '/path/ke/ssd_ext4/work'      // Hindari exFAT/NTFS
    scratch = true
  }

  // Contoh profile untuk cluster SLURM
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

// Parameter pipeline yang sering diganti
params {
  outdir = 'results'
  // Contoh referensi
  genome_fasta = '/data/ref/GRCh38.fa'
  genome_gtf   = '/data/ref/GRCh38.gtf'
  // Misal subset/target gene
  bed_regions  = 'assets/targets.bed'
}

## 3) Label Proses & Resource
Nextflow bisa kita setting untuk pakai **resource berbeda untuk setiap proses**.  
Gunakan `withLabel:` atau `withName:` untuk override kebutuhan CPU/RAM/time.

### Contoh dengan `label`
```groovy
process {
  // default semua proses
  cpus = 2
  memory = 4.GB
  time = '2h'

  // override berdasarkan label
  withLabel: 'map'       { cpus = 8;  memory = 32.GB; time = '24h' }
  withLabel: 'assemble'  { cpus = 12; memory = 48.GB; time = '36h' }
}

### Contoh dengan 'process name'
```groovy
process {
  withName: 'pipeline:reference_assembly:map_reads' { cpus = 8; memory = 32.GB; time = '24h' }
  withName: 'pipeline:split_bam'                    { cpus = 4; memory = 16.GB; time = '8h' }
  withName: 'pipeline:assemble_transcripts'         { cpus = 12; memory = 48.GB; time = '36h' }
}

## 4. Menghindari Error "No space left on device"
Error ini sering muncul karena:
1. workDir diarahkan ke disk exFAT/NTFS (tidak kompatibel).
2. Disk penuh (mis. /tmp terlalu kecil).

### Solusi
Atur workDir di nextflow.config atau custom.config:
```groovy
workDir = '/mnt/nvme_ext4/work'   // gunakan filesystem EXT4/XFS
process {
  scratch = true                  // pakai scratch lokal
  env.TMPDIR = '/mnt/nvme_ext4/tmp'
}

## 5) 5. Mengatasi OOM (Out Of Memory)
Error exit 137 atau Killed biasanya karena RAM kurang.

Solusi
1. Tambahkan alokasi RAM di process:
```groovy
process {
  withName: 'pipeline:assemble_transcripts' {
    cpus = 16
    memory = 64.GB
    time = '48h'
  }
}
