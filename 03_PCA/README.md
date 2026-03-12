In order to performe a PCA, from my dataset ancient.merge I created a subset in wich I included the individuals in ancient.haplo.notri. The final subset in written inside ind2keep.txt
You can find the file here: [ind2keep.txt](./ind2keep)

```bash
cat <(cat /projects/mjolnir1/people/clx746/Romans/imputed/HG37/mesoneo.msd.inds2keep.txt) <(cut -f 1,2 ancient.haplo.notri.fam) > inds2keep.txt
```

The new subset, created in this way, is named neo.ancient.haplo.selection

```bash
plink --bfile ancient.merge --keep inds2keep.txt --make-bed --out neo.ancient.haplo.selection
```
It is really important do to a MAF filter, the final dataset now is neo.ancient.haplo.selection.var
```bash
plink --bfile neo.ancient.haplo.selection --maf 0.05 --geno 0.9 --make-bed --out neo.ancient.haplo.selection.var
```
In order to create a PCA, I need specidic file called EMU, to create those file i did:
```bash
echo '#!/bin/bash
#SBATCH -c 4
#SBATCH --mem-per-cpu 10G
#SBATCH -t 1400
module load emu/1.1.1

emu -b neo.ancient.haplo.selection.var -e 4 -t 1 -o neo.ancient.haplo.selection.var.emu

' > runEmu.sh
sbatch runEmu.sh
```
This script create two files: neo...emu.eigvals and neo...emu.eigvecs.

## R
Now we have to switch to R
```bash
info<-read.table("neo.impute.ho.20210824.sampleInfo.tsv", as.is=T, h=T, sep="\t")
eva<-readLines("neo.ancient.haplo.selection.var.emu.eigvals")
d<-read.table("neo.ancient.haplo.selection.var.emu.eigvecs", as.is=T)
colnames(d)<-c("pc1", "pc2", "pc3", "pc4")
fam<-read.table("neo.ancient.haplo.selection.var.fam", as.is=T)[,1]
```
