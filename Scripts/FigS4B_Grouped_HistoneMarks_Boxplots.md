```R
## Load the necessary packages
library(GenomicRanges)
library(edgeR)

## Process PCHiC data
# Import the interactions
Interactions <- read.delim("Data/Interactions/PCHiC/Interactions.txt", header = T)

# Remove interactions from or to blacklisted fragments
Blacklist <- read.delim("Data/Interactions/PCHiC/Annotation/Blacklisted_Fragments.txt", header=F)
FilteredInteractions <- Interactions[ !(Interactions$oeID %in% Blacklist[,1]),]
FilteredInteractions <- FilteredInteractions[ !(FilteredInteractions$baitID %in% Blacklist[,1]),]

# Filter away interactions in trans
FilteredInteractions <- FilteredInteractions[ FilteredInteractions$oeChr == FilteredInteractions$baitChr,]

# Filter away interactions spanning more than 1 megabase
FilteredInteractions <- FilteredInteractions[ abs(FilteredInteractions$dist) <= 1000000,]

# Remove baited fragments that are not genes, but ultra-conserved elements
UCE <- read.delim("Data/Interactions/PCHiC/Annotation/UCE_Fragments.txt", header=F)
FilteredInteractions <- FilteredInteractions[ !(FilteredInteractions$baitID %in% UCE[,1]),]

# Remove non-baited fragments that contain any type of gene
Genes <- read.delim("Data/Interactions/PCHiC/Annotation/Genes_Fragments.txt", header=T)
FilteredInteractions <- FilteredInteractions[ !(FilteredInteractions$oeID %in% Genes$Fragment),] 

# Remove baited fragments that does not contain a protein-coding gene
Genes <- Genes[ Genes$Type == "protein_coding",]
FilteredInteractions <- FilteredInteractions[ FilteredInteractions$baitID %in% Genes$Fragment,] 

# Normalize the read counts using edgeR
rownames(FilteredInteractions) <- FilteredInteractions$ID
D <- DGEList(as.matrix(FilteredInteractions[,c("Zero_R","Four_R","Two_R")]))
D <- calcNormFactors(D)
Normalized_Counts <- cpm(D, normalized.lib.sizes = T, log = T)
Normalized_Counts <- 2^Normalized_Counts
Normalized_Counts <- as.data.frame(Normalized_Counts)
colnames(Normalized_Counts) <- c("Zero_Norm","Four_Norm","Two_Norm")
Normalized_Counts$ID <- rownames(Normalized_Counts)
FilteredInteractions <- merge(FilteredInteractions, Normalized_Counts, by="ID")

# Calculate log2 fold changes between all samples using an additional pseudo-count of 0.25
FilteredInteractions$logFC_Four_Zero <- log2((FilteredInteractions$Four_Norm + 0.25) / (FilteredInteractions$Zero_Norm + 0.25))
FilteredInteractions$logFC_Two_Zero <- log2((FilteredInteractions$Two_Norm + 0.25) / (FilteredInteractions$Zero_Norm + 0.25))
FilteredInteractions$logFC_Two_Four <- log2((FilteredInteractions$Two_Norm + 0.25) / (FilteredInteractions$Four_Norm + 0.25))

# Split the interactions into three groups depending on the maximal absolute log2 fold change
Group1 <- FilteredInteractions[ abs(FilteredInteractions$logFC_Four_Zero) >= abs(FilteredInteractions$logFC_Two_Zero) & abs(FilteredInteractions$logFC_Four_Zero) >= abs(FilteredInteractions$logFC_Two_Four), ]
Group2 <- FilteredInteractions[ abs(FilteredInteractions$logFC_Two_Zero) >= abs(FilteredInteractions$logFC_Four_Zero) & abs(FilteredInteractions$logFC_Two_Zero) >= abs(FilteredInteractions$logFC_Two_Four), ]
Group3 <- FilteredInteractions[ abs(FilteredInteractions$logFC_Two_Four) >= abs(FilteredInteractions$logFC_Two_Zero) & abs(FilteredInteractions$logFC_Two_Four) >= abs(FilteredInteractions$logFC_Four_Zero), ]

# Set the true log2 fold change into new column for each group with different maxima
Group1$logFC_max <- Group1$logFC_Four_Zero
Group2$logFC_max <- Group2$logFC_Two_Zero
Group3$logFC_max <- Group3$logFC_Two_Four

# Combine the 3 groups into a new data.frame
Combined <- rbind(Group1, Group2, Group3)

# Select one maximal fold change for entries with duplicated maximal absolute log2 fold changes
set.seed(9999)
Combined$Random <- sample(c(1:nrow(Combined)), nrow(Combined), replace = T)
Combined <- Combined[ order(Combined$ID, Combined$Random),]
Combined <- Combined[ duplicated(Combined$ID)==F,]

# Merge maximal log2 fold change and the remaining data
FilteredInteractions <- merge(FilteredInteractions, Combined[,c("ID","logFC_max")], by="ID", all.x=T)

## Process histone mark data
# Import the data
Enhancers <- read.delim("Data/ChIPseq/Counts/Histone_Marks.count")

# Set column names and keep only columns of interest
Enhancers <- Enhancers[,c(1:4,8:20)]
colnames(Enhancers) <- c("PeakID","Chr","Start","End","H3K27ac_H4_E1","H3K27ac_H4_E2","H3K27ac_D0_E1","H3K27ac_D0_E2","H3K4me1_H4_E1","H3K4me1_H4_E2","H3K4me1_D0_E1","H3K4me1_D0_E2","H3K4me2_H4_E1","H3K4me2_H4_E2","H3K4me2_D0_E1","H3K4me2_D0_E2","Input")

# Average the replicates and log transform the data
Enhancers$H3K27ac_D0 <- (rowMeans(Enhancers[,c(7,8)]))
Enhancers$H3K27ac_H4 <- (rowMeans(Enhancers[,c(5,6)]))
Enhancers$H3K4me1_D0 <- (rowMeans(Enhancers[,c(11,12)]))
Enhancers$H3K4me1_H4 <- (rowMeans(Enhancers[,c(9,10)]))
Enhancers$H3K4me2_D0 <- (rowMeans(Enhancers[,c(15,16)]))
Enhancers$H3K4me2_H4 <- (rowMeans(Enhancers[,c(13,14)]))
Enhancers$Input <- (Enhancers$Input)
for (i in 17:23) { Enhancers[,i] <- log2(Enhancers[,i]) }
Enhancers$ID <- paste(Enhancers[,2],Enhancers[,3],Enhancers[,4],sep="-")

# Define max and filter
Enhancers$Max <- apply(Enhancers[,c(20:23)],1,FUN="max")
Enhancers <- Enhancers[ Enhancers$Max >= log2(30),]

Enhancers$Min <- apply(Enhancers[,c(20:23)],1,FUN="min")
Enhancers <- Enhancers[ Enhancers$Min <= log2(250),]

# Group the sites
Enhancers$Group <- "Undefined"
Enhancers[ Enhancers$H3K27ac_H4 - Enhancers$H3K27ac_D0 >= log2(1.5) & Enhancers$H3K27ac_H4 >= log2(30),"Group"] <- "Induced"
Enhancers[ Enhancers$H3K27ac_H4 - Enhancers$H3K27ac_D0 <= -log2(1.5) & Enhancers$H3K27ac_D0 >= log2(30),"Group"] <- "Repressed"
Enhancers[ abs(Enhancers$H3K27ac_H4 - Enhancers$H3K27ac_D0) < log2(1.5) & (Enhancers$H3K27ac_D0 >= log2(30) | Enhancers$H3K27ac_H4 >= log2(30)),"Group"] <- "Constitutive"
Enhancers[ (Enhancers$H3K27ac_D0 <= log2(30) & Enhancers$H3K27ac_H4 <= log2(30)),"Group"] <- "Absent"

## Overlap histone mark data and interactions
# Make a GRanges object from otherEnds
otherEnds <- FilteredInteractions[ duplicated(FilteredInteractions$oeID)==F,]
otherEndRanges <- GRanges(seqnames = otherEnds$oeChr, IRanges(start = otherEnds$oeStart, end = otherEnds$oeEnd), strand = rep("+",nrow(otherEnds)), mcols = otherEnds$oeID)

# Make a GRanges object from the enhancers
Enhancers$Start <- Enhancers$Start + 1900 
Enhancers$End <- Enhancers$End - 1900 
EnhancerRanges <- GRanges(seqnames = Enhancers$Chr, IRanges(start = Enhancers$Start, end = Enhancers$End), strand = rep("+",nrow(Enhancers)), mcols = Enhancers$PeakID)

# Get overlap and width of overlap, merge into a single data.frame and collaps
Overlap <- as.data.frame(suppressWarnings(findOverlaps(otherEndRanges, EnhancerRanges)))
Overlap <- cbind(as.data.frame(otherEndRanges[Overlap[,1]]),as.data.frame(EnhancerRanges[Overlap[,2]]))

# Keep only sites overlapping an interactions
Enhancers <- Enhancers[ Enhancers$PeakID %in% Overlap[,12],]

# Plot it
par(mfcol=c(1,4))
boxplot(Enhancers[ Enhancers$Group == "Induced", c(17,20:23,18,19)], outline=F, col=c("grey","blue3","red3","blue3","red3","blue3","red3"), ylim=c(0,12), las=2)
abline(v=c(1.5,3.5,5.5), lty=2)
boxplot(Enhancers[ Enhancers$Group == "Constitutive", c(17,20:23,18,19)], outline=F, col=c("grey","blue3","red3","blue3","red3","blue3","red3"), ylim=c(0,12), las=2)
abline(v=c(1.5,3.5,5.5), lty=2)
boxplot(Enhancers[ Enhancers$Group == "Repressed", c(17,20:23,18,19)], outline=F, col=c("grey","blue3","red3","blue3","red3","blue3","red3"), ylim=c(0,12), las=2)
abline(v=c(1.5,3.5,5.5), lty=2)
boxplot(Enhancers[ Enhancers$Group == "Absent", c(17,20:23,18,19)], outline=F, col=c("grey","blue3","red3","blue3","red3","blue3","red3"), ylim=c(0,12), las=2)
abline(v=c(1.5,3.5,5.5), lty=2)
```

[Back to start](../README.md)<br>
[Back to overview of Figure S4](../Links/FigureS4.md)
