My file ancient.merge.fam is renamed ancient.merge.popnames.fam
```bash
cp ancient.merge.fam ancient.merge.popnames.fam
cp ancient.merge.bed ancient.merge.popnames.bed
cp ancient.merge.bim ancient.merge.popnames.bim
```

Then switch to R

## R
```bash
fam<-read.table("ancient.merge.popnames.fam", as.is=T)
info<-read.table("/projects/mjolnir1/people/clx746/Romans/imputed/neo.impute.ho.20210824.sampleInfo.tsv", as.is=T, h=T, sep="\t")

newfam<-fam
for(i in 1:length(fam[,1])){
	if(sum(fam[i,1]==info[,1])>0){
		newfam[i,1]<-info$groupLabel[fam[i,1]==info[,1]]
	}
}
newfam[,6]<-1

write.table(newfam, quote=F, sep="\t", col.names=F, row.names=F, file="ancient.merge.popnames.fam")
```
```bash
grep -e RL -e CanaryIslands_Guanche -e Estonia_VikingAge -e France_IronAge_Hallstatt -e France_IronAge_LaTene -e France_Mesolithic -e Germany_Medieval -e Hungary_Medieval_Langobard -e Ireland_Medieval -e Italy_Imperial -e Italy_IronAge -e Italy_Medieval -e Russia_IronAge_Sarmatian -e Ukraine_IronAge_Scythian -e Estonia_IronAge -e Britain_IronAge -e Lebanon_Roman -e SouthAfrica_Neolithic ancient.merge.popnames.fam | cut -f 1,2 > listokeep.txt
```
```bash
plink --bfile ancient.merge.popnames --keep listokeep.txt --make-bed --out ancient.merge.popnames.final
```

## R
```bash
library(admixtools)

prefix<-'ancient.merge.popnames.final'
my_f2_dir<-'f2data'

extract_f2(prefix, my_f2_dir, maxmiss=0.9999, adjust_pseudohaploid=T, overwrite=T)
options(future.globals.maxSize = 800 * 1024^2)

f2_blocks<-f2_from_precomp(my_f2_dir, remove_na=F)

resoult <- qpadm_rotate(f2_blocks, leftright = c("CanaryIslands_Guanche", "Estonia_VikingAge", "France_IronAge_Hallstatt" , "France_IronAge_LaTene" , "France_Mesolithic" , "Germany_Medieval" , "Hungary_Medieval_Langobard" , "Ireland_Medieval" , "Italy_Imperial" , "Italy_IronAge" , "Italy_Medieval" , "Russia_IronAge_Sarmatian" , "Ukraine_IronAge_Scythian" , "Estonia_IronAge" , "Britain_IronAge" , "Lebanon_Roman"), target = "RL16", rightfix = "SouthAfrica_Neolithic")
for (col in names(resoult)) {
  if (is.list(resoult[[col]])) {
    resoult[[col]] <- sapply(resoult[[col]], toString)
  }
}
write.table(resoult, file = "RL16.txt", sep = "\t")

Right <- NULL
Left <- NULL
Tab <- NULL
fam<-read.table("ancient.merge.final.fam", as.is=T)[,1]

for( s1 in 1:5) {
   for(s2 in 1:5) {
        if(fam[s1]!=fam[s2]) {
          left= fam[s2]
          right= c("CanaryIslands_Guanche", "Estonia_VikingAge","France_IronAge_Hallstatt" , "France_IronAge_LaTene" , "France_Mesolithic" ,"Germany_Medieval" ,"Hungary_Medieval_Langobard" , "Ireland_Medieval" ,         "Italy_Imperial" , "Italy_IronAge" , "Italy_Medieval" , "Russia_IronAge_Sarmatian" , "Ukraine_IronAge_Scythian" , "Estonia_IronAge" , "Britain_IronAge" ,"Lebanon_Roman")
          target= fam[s1]
          r <- qpadm(f2_blocks, left , right, target)
          Left <- c(Left,fam[s2])
          Target <- c(Target,fam[s1])
          p <- c(p, r$popdrop[,"p"])
          
     }
  }
}    
Tab<- data.frame( Target=Target, Left=Left , p=p)


left= fam[2]
right= c("CanaryIslands_Guanche", "Estonia_VikingAge","France_IronAge_Hallstatt" , "France_IronAge_LaTene" , "France_Mesolithic" ,"Germany_Medieval" ,"Hungary_Medieval_Langobard" , "Ireland_Medieval" ,         "Italy_Imperial" , "Italy_IronAge" , "Italy_Medieval" , "Russia_IronAge_Sarmatian" , "Ukraine_IronAge_Scythian" , "Estonia_IronAge" , "Britain_IronAge" ,"Lebanon_Roman")
target= fam[1]
r <- qpadm(f2_blocks, left , right, target)
Left = fam[2]
Right = fam[s1]
p = r$popdrop[,"p"]
Tab= data.frame( Right, Left , p)


Left <- c()
Target <- c()
p <- c()

for (s1 in 1:length(fam)) {
  for (s2 in 1:length(fam)) {
    if (fam[s1] != fam[s2]) {
      left <- fam[s2]
      right <- c("CanaryIslands_Guanche", "Estonia_VikingAge", "France_IronAge_Hallstatt", 
                 "France_IronAge_LaTene", "France_Mesolithic", "Germany_Medieval", 
                 "Hungary_Medieval_Langobard", "Ireland_Medieval", "Italy_Imperial", 
                 "Italy_IronAge", "Italy_Medieval", "Russia_IronAge_Sarmatian", 
                 "Ukraine_IronAge_Scythian", "Estonia_IronAge", "Britain_IronAge", 
                 "Lebanon_Roman")
      target <- fam[s1]
      r <- qpadm(f2_blocks, left, right, target)
      Left <- c(Left, fam[s2])
      Target <- c(Target, fam[s1])
      p <- c(p, as.numeric(r$rankdrop[1, "p"]))
    }
  }
}
Tab <- data.frame(Target = Target, Left = Left, p = p)
print(Tab)

write.table(Tab, "Conf.txt", sep = "\t", row.names = FALSE, quote = FALSE, col.names = TRUE)
```
