## Creating a new truth set for training DeepPolisher, using only places where GIAB v4.2.1 and T2T v1.0.1 agree

02/22/2024

### 1. Download HG002_T2T_v1.0.1 assemblies

https://s3-us-west-2.amazonaws.com/human-pangenomics/index.html?prefix=T2T/HG002/assemblies/

Location on phoenix cluster:
```
ls /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data

hg002v1.0.1.mat.fasta.gz
hg002v1.0.1.mat.fasta.gz.fa.gz
hg002v1.0.1.pat.fasta.gz
hg002v1.0.1.pat.fasta.gz.fai
```
### 2. Run dipcall on HG002_T2T_v1.0.1 vs GRCh38

`dipcall_inputs.json`:

```json
{
  "runDipcall.dipcall.referenceFai": "/private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.fai",
  "runDipcall.dipcall.referenceFasta": "/private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta",
  "runDipcall.dipcall.assemblyFastaMat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.mat.fasta.gz",
  "runDipcall.dipcall.assemblyFastaPat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.pat.fasta.gz",
  "runDipcall.dipcall.referenceIsHS38": true,
  "dipcall.isMaleSample":true
}
```
Run dipcall

```sh
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38

export SINGULARITY_CACHEDIR=`pwd`/../cache/.singularity/cache
export MINIWDL__SINGULARITY__IMAGE_CACHE=`pwd`/../cache/.cache/miniwdl
export TOIL_SLURM_ARGS="--time=4:00:00 --partition=high_priority"
export TOIL_COORDINATION_DIR=/data/tmp

mkdir -p toil_logs

time toil-wdl-runner \
    --jobStore ./jobstore \
    --stats \
    --clean=never \
    --batchSystem slurm \
    --batchLogsDir ./toil_logs \
    /private/groups/hprc/polishing/hpp_production_workflows/QC/wdl/tasks/dipcall.wdl \
    dipcall_inputs.json \
    --outputDirectory ./dipcall_outfiles \
    --outputFile dipcall_outputs.json \
    --runLocalJobsOnWorkers \
    --retryCount 1 \
    --disableProgress \
    --logDebug \
    2>&1 | tee log.txt
```
### 3. Run happy on the output of step 2 against GIAB v4.2.1 VCF, in high confidence regions

Run hap.py
```
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/happy/happy_out
```

```
#!/bin/bash
#SBATCH --job-name=happy
#SBATCH --mail-type=FAIL,END
#SBATCH --partition=short
#SBATCH --mail-user=mmastora@ucsc.edu
#SBATCH --nodes=1
#SBATCH --mem=256gb
#SBATCH --cpus-per-task=16
#SBATCH --output=%x.%j.log
#SBATCH --time=1:00:00

docker run --rm -u `id -u`:`id -g` \
-v /private/groups/patenlab/mira:/private/groups/patenlab/mira \
jmcdani20/hap.py:v0.3.12 /opt/hap.py/bin/hap.py \
/private/groups/patenlab/mira/data/HG002_GRCh38_1_22_v4.2.1_benchmark.vcf.gz \
/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/dipcall_outfiles/hg002v1.0.1.dipcall.vcf.gz \
-r /private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta \
-f /private/groups/patenlab/mira/data/HG002_GRCh38_1_22_v4.2.1_benchmark_noinconsistent.bed \
-o /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/happy/happy_out \
--pass-only --no-roc --no-json --engine=vcfeval --threads=16
```

Results: [Rows 2-3](https://docs.google.com/spreadsheets/d/1V4aKo-NwwafFELxX0-LcCRxG9hmJ03t3jmxiLfdmEbU/edit#gid=0)
### 4. From hap.py output, subtract all regions with FP/FN from GIAB high confidence bed

```
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/happy

# extract FP/FN from happy vcf
zcat happy_out.vcf.gz | grep "^#" > happy_out.FPFN.vcf
zcat happy_out.vcf.gz | grep -v "^#" | grep :F >> happy_out.FPFN.vcf

# count variants subtracted
zcat happy_out.vcf.gz | grep -v "^#" | wc -l
# 4828845

grep -v "^#" happy_out.FPFN.vcf | wc -l
# 32503 to be subtracted

# subtract FP/FN from GIAB confidence set
bedtools subtract -a /private/groups/patenlab/mira/data/HG002_GRCh38_1_22_v4.2.1_benchmark_noinconsistent.bed -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/happy/happy_out.FPFN.vcf > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_concordant.bed
```

### 4. Also remove regions of low short read mappability in T2T from concordant.bed file

##### Project bed file to GRCh38 and subtract from platinum.bed file

Prepare paf files from dipcall output (T2T against GRCh38)
```
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/dipcall_outfiles
tar -zxvf hg002v1.0.1.dipcall.tar.gz

# remove unmapped paf records
grep "tp:A:P" hg002v1.0.1.dipcall/hg002v1.0.1.hap1.paf > hg002v1.0.1.GRCh38.hap1.AP.paf
grep "tp:A:P" hg002v1.0.1.dipcall/hg002v1.0.1.hap2.paf > hg002v1.0.1.GRCh38.hap2.AP.paf
```
Low mappability bed files shared from Nancy:https://s3-us-west-2.amazonaws.com/human-pangenomics/index.html?prefix=T2T/HG002/assemblies/annotation/mappability/element_500bpinsert_avgmq/
```
# merged regions that have avg MQ >=50
/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.window10k.avgmq.ge50.merge.bed
```

```
# project to pat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'asm2ref' \
--paf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/dipcall_outfiles/hg002v1.0.1.GRCh38.hap1.AP.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.window10k.avgmq.ge50.merge.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projectable.GRCh38.pat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.pat.bed

# project to mat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'asm2ref' \
--paf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/dipcall_outfiles/hg002v1.0.1.GRCh38.hap2.AP.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.window10k.avgmq.ge50.merge.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projectable.GRCh38.mat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.mat.bed

# combine, sort, merge bed files  
cat /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.mat.bed /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.pat.bed | sort -k1,1 -k2,2n - | bedtools merge -i - > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.dip.merged.bed
```

Intersect with GIAB / T2T concordant regions to produce platinum bed
```
bedtools intersect -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_concordant.bed -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.dip.merged.bed | awk '{sum += $3-$2}END{print sum}'
# 2512557760

awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_concordant.bed
# 2542201256

# removing low short read mappability regions removes ~ 2% of sequence from platinum bed

# create platinum bed
bedtools intersect -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_concordant.bed -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/low_mapq_short_read_T2T/hg002v1.0.1.ge50_mapq.projection.GRCh38.dip.merged.bed > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed

sort -k1,1 -k2,2n /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed | bedtools merge -i - > tmp ; mv tmp /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed
```

### 5. Run happy again on the output of step 2 against GIAB v4.2.1 but with  HG002_GRCh38_T2T_platinum.bed. F1-score should be close to 1

```
#!/bin/bash
#SBATCH --job-name=happy
#SBATCH --mail-type=FAIL,END
#SBATCH --partition=short
#SBATCH --mail-user=mmastora@ucsc.edu
#SBATCH --nodes=1
#SBATCH --mem=256gb
#SBATCH --cpus-per-task=16
#SBATCH --output=%x.%j.log
#SBATCH --time=1:00:00

cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/happy_platinum_bed/

docker run --rm -u `id -u`:`id -g` \
-v /private/groups/patenlab/mira:/private/groups/patenlab/mira \
jmcdani20/hap.py:v0.3.12 /opt/hap.py/bin/hap.py \
/private/groups/patenlab/mira/data/HG002_GRCh38_1_22_v4.2.1_benchmark.vcf.gz \
/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/dipcall_outfiles/hg002v1.0.1.dipcall.vcf.gz \
-r /private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta \
-f /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed \
-o /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_GRCh38/happy_platinum_bed/happy_out \
--pass-only --no-roc --no-json --engine=vcfeval --threads=16
```
Results: [Rows 4-5](https://docs.google.com/spreadsheets/d/1V4aKo-NwwafFELxX0-LcCRxG9hmJ03t3jmxiLfdmEbU/edit#gid=0)

### 6. Project HG002_GRCh38_T2T_platinum.bed to hg002_hprc_y2 assembly

Using alignments of GRCh38 to hprc y2 assemblies generated by a previous dipcall run:

```
/private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/dipcall_grch38/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.hap1.AP.grch38.paf

/private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/dipcall_grch38/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.hap2.AP.grch38.paf
```
Project platinum bed to hprc y2 assemblies
```
# project to pat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'ref2asm' \
--paf /private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/dipcall_grch38/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.hap1.AP.grch38.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projectable.y2.pat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projection.y2.pat.bed

# project to mat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'ref2asm' \
--paf /private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/dipcall_grch38/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.hap2.AP.grch38.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projectable.y2.mat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projection.y2.mat.bed

```
Count projectable bases
```
# total bases
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed

2512557760

# total projectable bases to pat
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projectable.y2.pat.bed

2511549690

# total projectable bases to mat
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projectable.y2.mat.bed

2512523433
```

### 7. Run dipcall on HG002_T2T_v1.0.1 vs HG002 hprc y2 assembly

Dipcall T2T Mat against HPRC y2 Mat

`dipcall_inputs.json`:

Made a copy of t2t assembly fasta files, so input could work with dipcall wdl, since it throws an error if assemblyFastaMat and assemblyFastaPat have the same name

```json
{
  "runDipcall.dipcall.referenceFai": "/private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.mat.fa.fai",
  "runDipcall.dipcall.referenceFasta": "/private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.mat.fa",
  "runDipcall.dipcall.assemblyFastaMat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.mat.fasta.gz",
  "runDipcall.dipcall.assemblyFastaPat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.mat.copy.fasta.gz",
  "runDipcall.dipcall.referenceIsHS38": false,
  "dipcall.isMaleSample":true
}
```

Dipcall T2T Pat against HPRC y2 Pat

`dipcall_inputs.json`:

```json
{
  "runDipcall.dipcall.referenceFai": "/private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.pat.fa.fai",
  "runDipcall.dipcall.referenceFasta": "/private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.pat.fa",
  "runDipcall.dipcall.assemblyFastaMat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.pat.fasta.gz",
  "runDipcall.dipcall.assemblyFastaPat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.pat.copy.fasta.gz",
  "runDipcall.dipcall.referenceIsHS38": false,
  "dipcall.isMaleSample":true
}
```

toil locations:
```
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat

cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat
```
toil command:
```
export SINGULARITY_CACHEDIR=`pwd`/../cache/.singularity/cache
export MINIWDL__SINGULARITY__IMAGE_CACHE=`pwd`/../cache/.cache/miniwdl
export TOIL_SLURM_ARGS="--time=4:00:00 --partition=medium"
export TOIL_COORDINATION_DIR=/data/tmp

mkdir -p toil_logs

time toil-wdl-runner \
    --jobStore ./jobstore \
    --stats \
    --clean=never \
    --batchSystem slurm \
    --batchLogsDir ./toil_logs \
    /private/groups/hprc/polishing/hpp_production_workflows/QC/wdl/tasks/dipcall.wdl \
    dipcall_inputs.json \
    --outputDirectory ./dipcall_outfiles \
    --outputFile dipcall_outputs.json \
    --runLocalJobsOnWorkers \
    --retryCount 1 \
    --disableProgress \
    --logDebug \
    2>&1 | tee log.txt
```

### 8. Intersect dipcall outputs (VCFs) with HG002_GRCh38_T2T_platinum_projected.bed

Prepare files
```
# unzip mat pair vcf
gunzip -c /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.pair.vcf.gz > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf

# unzip pat pair vcf
gunzip -c /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.pair.vcf.gz > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf

mkdir -p /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected

```
Intersect
```
grep "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.mat.dipcall.HPRC_y2_mat.platinum.pair.vcf

# mat to mat
bedtools intersect -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projection.y2.mat.bed >> /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.mat.dipcall.HPRC_y2_mat.platinum.pair.vcf

# get header
grep "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.pat.dipcall.HPRC_y2_pat.platinum.pair.vcf

bedtools intersect -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/project_platinum_to_hprc_y2/HG002_GRCh38_T2T_platinum.projection.y2.pat.bed >> /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.pat.dipcall.HPRC_y2_pat.platinum.pair.vcf
```

Count `pair.vcf` variants before intersect
```
grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf | wc -l
# 44548

grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf | wc -l
# 39345
```
Count `platinum.pair.vcf` (after intersecting)
```
grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.pat.dipcall.HPRC_y2_pat.platinum.pair.vcf | wc -l
# 6584

grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.mat.dipcall.HPRC_y2_mat.platinum.pair.vcf | wc -l
# 7292
```

### 9. Apply intersected outputs of step 7 to hg002_hprc_y2 assemblies

Prepare files
```
mkdir -p /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/hg002_hprc_y2_projected_platinum

bgzip /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.pat.dipcall.HPRC_y2_pat.platinum.pair.vcf
tabix -p vcf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.pat.dipcall.HPRC_y2_pat.platinum.pair.vcf.gz

bgzip /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.mat.dipcall.HPRC_y2_mat.platinum.pair.vcf
tabix -p vcf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.mat.dipcall.HPRC_y2_mat.platinum.pair.vcf.gz
```
Apply edits to pat
```
# apply vcf edits to pat assembly
bcftools consensus --sample hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.hap1.bam -f /private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.pat.fa -H 2 /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.pat.dipcall.HPRC_y2_pat.platinum.pair.vcf.gz > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/hg002_hprc_y2_projected_platinum/HG002.y2.pat.platinum.fa
# The site h1tg000001l:60405239 overlaps with another variant, skipping...
# The site h1tg000025l:2040254 overlaps with another variant, skipping...
# Applied 6582 variants

```
Apply edits to mat
```
# apply vcf edits to mat assembly
bcftools consensus --sample hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.hap1.bam -f /private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.mat.fa -H 2 /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_dipcall_platinum_projected/hg002v1.0.1.mat.dipcall.HPRC_y2_mat.platinum.pair.vcf.gz > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/hg002_hprc_y2_projected_platinum/HG002.y2.mat.platinum.fa

# The site h2tg000022l:16474026 overlaps with another variant, skipping...
# Applied 7291 variants
```
### 10. Run dipcall on hg002_hprc_y2_projected_platinum vs GRCh38

Dipcall HPRC y2 platinum against GRCh38

`dipcall_inputs.json`:

```json
{
  "runDipcall.dipcall.referenceFai": "/private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta.fai",
  "runDipcall.dipcall.referenceFasta": "/private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta",
  "runDipcall.dipcall.assemblyFastaMat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/hg002_hprc_y2_projected_platinum/HG002.y2.mat.platinum.fa",
  "runDipcall.dipcall.assemblyFastaPat": "/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/hg002_hprc_y2_projected_platinum/HG002.y2.pat.platinum.fa",
  "runDipcall.dipcall.referenceIsHS38": true,
  "dipcall.isMaleSample":true
}
```
Run dipcall

```sh
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_HPRC_y2_platinum_GRCh38

export SINGULARITY_CACHEDIR=`pwd`/../cache/.singularity/cache
export MINIWDL__SINGULARITY__IMAGE_CACHE=`pwd`/../cache/.cache/miniwdl
export TOIL_SLURM_ARGS="--time=4:00:00 --partition=high_priority"
export TOIL_COORDINATION_DIR=/data/tmp

mkdir -p toil_logs

time toil-wdl-runner \
    --jobStore ./jobstore \
    --stats \
    --clean=never \
    --batchSystem slurm \
    --batchLogsDir ./toil_logs \
    /private/groups/hprc/polishing/hpp_production_workflows/QC/wdl/tasks/dipcall.wdl \
    dipcall_inputs.json \
    --outputDirectory ./dipcall_outfiles \
    --outputFile dipcall_outputs.json \
    --runLocalJobsOnWorkers \
    --retryCount 1 \
    --disableProgress \
    --logDebug \
    2>&1 | tee log.txt
```

### 11. Run happy on the output of step 10 against GIAB v4.2.1 with HG002_GRCh38_T2T_platinum.bed as the bed file.

```
#!/bin/bash
#SBATCH --job-name=happy
#SBATCH --mail-type=FAIL,END
#SBATCH --partition=short
#SBATCH --mail-user=mmastora@ucsc.edu
#SBATCH --nodes=1
#SBATCH --mem=256gb
#SBATCH --cpus-per-task=16
#SBATCH --output=%x.%j.log
#SBATCH --time=1:00:00

docker run --rm -u `id -u`:`id -g` \
-v /private/groups/patenlab/mira:/private/groups/patenlab/mira \
jmcdani20/hap.py:v0.3.12 /opt/hap.py/bin/hap.py \
/private/groups/patenlab/mira/data/HG002_GRCh38_1_22_v4.2.1_benchmark.vcf.gz \
/private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_HPRC_y2_platinum_GRCh38/dipcall_outfiles/HG002.y2.platinum.dipcall.vcf.gz \
-r /private/groups/patenlab/mira/data/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta \
-f /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/HG002_GRCh38_T2T_platinum.bed \
-o /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_HPRC_y2_platinum_GRCh38/happy/happy_out \
--pass-only --no-roc --no-json --engine=vcfeval --threads=16
```
Results: [Rows 6-7](https://docs.google.com/spreadsheets/d/1V4aKo-NwwafFELxX0-LcCRxG9hmJ03t3jmxiLfdmEbU/edit#gid=0)


## Creating another truth set for training DeepPolisher, just using T2T v1.0.1

- subtracting regions of low short read mappability
- subtracting regions identified by Justin Zook as errors


### 1. Dipcall T2T v1.0.1 against HPRC y2 assembly

[See step 7 above](https://github.com/miramastoras/DeepPolisher_project/blob/main/GIAB_T2T_platinum_truthset.md#7-run-dipcall-on-hg002_t2t_v101-vs-hg002-hprc-y2-assembly)

```
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat

cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat
```

### 2. Project T2T v1.0.1 high short read mappability bed to HPRC y2

Prepare paf files
```
# remove unmapped paf records
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles
gunzip -c hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.hap2.paf.gz | grep "tp:A:P" > hg002v1.0.1.dipcall.HPRC_y2_mat.AP.paf

# remove unmapped paf records
cd /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles
gunzip -c hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.hap1.paf.gz | grep "tp:A:P" > hg002v1.0.1.dipcall.HPRC_y2_pat.AP.paf

mkdir -p /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth
```

```
# project to mat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'asm2ref' \
--paf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.dipcall.HPRC_y2_mat.AP.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.window10k.avgmq.ge50.merge.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projectable.y2.mat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projection.y2.mat.bed

# project to pat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'asm2ref' \
--paf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.dipcall.HPRC_y2_pat.AP.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/hg002v1.0.1.window10k.avgmq.ge50.merge.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projectable.y2.pat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projection.y2.pat.bed
```

### 3. Project draft error / mosaic bed file from Justin Zook to HPRC Y2  

```
# project to mat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'asm2ref' \
--paf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.dipcall.HPRC_y2_mat.AP.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/mergedQ100errorexclusions.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/mergedQ100errorexclusions.projectable.y2.mat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/mergedQ100errorexclusions.projection.y2.mat.bed

# project to pat
docker run --rm -u `id -u`:`id -g` \
-v /private/groups:/private/groups \
mobinasri/flagger:latest python3 \
/home/programs/src/project_blocks_multi_thread.py \
--threads 16 --mode 'asm2ref' \
--paf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.dipcall.HPRC_y2_pat.AP.paf \
--blocks /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/data/mergedQ100errorexclusions.bed \
--outputProjectable /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/mergedQ100errorexclusions.projectable.y2.pat.bed \
--outputProjection /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/mergedQ100errorexclusions.projection.y2.pat.bed
```
### 4. Exclude errors from high short read mappability bed to create an "error free" T2T HG002 v1.0.1 bed

```
mkdir -p /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free
```

```
bedtools subtract -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projection.y2.pat.bed -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/mergedQ100errorexclusions.projection.y2.pat.bed > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002_v1.0.1.error_free.y2_projection.pat.bed

bedtools subtract -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projection.y2.mat.bed -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/mergedQ100errorexclusions.projection.y2.mat.bed > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002_v1.0.1.error_free.y2_projection.mat.bed
```

```
# count bases in good mappability bed
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projection.y2.pat.bed
 # 2640498608

# count bases in "error free" bed
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002_v1.0.1.error_free.y2_projection.pat.bed
# 2640063871

# count bases in good mappability bed
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/hg002v1.0.1.avgmq.ge50.projection.y2.mat.bed
 # 2765596752

# count bases in "error free" bed
awk '{sum += $3-$2}END{print sum}' /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002_v1.0.1.error_free.y2_projection.mat.bed
# 2765193422
```
### 5. Intersect dipcall vcfs with "error free" bed file

Intersect
```
grep "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.mat.HPRC_y2.pair.vcf

# mat to mat
bedtools intersect -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002_v1.0.1.error_free.y2_projection.mat.bed >> /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.mat.HPRC_y2.pair.vcf

# get header
grep "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.pat.HPRC_y2.pair.vcf

bedtools intersect -a /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf -b /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002_v1.0.1.error_free.y2_projection.pat.bed >> /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.pat.HPRC_y2.pair.vcf
```

Count `pair.vcf` variants before intersect
```
grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_pat/dipcall_outfiles/hg002v1.0.1.pat.pair.vcf | wc -l
# 44548

grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/dipcall_T2T_HPRC_y2_mat/dipcall_outfiles/hg002v1.0.1.mat.pair.vcf | wc -l
# 39345
```
Count `error_free*.pair.vcf` (after intersecting)
```
grep -v "^#" /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.pat.HPRC_y2.pair.vcf | wc -l
# 19178

grep -v "^#"  /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.mat.HPRC_y2.pair.vcf | wc -l
# 20605
```

### 6. Apply intersected outputs of step 5 with HG002_hprc_y2 assemblies
Prepare files
```

bgzip /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.mat.HPRC_y2.pair.vcf
tabix -p vcf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.mat.HPRC_y2.pair.vcf.gz

bgzip /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.pat.HPRC_y2.pair.vcf
tabix -p vcf /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.pat.HPRC_y2.pair.vcf.gz
```
Apply edits to pat
```
# apply vcf edits to pat assembly
bcftools consensus --sample hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.hap1.bam -f /private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.pat.fa -H 2 /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.pat.HPRC_y2.pair.vcf.gz > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/HG002.y2.pat.T2Tv1.0.1_error_free.fa

# Applied 19178 variants

```
Apply edits to mat
```
# apply vcf edits to mat assembly
bcftools consensus --sample hg002v1.0.1.copy.dipcall/hg002v1.0.1.copy.hap1.bam -f /private/groups/patenlab/mira/hprc_polishing/data/HG002_y2_polishing/assembly/HG002.trio_hifiasm_0.19.5.DC_1.2_40x.mat.fa -H 2 /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/T2T_HG002v1.0.1.error_free.mat.HPRC_y2.pair.vcf.gz > /private/groups/patenlab/mira/hprc_polishing/GIAB_T2T_platinum_truthset/T2T_v1.0.1_good_mapq_excl_errors_mosaic_truth/T2T_HG002v1.0.1_error_free/HG002.y2.mat.T2Tv1.0.1_error_free.fa


# The site h2tg000020l:61022006 overlaps with another variant, skipping...
# Applied 20604 variants
```
