# 2025_WLBG


# Code and commands for invasion genomics of wavyleaf basketgrass (Oplismenus undulatifolius)

Craig Barrett (1), Carrie Wu (2), Cynthia Huebner (1,3)

1. West Virginia University
2. University of Richmond
3. USDA Forest Service


## 1. Rename fastq files to get rid of _L001, _001, and underscore (_S). Only underscore should be _R1 and _R2.

```bash
mkdir reads   # copy all read files to this directory
cd reads
rename 's/_001.fastq.gz/.fastq.gz/' *.fastq.gz
rename 's/_L001//' *.fastq.gz
rename 's/_S/-S/' *.fastq.gz
```

## 2. Process reads with fastp

```bash
# Loop through paired-end files

cd ..   # run from one level above reads and fastp

for file in reads/*_R1.fastq.gz; do
    base=$(basename $file _R1.fastq.gz)
    R1="reads/${base}_R1.fastq.gz"
    R2="reads/${base}_R2.fastq.gz"
    output_R1="fastp/${base}_R1.fastq.gz"
    output_R2="fastp/${base}_R2.fastq.gz"

    fastp -i $R1 -I $R2 -o $output_R1 -O $output_R2 -w 16 --trim_poly_x --trim_poly_g --cut_right -l 75
done
```

## 3. Make a samples file (delete e.g. "undetermined" if that is in the list)

```bash
cd fastp
ls *R1.fastq.gz | sed 's/_R1.fastq.gz//g' > ../samples.txt
```

### Mig-seq primers for next trimming step in fasta format (migseq_primers.txt)

```bash
>ACT4TG-f
CGCTCTTCCGATCTCTGACTACTACTACTTG
>CTA4TG-f
CGCTCTTCCGATCTCTGCTACTACTACTATG
>TTG4AC-f
CGCTCTTCCGATCTCTGTTGTTGTTGTTGAC
>GTT4CC-f
CGCTCTTCCGATCTCTGGTTGTTGTTGTTCC
>GTT4TC-f
CGCTCTTCCGATCTCTGGTTGTTGTTGTTTC
>GTG4AC-f
CGCTCTTCCGATCTCTGGTGGTGGTGGTGAC
>GT6TC-f
CGCTCTTCCGATCTCTGGTGTGTGTGTGTTC
>TG6AC-f
CGCTCTTCCGATCTCTGTGTGTGTGTGTGAC
>ACT4TG-r
TGCTCTTCCGATCTGACACTACTACTACTTG
>CTA4TG-r
TGCTCTTCCGATCTGACCTACTACTACTATG
>TTG4AC-r
TGCTCTTCCGATCTGACTTGTTGTTGTTGAC
>GTT4CC-r
TGCTCTTCCGATCTGACGTTGTTGTTGTTCC
>GTT4TC-r
TGCTCTTCCGATCTGACGTTGTTGTTGTTTC
>GTG4AC-r
TGCTCTTCCGATCTGACGTGGTGGTGGTGAC
>GT6TC-r
TGCTCTTCCGATCTGACGTGTGTGTGTGTTC
>TG6AC-r
TGCTCTTCCGATCTGACTGTGTGTGTGTGAC
```

## 4. Unzip reads in fastp folder (may take a few minutes)

```bash
pigz -d fastp/*.fastq.gz
```

## 5. Begin assembling with pseudo-reference with ISSR-seq pipeline (may take a a few hours! Sit tight.)

```bash
nohup ISSRseq_AssembleReference.sh -O 13Jan_pseudoref -I fastp -S samples.txt -R  MD-Baltimore-LOCH-11-S24 -T 32 -M 35 -H 0 -P migseq_primers.txt -K 31 -L 100 -X 50 -N O_hirtellus_plastome_KU291473.fasta &



### Output will be:
### 13Jan_pseudoref_2025_01_13_11_19
### Use this name for subsequent steps

```

### Argument interpretation:

```bash
-O [desired prefix of output directory]

-I [path to directory containing sequences]

-S [path to the samples file]

-R [verbatim name of sample to be used for to create the reference assembly]

-T [number of parallel processing threads -- I recommend not exceeding number of virtualized cores]

-M [minimum post-trim read length]

-H [number of bases to hard trim from the end of reads]

-P [fasta file of ISSR motifs used]

-K [kmer choice for ABYSS assembly]

-L [minimum assembled contig length]

-N [negative reference in fasta format to filter reads against, ex. sequenced plastome or specific contaminant]

-X [bbduk trimming kmer, equal to or longer than shortest primer used]
```

## 6. Create BAM files (mapping reads to pseudoreference, filtering, removing duplicates, etc.)

```bash
nohup ISSRseq_CreateBAMs.sh -O 13Jan_pseudoref_2025_01_13_11_19 -T 32 &

# -T 32 = use 32 threads (max on myco)

```

## 7. Call biallelic variants with GATK4

```bash
-O [desired prefix of output directory]

-T [number of parallel processing threads -- I recommend not exceeding number of virtualized cores]

-P [ploidy of organism] --> O. undulatifolius is an octaploid (8N)!

### Command 

nohup ISSRseq_AnalyzeBAMs.sh -O 13Jan_pseudoref_2025_01_13_11_19 -T 30 -P 8 &

# output will be a vcf file called 'filtered_snps.vcf'
```

