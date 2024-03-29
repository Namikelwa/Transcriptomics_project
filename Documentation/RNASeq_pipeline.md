
# Bioinformatics analysis
## Dependencies

* sratoolkit/2.9
* fastqc/0.11.9
* multiqc/1.4
* trimmomatic/0.39
* hisat2/2.2.1
* samtools/1.9
* htseq/0.13.5
* star/2.7.6a
## Create a conda environment & load modules
```
conda create --env Anopheles
conda activate Anopheles
module load <name of the module>
```

## Fetching raw data from NCBI
The first step is to download the accession list into a txt file from the command line using **esearch**
```
esearch -db sra -query PRJNA560504 | efetch --format runinfo | cut -d "," -f 1 > SraAccList.txt
# esearch command uses the ESearch utility to search on the NCBI database for query and finds the unique identifiers for all query that match the search query.
#-db database
#efetch downloads selected records in a style designated by -format
```
Once you have the accession list,the next step is to download the data.
**fastq-dump** is a tool in SRAtoolkit used for downloading sequenced reads from NCBI Sequence Read Archive(SRA).The data is dowloaded in fastq format.Here we are using the two options *--gzip* for compressing the sequence reads and *--split-files* to give both forward and reverse reads since the reads are from Illumina platform.

Getting the data one data-set at a time


```
fastq-dump --gzip --split-files <Accession number>
fastq-dump --gzip --split-files SRR9987839
```
Getting all data at once
```
mkdir raw-data
cd raw-data
for i in $(cat …/SraAccList.txt);    #SraAcclis.txt contains a list of the accesion numbers
do
    echo $i
    fastq-dump --gzip --split-files $i  #fastq-dump gets data in fastq format
done
```
## Quality analysis
**fastqc** checks the quality of the raw reads from high throughput sequencing platforms like Illumina.It provides a html report summary of the quality of reads in graphs and tables.This makes you aware of the quality of reads for downstream analysis.
**multiqc** makes fastqc output more manageable by compiling them and generating one report.

```
mkdir rawfastqc
cd rawfastqc
for i in `../*.gz`;
do
  	fastqc $i
done
multiqc ./
```

![image](https://drive.google.com/uc?export=view&id=1WGZUvOHSZXobBbo305dQou5cMSNdgNun)                         



## Trimming
low quality reads together with adapters are removed after quality assessment.**Trimmomatic** is a  Java executable software used to trim and crop reads. 
```
java -jar <path to trimmomatic.jar> PE <input 1> <input 2>] <paired output 1> <unpaired output 1> <paired output 2> <unpaired output 2> <options>
```
```
#basename is a command that strips trailing suffix in a file name
for i in *_1.fastq.gz;
do
    name=$(basename ${i} _1.fastq.gz)
    trimmomatic PE ${i} ${name}_2.fastq.gz\
                   ${name}_1.trim.fastq.gz ${name}_1un.trim.fastq.gz \
                    ${name}_2.trim.fastq.gz ${name}_2un.trim.fastq.gz \
                    HEADCROP:11
done
PE - paired end
HEADCROP -removes the first 11 bases of the reads

```
The trimmed reads are then checked for quality.


![image](https://drive.google.com/uc?export=view&id=1s2scdPc_Es-1_qPEeqv8CVPoAjPLokUb)             


![image](https://drive.google.com/uc?export=view&id=1fvmBuAmgn0kgQW2bRF3ZNOyoMpxoLV9M)          



## Mapping
**hisat2** is a fast and sensitive splice-aware aligner that compresses the genome using an indexing scheme to reduce the amount of space needed to store the genome. This also makes the genome quick to search, using a whole-genome index.We use samtools to convert the output file from mapping to bam format and to index the bam files.Indexing creates a searchable index of sorted bam files required in some programs.

```
wget https://vectorbase.org/common/downloads/Current_Release/AgambiaePEST/fasta/data/VectorBase-53_AgambiaePEST_AnnotatedCDSs.fasta

#Building a reference genome index
hisat2-build -p25 ./VectorBase-53_AgambiaePEST_AnnotatedCDSs.fasta  ../hisat-index/VectorBase-53_AgambiaePEST.idx

#Run hisat2 using indexed reference
mkdir Alignment_hisat
for i in $(cat ../SraAccList.txt);
do
   echo ${i}
   hisat2 -p25 -x ../hisat-index/VectorBase-53_AgambiaePEST.idx\
               -1 ${i}_1.fastq.gz -2 ${i}_2.fastq.gz\
               -S Alignment_hisat/${i}_hisat.sam
   samtools view -Sb Alignment_hisat/${i}_hisat.sam  | samtools sort  > Alignment_hisat/${i}_hisat_sorted.bam
   samtools index Alignment_hisat/${i}_hisat_sorted.bam
   rm Alignment_hisat/${i}_hisat.sam 
done
```
### Output

Unfornately our overall alignment rate for each of the datasets was barely 50% as shown below,
```
SRR9987838 -50.70% overall alignment rate
SRR9987839 -49.69% overall alignment rate
SRR9987840 -49.46% overall alignment rate
SRR9987841 -36.97% overall alignment rate
```
#### Troubleshooting

We choose one data set (SRR9987840) for troubleshooting.

* Step 1

Use a different reference genome; VectorBase-53_AgambiaePEST_Genome.fasta.   
The  overall alignment rate rose to 83%


![image](https://drive.google.com/uc?export=view&id=1BAgqFHGlRBpc0dIsHD34AUIoPbCcH6La)          


* Step 2

Use a different aligner; STAR.    
The overall alignment rate was 86%


**STAR Aligner**(Spliced Transcripts Alignment to a Reference)

STAR is a splice aware aligner designed to specifically address many of the challenges of RNA-seq data.It shows high accuracy and mapping speed.Alignemnt in STAR involves two steps;

1. Creating genome index
```
STAR --runThreadN 6  # number of threads\
    --runMode genomeGenerate \
    --genomeDir ./starr #path to store genome indices\
    --genomeFastaFiles VectorBase-53_AgambiaePEST_Genome.fasta \
    --sjdbGTFfile VectorBase-53_AgambiaePEST.gff \
    --sjdbOverhang 99 #readlength-1 --sjdbGTFtagExonParentTranscript gene
```

2. Mapping reads to the genome
```
mkdir alignments
for i in $(cat SraAcclist.txt);
do
    STAR --genomeDir starr \
    --readFilesIn  ${i}_1.fastq.gz ${i}_2.fastq.gz\
    --readFilesCommand zcat  \
    --outSAMtype BAM SortedByCoordinate \
    --quantMode GeneCounts \
    --outFileNamePrefix alignments/${i}
done
#zcat decompress the files

```
### Output
```
SRR9987838 -86.83% overall alignment rate
SRR9987839 -86.80% overall alignment rate
SRR9987840 -85.57% overall alignment rate
SRR9987841 -89.64% overall alignment rate
```

## Abundance estimation

Once you have your aligned reads,**htseq** is used to give counts of reads mapped to each feature.A feature is an interval on a chromosome.
```
htseq-count -t exon -i gene_id -f bam *._hisat_sorted.bam  VectorBase-53_AgambiaePEST.gff > htseq/counts.txt
```
### Output


![image](https://drive.google.com/uc?export=view&id=11H3sS_UR2PY0yY_tLA8MoGh_Incr3BT2)         



## Differential analysis

Differential analysis involves using read counts to perform statistical analysis to discover quantitative changes in gene expression levels between experimental groups;exposed and non-exposed.**DESeq2** is used for differential analysis. 


```
library(DESeq2)
library(limma)
library(edgeR)
library(Biobase)
```
Reading data into R
```
#reading data and storing it to a variable countdata
countdata <- read.table("counts.txt",header=TRUE,row.names = 1)
#converting the data to matrix
countdata <- as.matrix(countdata)
#assign condition
condition <- factor(c(rep("ctrl",2),rep("exp",2)))
#analysis with DESeq
#create a coldata frame
coldata <- data.frame(row.names = colnames(countdata),condition)
```
Creating a deseqdataset object and running Deseq
```
dds <- DESeqDataSetFromMatrix(countData = countdata,colData = coldata,design =~condition)
#run DESeq pipeline
dds <- DESeq(dds)
```
Dispersion Estimate plot
```
plotDispEsts(dds)
# DE using edgeR
dge <- DGEList(counts=countdata, group=condition)
#normalize by total counts
dge <- calcNormFactors(dge)
# Create the contrast matrix
design.mat <- model.matrix(~ 0 + dge$samples$group)
colnames(design.mat) <- levels(dge$samples$group)
# Estimate dispersion parameter for GLM
dge <- estimateGLMCommonDisp(dge, design.mat)
dge <- estimateGLMTrendedDisp(dge, design.mat, method="power")
dge<- estimateGLMTagwiseDisp(dge,design.mat)
```
Mean Variance plot
```
# Plot mean-variance
plotBCV(dge)
```

### Results

![MA plot](https://user-images.githubusercontent.com/85280529/184815552-47e6c526-cd54-4c63-9703-4da4e19fac88.PNG)

![dispersion](https://user-images.githubusercontent.com/85280529/184815442-c8ecd61f-d250-4061-904b-e62730819c10.PNG)

![heatmap](https://user-images.githubusercontent.com/85280529/184815520-ec2dc9a9-53ba-4938-a160-bf0fe5ffed2b.PNG)

![PCoA](https://user-images.githubusercontent.com/85280529/184815600-11c559ab-9db5-4b37-95c3-5a27c931c97f.PNG)




