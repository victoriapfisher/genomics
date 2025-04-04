---
title: "Statistics for Genomics Exercises"
date: 2025-04-03
layout: post
output: jekyllthat::jekylldown
---

3.4 Statistics for Genomics Exercises

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

```{r}
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install(c('qvalue','plot3D','ggplot2','pheatmap','cowplot',
                      'cluster', 'NbClust', 'fastICA', 'NMF','matrixStats',
                      'Rtsne', 'mosaic', 'knitr', 'genomation',
                      'ggbio', 'Gviz', 'DESeq2', 'RUVSeq',
                      'gProfileR', 'ggfortify', 'corrplot',
                      'gage', 'EDASeq', 'citr', 'formatR',
                      'svglite', 'Rqc', 'ShortRead', 'QuasR',
                      'methylKit','FactoMineR', 'iClusterPlus',
                      'enrichR','caret','xgboost','glmnet',
                      'DALEX','kernlab','pROC','nnet','RANN',
                      'ranger','GenomeInfoDb', 'GenomicRanges',
                      'GenomicAlignments', 'ComplexHeatmap', 'circlize', 
                      'rtracklayer', 'BSgenome.Hsapiens.UCSC.hg38',
                      'BSgenome.Hsapiens.UCSC.hg19','tidyr',
                      'AnnotationHub', 'GenomicFeatures', 'normr',
                      'MotifDb', 'TFBSTools', 'rGADEM', 'JASPAR2018'
                     ))

options(timeout=300)

devtools::install_github("compgenomr/compGenomRData")

```


```{r}
enhancerFilePath=system.file("extdata",
                "subset.enhancers.hg18.bed",
                package="compGenomRData")
cpgiFilePath=system.file("extdata",
                "subset.cpgi.hg18.bed",
                package="compGenomRData")
# read enhancer marker BED file
enh.df <- read.table(enhancerFilePath, header = FALSE) 

# read CpG island BED file
cpgi.df <- read.table(cpgiFilePath, header = FALSE) 

# check first lines to see how the data looks like
head(enh.df)

head(cpgi.df)

save(cpgi.df,enh.df,file="mydata.RData")

```

```{r}
#CpG Island Dataset

cpgtFilePath=system.file("extdata",
                "CpGi.table.hg18.txt",
                package="compGenomRData")
cpgi=read.table(cpgtFilePath,header=TRUE,sep="\t")
head(cpgi)

hist(cpgi$perGc)

ggplot(cpgi, aes(perGc))+
  geom_boxplot()

GCclass<-function(my.gc){
  
  if(my.gc < 60){
    result="low"
    cat("low")
  }
  else if(my.gc > 75){  # check if GC value is higher than 75,      
                           #assign "high" to result
    result="high"
    cat("high")
  }else{ # if those two conditions fail then it must be "medium"
    result="medium"
  }
  
  return(result)
}
GCclass(10) # should return "low"
GCclass(90) # should return "high"
GCclass(65) # should return "medium"

gcValues=c(10,50,70,65,90)
for(i in gcValues){
  if(i < 60){
    result="low "
    cat("low")
  }
  else if(i > 75){  # check if GC value is higher than 75,      
                           #assign "high" to result
    result="high "
    cat("high")
  }else{ # if those two conditions fail then it must be "medium"
    result="medium "
    cat("medium")
  }
  
  return = c(result)
}


power2=function(x){ return(x^2)  }
    sapply(gcValues,power2)

```

```{r}
library(mosaic)
set.seed(21)
sample1= rnorm(50,20,5) # simulate a sample

# do bootstrap resampling, sampling with replacement
boot.means=do(1000) * mean(resample(sample1))

# get percentiles from the bootstrap means
q=quantile(boot.means[,1],p=c(0.025,0.975))

# plot the histogram
hist(boot.means[,1],col="cornflowerblue",border="white",
                    xlab="sample means")
abline(v=c(q[1], q[2] ),col="red")
text(x=q[1],y=200,round(q[1],3),adj=c(1,0))
text(x=q[2],y=200,round(q[2],3),adj=c(0,0))

#randomization based testing

set.seed(100)
gene1=rnorm(30,mean=4,sd=2)
gene2=rnorm(30,mean=2,sd=2)
org.diff=mean(gene1)-mean(gene2)
gene.df=data.frame(exp=c(gene1,gene2),
                  group=c( rep("test",30),rep("control",30) ) )


exp.null <- do(1000) * diff(mosaic::mean(exp ~ shuffle(group), data=gene.df))
hist(exp.null[,1],xlab="null distribution | no difference in samples",
     main=expression(paste(H[0]," :no difference in means") ),
     xlim=c(-2,2),col="cornflowerblue",border="white")
abline(v=quantile(exp.null[,1],0.95),col="red" )
abline(v=org.diff,col="blue" )
text(x=quantile(exp.null[,1],0.95),y=200,"0.05",adj=c(1,0),col="red")
text(x=org.diff,y=200,"org. diff.",adj=c(1,0),col="blue")

#multiple testing

library(qvalue)
data(hedenfalk)

qvalues <- qvalue(hedenfalk$p)$q
bonf.pval=p.adjust(hedenfalk$p,method ="bonferroni")
fdr.adj.pval=p.adjust(hedenfalk$p,method ="fdr")

plot(hedenfalk$p,qvalues,pch=19,ylim=c(0,1),
     xlab="raw P-values",ylab="adjusted P-values")
points(hedenfalk$p,bonf.pval,pch=19,col="red")
points(hedenfalk$p,fdr.adj.pval,pch=19,col="blue")
legend("bottomright",legend=c("q-value","FDR (BH)","Bonferroni"),
       fill=c("black","blue","red"))

#Moderated T-tests

set.seed(100)

#sample data matrix from normal distribution

gset=rnorm(3000,mean=200,sd=70)
data=matrix(gset,ncol=6)

# set groups
group1=1:3
group2=4:6
n1=3
n2=3
dx=rowMeans(data[,group1])-rowMeans(data[,group2])
  
require(matrixStats)

# get the esimate of pooled variance 
stderr = sqrt( (rowVars(data[,group1])*(n1-1) + 
       rowVars(data[,group2])*(n2-1)) / (n1+n2-2) * ( 1/n1 + 1/n2 ))

# do the shrinking towards median
mod.stderr = (stderr + median(stderr)) / 2 # moderation in variation

# esimate t statistic with moderated variance
t.mod <- dx / mod.stderr

# calculate P-value of rejecting null 
p.mod = 2*pt( -abs(t.mod), n1+n2-2 )

# esimate t statistic without moderated variance
t = dx / stderr

# calculate P-value of rejecting null 
p = 2*pt( -abs(t), n1+n2-2 )

par(mfrow=c(1,2))
hist(p,col="cornflowerblue",border="white",main="",xlab="P-values t-test")
mtext(paste("signifcant tests:",sum(p<0.05))  )
hist(p.mod,col="cornflowerblue",border="white",main="",
     xlab="P-values mod. t-test")
mtext(paste("signifcant tests:",sum(p.mod<0.05))  )

```

```{r}
#statistical distribution exercises

set.seed(100)

#sample data matrix from normal distribution
gset=rnorm(600,mean=200,sd=70)
data=matrix(gset,ncol=6)

sd(data) #70.85806

data=matrix(rnorm(6000,mean=200,sd=70),ncol=6)
sd(data) #69.65229

#Poisson simulation
pois1=rpois(30,lambda=5)
boot.means=do(1000) * mean(resample(pois1))
boxplot(boot.means)

t.test(pois1)
```
```{r}
#scatter plot

remove.packages("cli")
install.packages("cli")

rm(hmodFile)
hmodFile=system.file("extdata",
                    "HistoneModeVSgeneExp.rds",
                     package="compGenomRData")

hmod_df <- readRDS(hmodFile)
colnames(hmod_df)

ggplot(hmod_df, aes(H3k4me3, H3k27me3))+
  geom_point()+
  geom_smooth(method = "lm")

```

