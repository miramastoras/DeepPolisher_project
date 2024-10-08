## Location of scripts used for analysis of DeepPolisher

#### 1. Annotate vcf by homopolymer or dimer context

The following script takes in a vcf and outputs a new vcf with annotations for the homopolymer and dimer context of each variant. Annotations will be written for any homopolymers > length 2 and dimers > length 4 that are overlapping or within 1bp of a variant. A bed file containing all annotations within 200bp of the each variant is output for debugging.

```
Usage: python3 annotate_vcf_by_seq_context.py [-h] -f FASTA -o OUT_DIR -v VCF

Annotate variants in vcf by sequence context (dimer or homopolymer).
Repeat annotations will be stored in the vcf INFO column with the format:
REP_ANN=[HOMOPOLYMER/DIMER]:[repeat pattern]:[repeat length]

Optional arguments:
  -h, --help            show this help message and exit
  -f FASTA, --fasta FASTA
                        input fasta file
  -o OUT_PREFIX, --out_prefix OUT_PREFIX
                        output files prefix
  -v VCF, --vcf VCF     input vcf file

```


#### 2. Python notebook used for optimizing DeepPolisher GQ filters

`Optimizing_GQ_filters_HPRC_int_asm.ipynb`
