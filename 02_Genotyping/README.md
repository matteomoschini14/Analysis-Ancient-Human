First of all, I took from a reference genome (neo.impute.ho.20210824.haploid.bim) the SNPs site and I created a table which is composed by: the chromosome of the SNP, the location, the major and minor allele.
The table can be find here: [sites.txt](./sites.txt).

```bash
cut -f 1,4,5,6 /projects/mjolnir1/people/clx746/Romans/imputed/HG37/neo.impute.ho.20210824.haploid.bim  > sites.txt
```

Then I created a BAM list, where I put the path for every individual of my analysis (those of the previous step and other from different work)

```bash
ls /projects/mjolnir1/people/jrx808/scheib/mapping/*.bam  > BamFiles.txt
ls /projects/mjolnir1/people/clx746/Romans/MappingMA/*.bam  >> BamFiles.txt
ls /projects/mjolnir1/people/clx746/Romans/Reference/*.bam  >> BamFiles.txt
```
Now it is possible to compare the file bam with the SNPs site and obtain for every individual a file bam haployd (only one allele is manteined and the other discarded, randomly chosen) that contain only the SNP site, the other part of the genome can be discarded

```bash
echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 10G
#SBATCH -t 2880

module load angsd/0.940

angsd sites index sites.txt

' > gethaplo.sh

cat BamFiles.txt |while read l; do na=$(basename $l |cut -f 1 -d "_"); echo "angsd -dohaplocall 1 -i $l -out $na  -doCounts 1 -nThreads 1 -minMapQ 30 -minQ 20  -doMajorMinor 3 -sites sites.txt"; done >> gethaplo.sh

# run the script and wait until it is done:

sbatch gethaplo.sh
```
What I have are file haplo.gz, they must to be changed in file haplo.tped and haplo.tfam 

```bash
echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 10G
#SBATCH -t 1000

module load angsd/0.940

' > gethaplo2.sh

cat BamFiles.txt |while read l; do na=$(basename $l  |perl -pe 's/_final.bam//g;'); echo "/projects/mjolnir1/people/clx746/Scripts/angsd/misc/haploToPlink $na.haplo.gz $na.haplo"; done >> gethaplo2.sh

# run the script and wait until it is done:

sbatch gethaplo2.sh
```
Then the files haplo.tped and haplo.tfam need to be changed in the plink format: .bed .bim and .fam

```bash
cat BamFiles.txt |while read l; do na=$(basename $l  |perl -pe 's/_final.bam//g;'); plink --tfile $na".haplo" --make-bed --out $na.haplo; done

for i in *haplo.bim; do cat $i |perl -pe 's/_/:/g;' > t; mv t $i; done
```
Now i matched the reference pannel (neo.impute.ho.20210824.haploid.bim) with my .bin file (the allele randomly chosen) to see if it is compatible. If not, all the SNPs will be annoted with .missnp (triallelic site)
```bash
ls *haplo.bim |while read l; do s=$(echo $l |perl -pe 's/.haplo.bim//g;');  echo "/projects/mjolnir1/people/clx746/Scripts/plink --bfile /projects/mjolnir1/people/clx746/Romans/imputed/HG37/neo.impute.ho.20210824.haploid --bmerge $s.haplo --allow-no-sex --make-bed --out neo.$s" ; done > findtrisites.sh

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 10G
#SBATCH -t 180
#SBATCH --array=1-353%50

LINE=$(sed -n "$SLURM_ARRAY_TASK_ID"p findtrisites.sh) 
echo $LINE
$LINE
' > gettri.array.sh

sbatch gettri.array.sh
```
The new files will be annotated as neo.*.bed/bim/fam

Now I escluded all the missnp from my dataset

```bash
ls *haplo.bim |while read l; do s=$(echo $l |perl -pe 's/.haplo.bim//g;'); plink --bfile $s.haplo --exclude neo.$s-merge.missnp --allow-no-sex --make-bed --out $s.notri; done
```
The output will be files as .notri.bed/bim/fam
Now it is usefull to merge my file per individuals, the file ml contain for every row the files .notri.bed .notri.fam and .notri.bim for every individuls.
To create a genetic matrix, so to concatenate the individuals in one big file, the script is:

```bash
plink --merge-list ml --make-bed --allow-no-sex --out  ancient.haplo.notri
```
The file is ancient.haplo.notri.
