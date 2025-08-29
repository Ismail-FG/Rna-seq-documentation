## Command-line epi2me-labs with Nextflow
nextflow run epi2me-labs/wf-transcriptomes \
--de_analysis
--direct_rna |
--fastq '/media/v100/LaCie/02_TrainingRNA-seq/de_seq/' \
--minimap2_index_opts -k 15'
--ref_annotation '/media/v100/LaCie/02_TrainingRNA-seq/GCF_000001405.40_GRCh38.p14_genomic.gtf' \
--ref_genome '/media/v100/LaCie/02_TrainingRNA-seq/GCF_000001405.40_GRCh38.p14_genomic.fna' \
--sample_sheet '/media/v100/LaCie/02_TrainingRNA-seq/sample_sheet.csv' \
-profile standard
