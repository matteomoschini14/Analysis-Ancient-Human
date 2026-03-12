## Create the Dataset
In order to performe a PCA, from my dataset ancient.merge I created a subset in wich I included the individuals in ancient.haplo.notri. The final subset in written inside ind2keep.txt
You can find the file here: [inds2keep.txt](./inds2keep.txt)

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

Roman<-read.table("Roman.tsv" , as.is=T, h=T, sep="\t")
groupAge<-NULL
region<-NULL
country<-NULL
popId<-NULL
subregion<-NULL
for(i in 1:length(fam)){
	if(sum(info$sampleId==fam[i])>0){
		groupAge<-c(groupAge, info$groupAge[info$sampleId==fam[i]])
		region<-c(region, info$region[info$sampleId==fam[i]])
		country<-c(country, info$country[info$sampleId==fam[i]])
		popId<-c(popId, info$popId[info$sampleId==fam[i]])
                subregion<-c(subregion, "Null")
	}else{
		groupAge <-c(groupAge, "RomanLondon")
		region <-c(region, "RomanLondon")
		country <-c(country, "RomanLondon")
		popId <-c(popId, "RomanLondon")
                subregion<-c(subregion, "good")
		message(fam[i])
	}
}
Roman<- Roman[, 1:(ncol(Roman) - 4)]
pcares<-data.frame(Sample=fam, pc1=d$pc1, pc2=d$pc2, pc3=d$pc3, pc4=d$pc4, groupAge=groupAge, region=region, country=country, popid=popId, subregion=subregion)
pcares$subregion <- Roman$subregion[match(pcares$Sample, Roman$sample)]

library(ggplot2)
library(ggrepel)


colvals<-c("purple", "black", "gold", "turquoise", "pink", "orange", "magenta", "green") 
names(colvals)<-unique(pcares$region)

fillvals <-c("purple", "black", "gold", "turquoise", "pink", "orange", "magenta", "green")
names(fillvals)<-unique(pcares$region) 

Pchs<-c(20, 4, 20, 24) 
names(Pchs)<-c("Modern", "Ancient", "RomanLondon", "Unknown")
pdf("mesoneo.pca.Romans.pdf", width = 18, height = 10, useDingbats = FALSE)

ggplot(pcares, aes(x = pc1, y = pc2)) +
  geom_vline(xintercept = 0, colour = "gray", linetype = "longdash") +
  geom_hline(yintercept = 0, colour = "gray", linetype = "longdash") +
  geom_point(aes(
    colour = region,
    shape = ifelse(
      groupAge == "RomanLondon",
      paste(groupAge, subregion, sep = "_"),
      groupAge
    )
  ), size = 2.5, alpha = 0.8, show.legend = TRUE) +
  scale_shape_manual(
    name = "shape",
    values = c(
      "Modern" = 23,
      "Ancient" = 4,
      "RomanLondon_Scheib" = 22,
      "RomanLondon_New" = 20,
      "RomanLondon_Reference" = 24,
      "Unknown" = 25
    )
  ) +
  scale_colour_manual(
    name = "colour",
    values = c(
      "CentralAsia" = "purple",
      "CentralEasternEurope" = "pink",
      "NorthAsia" = "gold",
      "NorthernEurope" = "turquoise",
      "RomanLondon" = "black",
      "SouthernEurope" = "orange",
      "WesternAsia" = "magenta",
      "WesternEurope" = "green"
    )
  ) +
  geom_text_repel(
    data = pcares[pcares$groupAge == "RomanLondon", ],
    aes(label = Sample),
    segment.size = 0.1,
    segment.color = "black",
    size = 1.5,
    box.padding = unit(0.001, 'npc'),
    max.overlaps = 100
  ) +
  xlab(paste0("PC1 (", round(as.numeric(eva[1]), 3), "%)")) +
  ylab(paste0("PC2 (", round(as.numeric(eva[2]), 3), "%)")) +
  theme(
    panel.background = element_rect(fill = 'white', colour = 'black'),
    axis.text.x = element_text(colour = 'black'),
    axis.text.y = element_text(colour = 'black'),
    strip.background = element_rect(size = 0.2, colour = "black", fill = "#FFA5004D"),
    panel.grid.major = element_blank(),
    panel.grid.minor = element_blank(),
    legend.title = element_text(size = 12),
    legend.key = element_rect(fill = "white", colour = "white"),
    axis.title = element_text(size = 12),
    legend.text = element_text(size = 9.5)
  )

dev.off()
```
