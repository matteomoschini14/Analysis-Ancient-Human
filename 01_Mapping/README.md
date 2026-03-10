# Mapping
The table that contains all the information from where I started is: 
First of all, i created a txt file: Scheibmap.txt wich every row is composed by the fastq.gz file, the ID name and the pathway where the file is found.
Every sequencing has 30 of mapping quality and single-end.

```bash
grep -v submitted_ftp filereport_read_run_PRJEB52707_tsv.txt | cut -f 7,8 |cut -f 1 -d ";" | while read l; do f=$(echo $l |cut -f 1 -d " " |cut -f 7 -d "/"); samp=$(echo $l |cut -f 12 -d "/" |cut -f 1 -d "."); echo "/projects/mjolnir1/people/jrx808/scheib/${f} 0 ${samp} se 30 ${samp} ILLUMINA ${samp} /projects/mjolnir1/people/clx746/Caribbean/HG37.1/hs.build37.1.fa"; done |perl -pe 's/ /\t/g;' > Scheibmap.txt
```

Once the table is created, I used a perl script to write it down all the command that will be launched in the next steps. 

```bash
perl /projects/mjolnir1/people/clx746/Scripts/MappingBWA_statsO.pl  -i Scheibmap.txt -p 4 -stats 0 -o Romans -tempdir temp
```
## Alignment
The script create a new directory: temp. It contains all the heavy files like Scheibmap.txt.log. This is the file I'm looking for, it contains all the commands for the alignment, but instead of launching all together, I divided it for make it light.   

Using a bash script, i took all the "bwa aln" inside the log file and put inside bwa_aln.sh. Once it is finished, I obtained a file .sai per individual, those are the information of the proper alingment, so this process is quite long.

```bash
grep 'bwa aln' temp/Scheibmap.txt.log > bwa_aln.sh

split -l 10 bwa_aln.sh 

for i in x??; do echo '#!/bin/bash

#
#SBATCH -c 4
#SBATCH --mem-per-cpu 5G
#SBATCH -t 4200
module load bwa/0.7.17 
' > t; cat $i >> t; mv t $i; done

for i in x??; do sbatch $i; sleep 1; done
```
Then, i did the same thing but with the samse. So the bwa_samse.sh was created.

```bash
grep 'bwa samse' temp/Scheibmap.txt.log > bwa_samse.sh

split -l 10 bwa_samse.sh 

for i in x??; do echo '#!/bin/bash
#
#SBATCH -c 1
#SBATCH --mem-per-cpu 5G
#SBATCH -t 4200
module load bwa/0.7.17 
module load samtools/1.12 
' > t; cat $i >> t; mv t $i; done

for i in x??; do sbatch $i; sleep 1; done
```
Bwa samse take the information inside the .sai file and create files .sam, wich are readable by the user, all the information of the alignment is written and also the quality.

## Filtering
Next I created bwa_filter.sh, where I excluded all the reads not mapped or with quality below 30 and created .bam files.

```bash
paste -d " " <(grep "samtools view -bF 4" temp/Scheibmap.txt.log |cut -f 9 -d " " |perl -pe 's/temp/samtools view -bF 4 -q 30 -o temp/g;' ) <(grep "samtools view -bF 4" temp/Scheibmap.txt.log |cut -f 7 -d " ") > bwa_filter.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 5G
#SBATCH -t 180
#SBATCH --array=1-96%10

module load samtools/1.12 

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p bwa_filter.sh)
echo $LINE
$LINE
' > bwa_filter.array.sh

sbatch bwa_filter.array.sh
```
## Sorting
What do to next, is to sorting all my reads and obtein file sorted_bam 

```bash
paste -d " " <(grep 'samtools sort' temp/Scheibmap.txt.log |cut -f 1,2 -d " ") <(grep 'samtools sort' temp/Scheibmap.txt.log |cut -f 4 -d " " |perl -pe 's/sorted/sorted.bam/g;'
|perl -pe 's/temp/-o temp/g;') <(grep 'samtools sort' temp/Scheibmap.txt.log |cut -f 3 -d " ")  > bwa_sort1.sh


echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 5G
#SBATCH -t 180
#SBATCH --array=1-96%10

module load samtools/1.12 

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p bwa_sort1.sh)
echo $LINE
$LINE
' > bwa.sort1.array.sh
```
## 1° index
Then, for every file .bam i created a file sorted.bam.bai, it is a index file

```bash
grep 'samtools index' temp/Scheibmap.txt.log  |grep -v dedup > index1.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 30G
#SBATCH -t 1000
#SBATCH --array=1-96%10

module load samtools/1.12 

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p index1.sh)
echo $LINE
$LINE
' > index1.array.sh

sbatch index1.array.sh
```
## Delete Deduplicate
```bash
grep 'picard MarkDuplicates' temp/Scheibmap.txt.log | perl -pe 's/picard/java -jar \/projects\/mjolnir1\/people\/clx746\/Scripts\/picard\/picard.jar/g;' >dedup.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 30G
#SBATCH -t 1000
#SBATCH --array=1-96%10

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p dedup.sh)
echo $LINE
$LINE
' > dedup.array.sh

sbatch dedup.array.sh
```
## 2° index
```bash
grep 'samtools index' temp/Scheibmap.txt.log |grep dedup > index2.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 30G
#SBATCH -t 1000
#SBATCH --array=1-96%10

module load samtools/1.12 

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p index2.sh)
echo $LINE
$LINE
' > index2.array.sh

sbatch index2.array.sh
```

```bash
grep 'RealignerTargetCreator' temp/Scheibmap.txt.log | perl -pe 's/GenomeAnalysisTK/gatk3/g;'  > raln.create.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 30G
#SBATCH -t 1400
#SBATCH --array=1-96%10

module unload python
module unload  openjdk/17.0.8
module unload gatk
module load gatk/3.6

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p raln.create.sh)
echo $LINE
$LINE
' > raln.create.array.sh

sbatch raln.create.array.sh
```
## Optimization
```bash
grep 'IndelRealigner' temp/Scheibmap.txt.log | perl -pe 's/GenomeAnalysisTK/gatk3/g;'  > raln.sh


echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 30G
#SBATCH -t 1400
#SBATCH --array=1-96%10

module unload python
module unload  openjdk/17.0.8
module unload gatk
module load gatk/3.6

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p raln.sh)
echo $LINE
$LINE
' > raln.array.sh

sbatch raln.array.sh
```
## Ricalibration
```bash
grep 'fillmd' temp/Scheibmap.txt.log   > fillmd.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 30G
#SBATCH -t 1400
#SBATCH --array=1-96%10

module load samtools/1.12 

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p fillmd.sh)
echo $LINE
$LINE
' > fillmd.array.sh

sbatch fillmd.array.sh
```
