# Mapping
The table that contains all the information from where I started is: 
First of all, i created a txt file: Scheibmap.txt wich every row is composed by the fastq.gz file, the ID name and the pathway where the file is found.
Every sequencing has 30 of mapping quality and single-end.

```bash
grep -v submitted_ftp filereport_read_run_PRJEB52707_tsv.txt | cut -f 7,8 |cut -f 1 -d ";" | while read l; do f=$(echo $l |cut -f 1 -d " " |cut -f 7 -d "/"); samp=$(echo $l |cut -f 12 -d "/" |cut -f 1 -d "."); echo "/projects/mjolnir1/people/jrx808/scheib/${f} 0 ${samp} se 30 ${samp} ILLUMINA ${samp} /projects/mjolnir1/people/clx746/Caribbean/HG37.1/hs.build37.1.fa"; done |perl -pe 's/ /\t/g;' > Scheibmap.txt
```

Once the table is created, I used a perl script to align my FASTQ file to a reference genome thanks to MappingBWA. 

```bash
perl /projects/mjolnir1/people/clx746/Scripts/MappingBWA_statsO.pl  -i Scheibmap.txt -p 4 -stats 0 -o Romans -tempdir temp
```

The script also create a new directory: temp. It contains all the heavy files like Scheibmap.txt.log and it will be healpfull for the next step.

Using a bash script, i took all the "bwa aln" inside the log file and put inside bwa_aln.sh, where every row is a file .sai, one per individual.

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

