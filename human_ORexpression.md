---
title: "Human ytranscriptome analysis"
date: '18 September, 2019'
output:
  html_document:
    keep_md: true
    fig_width: 5
    fig_height: 5
    code_folding: hide
    toc: true
    toc_float: 
      collapsed: false
---



We have RNA-seq data from 9 human olfactory mucosa biopsies (including the data from Saraiva et al., Sci Adv, 2019), plus three samples from Olender et al., BMC Genomics, 2016 (we only use the samples with paired-end data). The data was mapped with STAR and fragments in genes counted with HTSeq against the Ensembl annotation, version 95.


```r
data <- read.table(paste0(dir, "data/geneCounts_human.RAW.tsv"))

## separate gene annotation (which we obtined from the GTF used to count with HTSeq)
ann <- data[,1:6]
data <- data[,-c(1:6)]

## we'll work only with genes from standard chromosomes (autosomes, X, Y and MT)
sel <- ann$chr %in% unique(ann$chr)[1:25]
ann <- ann[sel,]
data <- data[sel,]
stopifnot(identical(row.names(data), row.names(ann)))
dim(data)
```

```
## [1] 58676    12
```

To normalise composition biases and differences in depth of sequencing, we use the method implemented in DESEq2.


```r
## estimte size factors
sf <- estimateSizeFactorsForMatrix(data)
plot(colSums(data), sf, pch=16, bty="l", xlab="library size", ylab="size factor")
abline(lm(sf~colSums(data)))
```

![](human_ORexpression_files/figure-html/normalise-1.png)<!-- -->

```r
## normalise
dataNorm <- t(t(data)/sf)
```

## OR repertoire

We are interested in the expression of OR genes. We have noted, however, that the proportion of OSNs in an olfactory epithelium sample is quite variable, and that affects the counts of OR genes within the rest of the transcriptome. For mouse, we used expression of genes known to be specific to OSNs to estimate the proportion of RNA contributed by OSNs (Ibarra-Soria et al., eLife, 2017).

We apply the same procedure here, using the same five OSN marker genes (and assuming that they are also specific to OSNs in humans) to estimate a size factor based on their geometric mean expression, as proposed in Khan et al., Mol Cell Neurosci, 2013. OR counts are then scaled by such factor to bring all replicates to a common scale. This is done on top of the depth normalisation performed above.

We can indeed see a correlation between total OR normalised counts and the expression of these OSN marker genes; all genes have strong Pearson correlation coefficients, with ADCY3 being the lowest at 0.85, but all others between 0.95 and 0.99.


```r
ensembl <- useMart(host = 'http://jan2019.archive.ensembl.org', biomart = 'ENSEMBL_MART_ENSEMBL', dataset = 'hsapiens_gene_ensembl')

gene.info <- getBM(attributes=c('ensembl_gene_id', 'ensembl_transcript_id', 'external_gene_name', 'chromosome_name', 'description', 'transcript_length'), mart = ensembl)

## retain only OR genes
## spelling error in ENSG00000279639	AC135068.5	olfactory receceptor pseudogene. Correct
gene.info[gene.info$external_gene_name=="AC135068.5",]$description <- "olfactory receptor pseudogene"
gene.info <- gene.info[grep("olfactory receptor", gene.info$description),]

## retain only the longest transcript for each gene
gene.info.longest <- t(sapply(unique(gene.info$ensembl_gene_id), function(x) gene.info[gene.info$ensembl_gene_id == x,][which.max(gene.info[gene.info$ensembl_gene_id == x,]$transcript_length),]))
gene.info.longest <- as.data.frame(apply(gene.info.longest, 2, unlist))

## create a data frame for OR expression only
sel <- intersect(row.names(gene.info.longest), row.names(dataNorm)) ## removed ORs in non-standard chromosomes
ors <- gene.info.longest[sel,]
gene.info.longest <- gene.info.longest[sel,]

## get normalised counts
ors <- dataNorm[row.names(ors),]

## check for evidence of OSN abundance differences
mature <- c("OMP", "CNGA2", "ADCY3", "GNAL", "ANO2")
mature <- dataNorm[row.names(ann[ann$gene %in% mature,]),]
row.names(mature) <- ann[row.names(mature),1]

par(mfrow=c(2,3))
plot(colSums(ors)/1e3, main="total OR counts", pch=16, axes=FALSE, xlab="", ylab="normalised expression x 1000")
box(bty="l"); axis(2, las=2)
for(i in 1:nrow(mature)){
  plot(mature[i,]/1e3, main=row.names(mature)[i], pch=16, axes=FALSE, xlab="", ylab="normalised expression x 1000")
  box(bty="l"); axis(2, las=2)
  legend("topright", legend=paste("r =", round(cor(mature[i,], colSums(ors)),2)), bty="n")
}
```

![](human_ORexpression_files/figure-html/orRepertoire-1.png)<!-- -->

We apply the normalisation to bring all counts to a common scale.


```r
geoMean <- geometric.mean(mature)
sf <- mean(geoMean)/geoMean
orsNorm <- as.data.frame(t(t(ors)*sf))
write.table(orsNorm, paste0(dir, "data/geneCounts_human_ORgenes.geoMeanNORMALISED.tsv"), quote = FALSE, sep="\t")

plot(c(colSums(ors)/1e3, colSums(orsNorm)/1e3), pch=16, col=rep(c("black", "red"), each=12), ylab="total OR counts (thousands)", xlab="", axes=FALSE)
box(bty="l"); axis(2, las=2)
mtext(side=3, line=1, text = c("RAW", "NORMALISED"), col=c("black", "red"), at=c(6.5,18.5))
```

![](human_ORexpression_files/figure-html/geoMeanNormalisation-1.png)<!-- -->

```r
means <- rowMeans(orsNorm)
```

There are 218 (25%) OR genes that are not expressed in any of the samples. The remaining span three orders of magnitude, with a few abundant receptors, but most expressed at low levels.


```r
barplot(log10(means[order(means, decreasing = TRUE)]+1), names.arg = "", ylab="normalised counts", axes = FALSE, ylim=c(0,3))
axis(2, at=0:3, labels = c(0,10,100,1000), las=2)
```

![](human_ORexpression_files/figure-html/meanExpr-1.png)<!-- -->

Unsurprisingly, protein-coding genes are expressed at much higher levels than pseudogenes. 


```r
tmp <- data.frame(expr=means, status=as.character(ann[names(means),'biotype']))
tmp$status <- ifelse(tmp$status == "protein_coding", "protein_coding", "pseudogene")

ggplot(tmp, aes(status, log10(expr+1))) + geom_violin(scale="width", trim = TRUE) + ylab(expression('log'[10]*' normalised counts')) + xlab("") + geom_boxplot(width=0.05) + th
```

![](human_ORexpression_files/figure-html/coding_vs_noncoding-1.png)<!-- -->

```r
# wilcox.test(tmp[tmp$status!="pseudogene",1], tmp[tmp$status=="pseudogene",1], alternative = "greater") # p-value < 2.2e-16
```

While nearly a quarter of receptor genes are not detected in any of the samples, 16% of the repertoire is consistently observed in all 12 samples, and almost a third in 9 or more.


```r
tmp <- apply(orsNorm, 2, function(x) ifelse(x>0, 1, 0)) # binarise

# table(rowSums(tmp))
# round(table(rowSums(tmp))/nrow(tmp)*100,2)

heatmap.2(tmp[order(rowSums(tmp), decreasing = TRUE), order(colSums(tmp))], trace="none", Rowv = FALSE, Colv = FALSE, col=brewer.pal(n=8, "Blues"), dendrogram = 'none', labRow = NA, key.xlab = "expressed")
```

![](human_ORexpression_files/figure-html/propExpr-1.png)<!-- -->

### Expression versus gene model completeness

We have used the RNA-seq data to generate gene models for these genes, adding the non-coding regions to the annotation. This is only possible when there is enough expression data to infer a model confidently. Gene models for receptors with little to no expression only contain the coding sequence identified through homology to other OR genes. 

However, it is very likely that a significant proportion of genes for which we have been able to generate a longer gene model are still incomplete, and their UTRs would be longer if more data were available.

To check this, we can look at the relationship between OR expression and gene length; we use the length of the longest annotated transcript for each gene. Of course longer genes produce more reads, so we divide by gene length (in kilobases) to account for this. 


```r
stopifnot(identical(names(means), row.names(gene.info.longest)))

tmp <- data.frame(expr=means, length=as.numeric(levels(gene.info.longest$transcript_length))[gene.info.longest$transcript_length])
## normalise expression by lenght in kb
tmp$exprNorm <- tmp$expr/(tmp$length/1e3)

tmp$type <- ifelse(ann[row.names(tmp),]$biotype == "protein_coding", "protein_coding", "pseudogene")
```

We bin the genes based on their length:

* 0 -> x < 1100bp

* 1 -> 1100 < x < 2000

* 2 -> 2000 < x < 3000

* 3 -> 3000 < x < 4000

* 4 -> 4000 < x < 5000

* 5 -> 5000 < x < 7500

* 6 -> x > 7500

As expected, gens from bin 0 which only contain the CDS have the lowest expression values. The number of reads then increases for bins 1, 2 and 3. For genes longer than 3kb, the amount of reads per kb doesn't change significantly.


```r
tmp$bin <- as.factor(ifelse(tmp$length < 1100, 0, ifelse(tmp$length < 2000, 1, ifelse(tmp$length < 3000, 2, ifelse(tmp$length < 4000, 3, ifelse(tmp$length < 5000, 4, ifelse(tmp$length < 7500, 5, 6)))))))

data_summary <- function(x) {
   m <- mean(x)
   ymin <- m-sd(x)
   ymax <- m+sd(x)
   return(c(y=m,ymin=ymin,ymax=ymax))
}

ggplot(tmp, aes(bin, log10(exprNorm+1))) + geom_violin() + geom_jitter(shape=16, position=position_jitter(0.1), aes(colour=bin)) + xlab("gene length") + ylab("normalised counts per kilobase") + scale_colour_brewer(palette = "YlOrBr") + stat_summary(fun.data=data_summary, colour="darkgrey") + th + theme(legend.position = "none")
```

![](human_ORexpression_files/figure-html/incompleteModels-1.png)<!-- -->

```r
# table(tmp$bin)
# 0   1   2   3   4   5   6 
# 485 204  67  45  27  30  14
# wilcox.test(tmp[tmp$bin==0,]$exprNorm, tmp[tmp$bin==1,]$exprNorm, alternative = "less") # p-value < 2.2e-16
# wilcox.test(tmp[tmp$bin==1,]$exprNorm, tmp[tmp$bin==2,]$exprNorm, alternative = "less") # p-value = 7.049e-11
# wilcox.test(tmp[tmp$bin==2,]$exprNorm, tmp[tmp$bin==3,]$exprNorm, alternative = "less") # p-value = 0.008521
# wilcox.test(tmp[tmp$bin==3,]$exprNorm, tmp[tmp$bin==4,]$exprNorm, alternative = "less") # p-value = 0.5551
# wilcox.test(tmp[tmp$bin==4,]$exprNorm, tmp[tmp$bin==5,]$exprNorm, alternative = "less") # p-value = 0.06382
# wilcox.test(tmp[tmp$bin==5,]$exprNorm, tmp[tmp$bin==6,]$exprNorm, alternative = "less") # p-value = 0.1557
```

Now we do the same analysis only with protein-coding genes, since pseudogenes have different regulation dynamics, and are more susceptible to degradation. We observe the same behaviour, with genes smaller than 3kb having significantly less reads, but no change thereafter.


```r
tmp2 <- tmp[tmp$type == "protein_coding",]
ggplot(tmp2, aes(bin, log10(exprNorm+1))) + geom_violin() + geom_jitter(shape=16, position=position_jitter(0.1), aes(colour=bin)) + xlab("gene length") + ylab("normalised counts per kilobase") + scale_colour_brewer(palette = "YlOrBr") + stat_summary(fun.data=data_summary, colour="darkgrey") + th + theme(legend.position = "none")
```

![](human_ORexpression_files/figure-html/incompleteModels_coding-1.png)<!-- -->

```r
# wilcox.test(tmp2[tmp2$length<=3000,]$exprNorm, tmp2[tmp2$length>3000,]$exprNorm, alternative = "less") # p-value < 2.2e-16
# wilcox.test(tmp2[tmp2$bin==0,]$exprNorm, tmp2[tmp2$bin==1,]$exprNorm, alternative = "less") # p-value = 0.0002219
# wilcox.test(tmp2[tmp2$bin==1,]$exprNorm, tmp2[tmp2$bin==2,]$exprNorm, alternative = "less") # p-value = 8.355e-09
# wilcox.test(tmp2[tmp2$bin==2,]$exprNorm, tmp2[tmp2$bin==3,]$exprNorm, alternative = "less") # p-value = 0.02289
# wilcox.test(tmp2[tmp2$bin==3,]$exprNorm, tmp2[tmp2$bin==4,]$exprNorm, alternative = "less") # p-value = 0.4038
# wilcox.test(tmp2[tmp2$bin==4,]$exprNorm, tmp2[tmp2$bin==5,]$exprNorm, alternative = "less") # p-value = 0.1252
# wilcox.test(tmp2[tmp2$bin==5,]$exprNorm, tmp2[tmp2$bin==6,]$exprNorm, alternative = "less") # p-value = 0.239
```

----

As an aside, the mouse data is of much better quality and has higher coverage of the repertoire compared to the human. So we can repeat the analysis with the mouse expression and gene length values.


```r
## save the data from https://doi.org/10.7554/eLife.21476.006 as plain text file 'mouseORexpr_IbarraSoria2017.tab'
## using only the 6 B6 samples
## annotation is Ensembl v72

## expr
ors.mouse <- read.table(paste0(dir, "data/mouseORexpr_IbarraSoria2017.tab"), header = TRUE, stringsAsFactors = FALSE)
ors.mouse$mean <- apply(ors.mouse[,7:12], 1, mean)

## this data already includes a length entry, but this is considering old models, before any of the annotation from this study
## so we include the revised length for the longest transcript after gene model curation
length <- read.table(paste0(dir, "data/mouseOR_length.tsv"), header = TRUE)

## combine
sel <- intersect(length$gene, ors.mouse$gene)
# 1241 out of the 1249 are found
# the remaining have changed name:
# Olfr406-ps to Olfr406
# Olfr1116-ps to Olfr1116
# Olfr418-ps1 to Olfr418
# Olfr1375-ps1 to Olfr1375
# Olfr1433 to Olfr1434
# Olfr179 to Olfr322
# Olfr144 has been removed
# amend the names so that we can match properly
ors.mouse[ors.mouse$gene == "Olfr406-ps",]$gene <- "Olfr406"
ors.mouse[ors.mouse$gene == "Olfr1116-ps",]$gene <- "Olfr1116"
ors.mouse[ors.mouse$gene == "Olfr418-ps1",]$gene <- "Olfr418"
ors.mouse[ors.mouse$gene == "Olfr1375-ps1",]$gene <- "Olfr1375"
ors.mouse[ors.mouse$gene == "Olfr1433",]$gene <- "Olfr1434"
ors.mouse[ors.mouse$gene == "Olfr179",]$gene <- "Olfr322"
sel <- intersect(length$gene, ors.mouse$gene)
tmp <- ors.mouse[ors.mouse$gene %in% sel,]
tmp$length2 <- length[match(tmp$gene, length$gene),]$length 

tmp$meanNorm <- tmp$mean/(tmp$length2/1e3)

## biotype
tmp$type <- length[match(tmp$gene, length$gene),]$biotype
```

And we observe the same, except in this case there is still a significant increase between genes in bins 3 and 4, suggesting that models of up to 4kb may be incomplete, after which point longer models are not the result of more data. 


```r
tmp$bin <- as.factor(ifelse(tmp$length < 1100, 0, ifelse(tmp$length < 2000, 1, ifelse(tmp$length < 3000, 2, ifelse(tmp$length < 4000, 3, ifelse(tmp$length < 5000, 4, ifelse(tmp$length < 7500, 5, 6)))))))

ggplot(tmp, aes(bin, log10(meanNorm+1))) + geom_violin() + geom_jitter(shape=16, position=position_jitter(0.1), aes(colour=bin)) + xlab("gene length") + ylab("normalised counts per kilobase") + scale_colour_brewer(palette = "YlOrBr") + stat_summary(fun.data=data_summary, colour="darkgrey") + th + theme(legend.position = "none") 
```

![](human_ORexpression_files/figure-html/incompleteModels_mouse-1.png)<!-- -->

```r
# table(tmp$bin)
# 0   1   2   3   4   5   6 
# 239 284 279 195 143  89  13 
# wilcox.test(tmp[tmp$bin==0,]$meanNorm, tmp[tmp$bin==1,]$meanNorm, alternative = "less") # p-value < 2.2e-16
# wilcox.test(tmp[tmp$bin==1,]$meanNorm, tmp[tmp$bin==2,]$meanNorm, alternative = "less") # p-value = 3.563e-09
# wilcox.test(tmp[tmp$bin==2,]$meanNorm, tmp[tmp$bin==3,]$meanNorm, alternative = "less") # p-value = 0.004703
# wilcox.test(tmp[tmp$bin==3,]$meanNorm, tmp[tmp$bin==4,]$meanNorm, alternative = "less") # p-value = 0.002967
# wilcox.test(tmp[tmp$bin==4,]$meanNorm, tmp[tmp$bin==5,]$meanNorm, alternative = "less") # p-value = 0.8751
# wilcox.test(tmp[tmp$bin==5,]$meanNorm, tmp[tmp$bin==6,]$meanNorm, alternative = "less") # p-value = 0.1104
```

And the same is true if we restrict to genes only. 


```r
tmp2 <- tmp[tmp$type == "gene",]
ggplot(tmp2, aes(bin, log10(meanNorm+1))) + geom_violin() + geom_jitter(shape=16, position=position_jitter(0.1), aes(colour=bin)) + xlab("gene length") + ylab("normalised counts per kilobase") + scale_colour_brewer(palette = "YlOrBr") + stat_summary(fun.data=data_summary, colour="darkgrey") + th + theme(legend.position = "none")
```

![](human_ORexpression_files/figure-html/incompleteModels_coding_mouse-1.png)<!-- -->

```r
# wilcox.test(tmp2[tmp2$bin==0,]$meanNorm, tmp2[tmp2$bin==1,]$meanNorm, alternative = "less") # p-value < 2.2e-16
# wilcox.test(tmp2[tmp2$bin==1,]$meanNorm, tmp2[tmp2$bin==2,]$meanNorm, alternative = "less") # p-value = 9.561e-08
# wilcox.test(tmp2[tmp2$bin==2,]$meanNorm, tmp2[tmp2$bin==3,]$meanNorm, alternative = "less") # p-value = 0.004202
# wilcox.test(tmp2[tmp2$bin==3,]$meanNorm, tmp2[tmp2$bin==4,]$meanNorm, alternative = "less") # p-value = 0.003728
# wilcox.test(tmp2[tmp2$bin==4,]$meanNorm, tmp2[tmp2$bin==5,]$meanNorm, alternative = "less") # p-value = 0.8758
# wilcox.test(tmp2[tmp2$bin==5,]$meanNorm, tmp2[tmp2$bin==6,]$meanNorm, alternative = "less") # p-value = 0.1848

# wilcox.test(tmp2[tmp2$length<=3000,]$meanNorm, tmp2[tmp2$length>3000,]$meanNorm, alternative = "less") # p-value < 2.2e-16
```

## Split OR genes

We have identified a number of genes that encode a putatively functional protein across two exons. Are these expressed at the same levels as protein-coding genes or do they resemble more the expression pattern of pseudogenes?

As seen before, pseudogenes are expressed at much lower levels, but intron-split genes are in the same range observed for protein-coding genes.


```r
# human
split.human <- c("OR4F5","OR2I1P","OR5D3P","OR5BS1P","OR4Q3","OR51C1P","OR10R2","OR2V1","OR52K1","OR10AG1","OR4F17","OR7E24","OR10H4")

orsNorm$mean <- apply(orsNorm, 1, mean)
orsNorm$type <- ifelse(ann[row.names(orsNorm),]$biotype == "protein_coding", "protein_coding", "pseudogene")
orsNorm$split <- ifelse(ann[row.names(orsNorm),]$gene %in% split.human, 1, 0)

orsNorm$class <- ifelse(orsNorm$type == "pseudogene", "pseudogene", ifelse(orsNorm$split==0, "protein_coding", "intron_split"))
plots <- list()
plots[[1]] <- ggplot(orsNorm, aes(class, log10(mean+1))) + geom_boxplot() + ggtitle("human") + th

## mouse
# in the mouse data, Olfr1291-ps is Olfr1291-ps1
# Olfr560 doesn't exist
split.mouse <- c("Olfr104-ps","Olfr105-ps","Olfr106-ps","Olfr1116","Olfr1117-ps1","Olfr1118","Olfr1123","Olfr1174-ps","Olfr1175-ps","Olfr1177-ps","Olfr1183","Olfr1289","Olfr1291-ps1","Olfr1293-ps","Olfr1331","Olfr1333","Olfr1358","Olfr239","Olfr240-ps1","Olfr286","Olfr287","Olfr288","Olfr324","Olfr55","Olfr560","Olfr592","Olfr607","Olfr680-ps1","Olfr682-ps1","Olfr718-ps1","Olfr735","Olfr745","Olfr764","Olfr766","Olfr844","Olfr857","Olfr869","Olfr872","Olfr873","Olfr18","Olfr94")

ors.mouse$type <- ifelse(length[match(ors.mouse$gene, length$gene),]$biotype == "gene", "protein_coding", "pseudogene")
ors.mouse$class <- ifelse(ors.mouse$gene %in% split.mouse, "intron_split", ors.mouse$type)
ors.mouse <- ors.mouse[!is.na(ors.mouse$class),]

plots[[2]] <- ggplot(ors.mouse, aes(class, log10(mean+1))) + geom_boxplot() + ggtitle("mouse") + th
ggarrange(plotlist = plots, ncol = 2, nrow = 1) 
```

![](human_ORexpression_files/figure-html/intron-split-1.png)<!-- -->

```r
# wilcox.test(orsNorm[orsNorm$class == "intron_split",]$mean, orsNorm[orsNorm$class == "protein_coding",]$mean, alternative = "less") # 0.9893
# wilcox.test(orsNorm[orsNorm$class == "intron_split",]$mean, orsNorm[orsNorm$class == "pseudogene",]$mean, alternative = "greater") # 1.024e-07
# 
# wilcox.test(ors.mouse[ors.mouse$class == "intron_split",]$mean, ors.mouse[ors.mouse$class == "protein_coding",]$mean, alternative = "less") # 0.2999
# wilcox.test(ors.mouse[ors.mouse$class == "intron_split",]$mean, ors.mouse[ors.mouse$class == "pseudogene",]$mean, alternative = "greater") # 2.2e-16
```

Additionally, there are no differences in the intron length between the intron-split and other protein-coding OR genes.


```r
## intron lengths for protein-coding OR genes
intronLength.human <- read.table(paste0(dir, "data/OR_intron_sizes_human.txt"), sep="\t", header = TRUE)
intronLength.mouse <- read.table(paste0(dir, "data/OR_intron_sizes_mouse.txt"), sep="\t", header = TRUE)

## the intron before the CDS is normally the most three prime, so we make a note of that
## in the table, all intron lengths are ordered from 5' to 3' for each gene, so we retain the last one
intronLength.human$threePrime <- apply(intronLength.human[,-c(1:3)], 1, function(x) x[max(which(!is.na(x[-length(x)])))] )
intronLength.mouse$threePrime <- apply(intronLength.mouse[,-c(1:3)], 1, function(x) x[max(which(!is.na(x[-length(x)])))] )

## annotate which genes are split
intronLength.human$split <- ifelse(intronLength.human$Gene_symbol %in% split.human, 1, 0)
intronLength.mouse$split <- ifelse(intronLength.mouse$Gene_symbol %in% split.mouse, 1, 0)
```

There are no differences in the size of the introns for the split genes compared to the rest of the protein-coding repertoire, when considering all introns from all transcripts.


```r
# human
intron.h <- unlist(c(intronLength.human[intronLength.human$split==0,4:8])) # combine all intron lengths
intron.h <- intron.h[!is.na(intron.h)] ## remove NAs
intron.h <- intron.h[intron.h>0] ## remove 0s, which indicate the gene has no introns
# length(intron.h) # 386 introns

intron.h.split <- unlist(c(intronLength.human[intronLength.human$split==1,4:8]))
intron.h.split <- intron.h.split[!is.na(intron.h.split)]
intron.h.split <- intron.h.split[intron.h.split>0]

# mouse
intron.m <- unlist(c(intronLength.mouse[intronLength.mouse$split==0,4:9])) # combine all intron lengths
intron.m <- intron.m[!is.na(intron.m)] ## remove NAs
intron.m <- intron.m[intron.m>0] ## remove 0s, which indicate the gene has no introns
# length(intron.m) # 2585 introns

intron.m.split <- unlist(c(intronLength.mouse[intronLength.mouse$split==1,4:9]))
intron.m.split <- intron.m.split[!is.na(intron.m.split)]
intron.m.split <- intron.m.split[intron.m.split>0]

par(mfrow=c(1,2))
boxplot(log10(intron.h), log10(intron.h.split), ylab=expression('log'[10]*' intron length (bp)'), names=c("protein-coding", "intron-split"), main="human")
boxplot(log10(intron.m), log10(intron.m.split), ylab=expression('log'[10]*' intron length (bp)'), names=c("protein-coding", "intron-split"), main="mouse")
```

![](human_ORexpression_files/figure-html/allIntrons-1.png)<!-- -->

```r
# wilcox.test(intron.h, intron.h.split) # p-value = 0.8524
# wilcox.test(intron.m, intron.m.split) # p-value = 0.08994
```

In most cases, the CDS is in the most 3' exon. SO we can restrict the analysis to only the most 3' intron, which would be the intron interrupting the ORF in the split genes, and the intron just before the ORF in the rest. Again, there are no significant differences in the lengths of the introns.


```r
par(mfrow=c(1,2))
tmp <- intronLength.human[intronLength.human$threePrime>0,]
boxplot(log10(tmp$threePrime)~tmp$split, ylab=expression('log'[10]*' intron length (bp)'), names=c("protein-coding", "intron-split"), main="human")
# wilcox.test(tmp[tmp$split==0,]$threePrime, tmp[tmp$split==1,]$threePrime) # p-value = 0.2847 (one-tail 0.1423)

tmp <- intronLength.mouse[intronLength.mouse$threePrime>0,]
boxplot(log10(tmp$threePrime)~tmp$split,  ylab=expression('log'[10]*' intron length (bp)'), names=c("protein-coding", "intron-split"), main="mouse")
```

![](human_ORexpression_files/figure-html/intronCDS-1.png)<!-- -->

```r
# wilcox.test(tmp[tmp$split==0,]$threePrime, tmp[tmp$split==1,]$threePrime) # p-value = 0.1232 (one-tail 0.0616)
```

Overall, the split OR genes do not show any distinct characteristics compared to the rest of the protein-coding repertoire. The are expressed at the same levels and have the same gene structure.



```r
sessionInfo()
```

```
## R version 3.5.3 (2019-03-11)
## Platform: x86_64-apple-darwin15.6.0 (64-bit)
## Running under: OS X El Capitan 10.11.6
## 
## Matrix products: default
## BLAS: /Library/Frameworks/R.framework/Versions/3.5/Resources/lib/libRblas.0.dylib
## LAPACK: /Library/Frameworks/R.framework/Versions/3.5/Resources/lib/libRlapack.dylib
## 
## locale:
## [1] en_GB.UTF-8/en_GB.UTF-8/en_GB.UTF-8/C/en_GB.UTF-8/en_GB.UTF-8
## 
## attached base packages:
## [1] parallel  stats4    stats     graphics  grDevices utils     datasets 
## [8] methods   base     
## 
## other attached packages:
##  [1] biomaRt_2.38.0              ggpubr_0.2                 
##  [3] magrittr_1.5                ggplot2_3.1.0              
##  [5] gplots_3.0.1.1              RColorBrewer_1.1-2         
##  [7] psych_1.8.12                DESeq2_1.22.2              
##  [9] SummarizedExperiment_1.12.0 DelayedArray_0.8.0         
## [11] BiocParallel_1.16.6         matrixStats_0.54.0         
## [13] Biobase_2.42.0              GenomicRanges_1.34.0       
## [15] GenomeInfoDb_1.18.2         IRanges_2.16.0             
## [17] S4Vectors_0.20.1            BiocGenerics_0.28.0        
## 
## loaded via a namespace (and not attached):
##  [1] nlme_3.1-137           bitops_1.0-6           bit64_0.9-7           
##  [4] progress_1.2.0         httr_1.4.0             tools_3.5.3           
##  [7] backports_1.1.3        R6_2.4.0               rpart_4.1-13          
## [10] KernSmooth_2.23-15     Hmisc_4.2-0            DBI_1.0.0             
## [13] lazyeval_0.2.1         colorspace_1.4-0       nnet_7.3-12           
## [16] withr_2.1.2            tidyselect_0.2.5       gridExtra_2.3         
## [19] prettyunits_1.0.2      mnormt_1.5-5           curl_3.3              
## [22] bit_1.1-14             compiler_3.5.3         htmlTable_1.13.1      
## [25] labeling_0.3           caTools_1.17.1.2       scales_1.0.0          
## [28] checkmate_1.9.1        genefilter_1.64.0      stringr_1.4.0         
## [31] digest_0.6.18          foreign_0.8-71         rmarkdown_1.12        
## [34] XVector_0.22.0         base64enc_0.1-3        pkgconfig_2.0.2       
## [37] htmltools_0.3.6        htmlwidgets_1.3        rlang_0.3.1           
## [40] rstudioapi_0.9.0       RSQLite_2.1.1          gtools_3.8.1          
## [43] acepack_1.4.1          dplyr_0.8.0.1          RCurl_1.95-4.12       
## [46] GenomeInfoDbData_1.2.0 Formula_1.2-3          Matrix_1.2-15         
## [49] Rcpp_1.0.0             munsell_0.5.0          stringi_1.4.3         
## [52] yaml_2.2.0             zlibbioc_1.28.0        plyr_1.8.4            
## [55] grid_3.5.3             blob_1.1.1             gdata_2.18.0          
## [58] crayon_1.3.4           lattice_0.20-38        cowplot_0.9.4         
## [61] splines_3.5.3          annotate_1.60.1        hms_0.4.2             
## [64] locfit_1.5-9.1         knitr_1.22             pillar_1.3.1          
## [67] geneplotter_1.60.0     XML_3.98-1.19          glue_1.3.0            
## [70] evaluate_0.13          latticeExtra_0.6-28    data.table_1.12.0     
## [73] gtable_0.2.0           purrr_0.3.1            assertthat_0.2.0      
## [76] xfun_0.5               xtable_1.8-3           survival_2.43-3       
## [79] tibble_2.0.1           AnnotationDbi_1.44.0   memoise_1.1.0         
## [82] cluster_2.0.7-1
```

