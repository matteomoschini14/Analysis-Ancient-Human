One usefull tool is to discover the average depth of coverage of the final.bam (file of alignement)

```bash
echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 5G
#SBATCH -t 1400
module load samtools/1.12 

for i in *final.bam; do base=$(echo $i |perl -pe "s/_final.bam//g;"); samtools depth -Q 30 $i |grep -Pv "\t0" |cut -f 3 |sort |uniq -c >"$base"".doc"; done
' > getDoc.sh

sbatch getDoc.sh
```
## R

All the .doc file are assembled in one file .txt using R

```bash
files<-dir(pattern="doc$")

sam<-gsub(".doc", "", files)

doc<-NULL
for(i in 1:length(files)){
	a<-read.table(files[i])
	d<-sum(a[,2]*a[,1])/3095693981
	doc<-c(doc, d)
}

write.table(cbind(sam, doc), quote=F, col.names=F, row.names=F, sep="\t", file="Avg_doc.txt")
```
