## SNP calling workflow for *Asclepias* sequencing data  

### 0. DNA extracted from plant tissue using the Qiagen DNeasy kit  
Libraries prepared with the PlexWell 96/384 library prep kit. Sequencing (paired end 150 bp) done on the Illumina HiSeq 2500 (Puzey)/Illumina HiSeq 4000 (the rest).

### 1. Trimming with trimmomatic 0.38   
for file in *.fastq.gz; do

java -jar /opt/trimmomatic/0.38/prebuilt/trimmomatic-0.38.jar PE -threads 32 -phred33 -basein $file -baseout "trmd$file" ILLUMINACLIP:/opt/trimmomatic/0.38/prebuilt/adapters/TruSeq3-PE-2.fa:2:30:10 SLIDINGWINDOW:4:20 LEADING:3 TRAILING:3 MINLEN:36   
done

### 2. Check trimming quality with FastQC  
fastqc --noextract --nogroup -t 16 *.fastq.gz

### 3a. Keep reads succesfuly mapped to *A. syriaca* nuclear genome (BBSplit 38.91)  
for R1 in \*_1P.fastq.gz; do   
name=${R1%_1P.fastq.gz}   
R2=${R1%*_1P.fastq.gz}_2P.fastq.gz   
bbsplit.sh in1=$R1 in2=$R2 ref=/syriaca_nu.fasta basename=${name}nu%.fq   
done

### 3b. Filter out non-nuclear reads  
for i in \*nu_clean.fq; do   
bbsplit.sh in=$i ref=/references/syriaca_cp.fasta,/references/syriaca_mt.fasta basename=${i}.out_%.fq outu=${i}nu.clean.fq int=t   
done

### 4. Mapping to the reference (index the reference with bwa index before starting)  
module load bwa/0.7.17   
for i in *_nu_clean.fq; do   
bwa mem -t 32 \/references/syriaca_nu.fasta \$R1 '$R2 > /sams/${name:0:4}.sam   
done

### 5. Sam to bam conversion, fixmates  
module load samtools  
for i in *.sam; do   
samtools sort -@ 16 -n -O sam \'$i | samtools fixmate -@ 16 -m -O bam - ${i:0:4}.bam   
done

### 6. Samtools_sort  
module load samtools   
for i in *.bam; do   
samtools sort -@ 16 -O bam '$i > /sorted/${i:0:4}.bam   
done

### 7. Samtools_mark_duplicates  
module load samtools  
samtools markdup -@ 16 -r -S '$i /dupped/${i:0:4}.bam  
done

### 8. Filtering by mapping quality (30)  
module load samtools  
for i in *.bam; do  
samtools view -h -b -q 30 '$i > /Q30/${i:0:4}.bam  
done

### 9. Idex bam files  
module load samtools  
for i in *.bam; do samtools index -@ 16 '$i  
done  

### 10. SNP calling with freebayes-parallel 1.3.1  
module load anaconda2/2019.10  
freebayes-parallel /references/ref.fasta.100000.regions 32 -f /references/syriaca_nu.fasta -L samples.list --ploidy 2 > snps_all.vcf

### 11. Vcf compression  
module load htslib  
for i in *.vcf; do  
bgzip $i  
done  

### 12. Indexing with tabix  
module load htslib  
for i in *.gz; do 
tabix -p vcf '$i  
done  

### 13. Filtering with bcftools.  
**Parameters: remove indels, quality score > 30, 5 percent of missing data per locus, minor allele frequency > 0.05, mean depth > 3, no loci in LD (r2 > 0.2)).**

bcftools view --types snps snps_all.vcf | bcftools view -i 'MIN(FMT/DP)>3 & MIN(FMT/GQ)>30 & F_MISSING < 0.05 & MAF > 0.05' | bcftools +prune -m 0.2 -w 1000 -Ov -o snps_all_filtered.vcf
