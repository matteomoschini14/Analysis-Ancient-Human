It is possibile, using the same file of before neo.ancient.haplo.selection.var, create a smart PCA

```bash
module load python/2.7.17
module load eigensoft/7.2.1

smartpca

echo "genotypename: "neo.ancient.haplo.selection.var".bed
snpname: "neo.ancient.haplo.selection.var".bim
indivname: "neo.ancient.haplo.selection.var".fam
outputformat: EIGENSTRAT
genotypeoutname: neo.ancient.haplo.selection.var.eigenstratgeno
snpoutname: neo.ancient.haplo.selection.var.snp
indivoutname: neo.ancient.haplo.selection.var.ind
familynames: YES
pordercheck: NO" > neo.ancient.haplo.selection.var.par


echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 10G
#SBATCH -t 1400

convertf -p neo.ancient.haplo.selection.var.par

' > convertf.sh

sbatch convertf.sh
```
```bash
echo "genotypename: neo.ancient.haplo.selection.var.eigenstratgeno
snpname: neo.ancient.haplo.selection.var.snp
indivname: neo.ancient.haplo.selection.var.ind
evecoutname: neo.ancient.haplo.selection.var.evec
evaloutname: neo.ancient.haplo.selection.var.eval
familynames: YES
numoutevec: 4
numthreads: 1
pordercheck: NO
lsqproject: NO" > neo.ancient.haplo.selection.var.pca.par

echo '#!/bin/bash
#SBATCH -c 1
#SBATCH --mem-per-cpu 10G
#SBATCH -t 1400

smartpca -p smartpca -p neo.ancient.haplo.selection.var.pca.par

' > smartpca.sh
sbatch smartpca.sh
```

## R 
```bash
module load gcc/13.2.0
module load openjdk/20.0.0
module load R/3.5.0

evec<-read.table("ancient.merge.selection.var.evec", as.is=T)
info<-read.table("neo.impute.ho.20210824.sampleInfo.tsv", as.is=T, sep="\t", h=T)
eval<-as.numeric(readLines("ancient.merge.selection.var.eval"))
fam<-read.table("ancient.merge.selection.var.fam", as.is=T)[,1]
samps<-sapply(strsplit(evec[,1], ":"), "[[", 1)


category<-NULL
region<-NULL
country<-NULL
popId<-NULL
groupAge<-NULL

for(i in 1:length(samps)){
	if(sum(info$sampleId==samps[i])>0){
		groupAge<-c(groupAge, info$groupAge[info$sampleId==samps[i]])
		region<-c(region, info$region[info$sampleId==samps[i]])
		country<-c(country, info$country[info$sampleId==samps[i]])
		popId<-c(popId, info$popId[info$sampleId==samps[i]])
	}else{
		groupAge <-c(groupAge, "RomanLondon")
		region <-c(region, "RomanLondon")
		country <-c(country, "RomanLondon")
		popId <-c(popId, "RomanLondon")
	}
}
m<-data.frame(region=region, PC1=evec[,2], PC2=evec[,3], sampleid=samps)

colvalues <-c("green", "blue", "orange", "lightsalmon1", "#F05837","#6465A5", "darkmagenta", "black")
names(colvalues)<-unique(m$region)

Pchs<-c(20, 4, 21, 24)
names(Pchs)<-c("Modern", "Ancient", "RomanLondon", "Unknown")

pdf("mesoneo2.smartpca.Romans.pdf", width=20, height=20, useDingbats=F)

ggplot(m, aes(x=PC2, y=PC1))+
geom_vline(xintercept=0, colour="gray", linetype = "longdash", size=0.5)+
geom_hline(yintercept=0, colour="gray", linetype = "longdash", size=0.5)+
geom_point(aes(colour=region, pch=groupAge), size=2, alpha=0.8, show.legend = T)+ scale_shape_manual(values=Pchs)+ 
scale_colour_manual(values=colvalues)+ 
#xlim(c(0.07, -0.04))+
geom_text_repel(data=m[m$region=="RomanLondon", ],aes(label=sampleid), segment.size=0.1, segment.color="black", size=1.5, box.padding=unit(0.001, 'npc', ), max.overlaps=150)+
ylab(paste0("PC1 (", round(eval[1], 2), "%)"))+
xlab(paste0("PC2 (", round(eval[2], 2), "%)"))+
theme(panel.background = element_rect(fill='white', colour='black'), axis.text.x = element_text(angle = 45, hjust = 1, colour='black'), axis.text.y = element_text(colour='black'), strip.background = element_rect(size=.2, colour="black", fill ="#FFA5004D"), panel.grid.major = element_blank(), panel.grid.minor = element_blank(), legend.title=element_text(size=10), legend.key=element_rect(fill="white", colour="white"), legend.position = "right", axis.title=element_text(size=14), legend.text=element_text(size=10))

dev.off()
```
