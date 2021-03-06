## Introduction

In this workflow, network diffusion analysis is performed to find
affected pathways during pancreatic cancer. In this workflow, a
gene-pathway association network will be created by using up and down
regulated genes of NanoString dataset (Folfirinox treatment) and the
pathway collection from WikiPathways.Then network diffusion algorithm
will be applied on the obtained gene-pathway association network to find
affected pathways during the disease.

The PancCanNet project is a collaboration between the Erasmus MC,
Maastricht University and Omnigen. The pathway portal is integrated in
the collaborative pathway database WikiPathways
(<http://panccannet.wikipathways.org>) and all analysis workflows are
available on Github (<https://panccannet.github.io>).

Pathway curation is an ongoing effort and we are continuously improving
current pathway models and adding new pathway models. The workflow will
always use the most-up-to-date information and results can slightly
differ to the information on the project website.

## R environment setup

First, we need to make sure all required R-packages are installed. Run
the following code snippet to install and load the libraries.

``` r
# check if libraries are installed > otherwise install
if (!requireNamespace("BiocManager", quietly = TRUE)) install.packages("BiocManager")
if(!"rstudioapi" %in% installed.packages()) BiocManager::install("rstudioapi")
if(!"rWikiPathways" %in% installed.packages()) BiocManager::install
if(!"clusterProfiler" %in% installed.packages()) BiocManager::install
if(!"org.Hs.eg.db" %in% installed.packages()) BiocManager::install
if(!"dplyr" %in% installed.packages()) BiocManager::install
if(!"igraph" %in% installed.packages()) BiocManager::install
if(!"RCy3" %in% installed.packages()) BiocManager::install

#load requried libraries
library(rstudioapi)
library(rWikiPathways)
library(clusterProfiler)
library(org.Hs.eg.db)
library(dplyr)
library(igraph)
library(RCy3)
library(diffusr)
library(data.table)
library(ggplot2)

# set working environment to source file
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
```

## Load differential expressed genes

In this section we will import differentially expressed gene list from a
file

``` r
#import the dataset from the file under "data" folder
dataset <- read.csv("data/DEresults-TreatmentFOLFIRINOX.csv", header=TRUE)

#change column name 
colnames(dataset)[1] <- "GeneName"
#filter out token "-mRNA" from each gene name 
dataset$GeneName <- gsub('-mRNA', '', dataset$GeneName)
#get Entrez Gene identifier for each gene symbols 
hgcn2entrez <- clusterProfiler::bitr(dataset$GeneName, fromType = "SYMBOL",
                                     toType = c("ENTREZID","SYMBOL"), 
                                     OrgDb = org.Hs.eg.db)

#add entrez ID to the each gene symbol in the dataset
data.mapped <- merge(dataset, hgcn2entrez, by.x="GeneName", by.y="SYMBOL", all.x = TRUE)

# filter genes without Entrez Gene identifier
data <- data.mapped %>% tidyr::drop_na(ENTREZID)
#get differentially expressed genes using the selection criteria 
deg <- unique(data[ (data$P.value < 0.05 & data$Log2.fold.change > 0.58) | 
                    (data$P.value < 0.05 & data$Log2.fold.change < -0.58), ])

#select only used columns
deg <- subset(deg, select = c(1,14) )
#change column names
colnames(deg)<- c("GENE_SYMBOL","ENTREZ_ID")
#write DEG list to the file
write.table(deg,"DEG_list",row.names=FALSE,col.names = FALSE,quote = FALSE)
```

## Gene-Pathway Association Network

In this section, we will create a gene-pathway association graph using
the imported gene list and the pathway collection from WikiPathways.

``` r
#download pathway archive with the format .gmt
options(RCurlOptions = list(cainfo = paste0( tempdir() , "/cacert.pem" ), ssl.verifypeer = FALSE))
wp.hs.gmt <- rWikiPathways::downloadPathwayArchive(date="current", organism="Homo sapiens", format = "gmt")

#get the information 
wp2gene <- readPathwayGMT(wp.hs.gmt)
#select only wpid and gene name columns
wpid2gene <- wp2gene %>% dplyr::select(wpid,gene) #TERM2GENE
#select only wpid and wp name columns
wpid2name <- wp2gene %>% dplyr::select(wpid,name) #TERM2NAME

#gene-pathway association graph (undirected) 
graph <- igraph::graph_from_edgelist(as.matrix(wpid2gene),directed = FALSE)
```

## Network Diffusion Algorithm

In the following section, a network diffusion algortihm will be applied
on the obtained gene-pathway association graph to find affected pathways
during the disease. Network diffusion algorithms such as heat diffusion
or Markov random walks spread the information in the form of node
weights from a node to another node in a graph. the node weights can
represented as a temperature or initial amount of water.The spreading of
information continue till a stop criterion occurs.

``` r
#to be used in the algorithm the graph should be first converted to an adjacency matrix
adj.m <- as.matrix(igraph::as_adjacency_matrix(graph, type="both"))
# starting distribution
#create a binary vector for hot genes with the same dimension of adj matrix
h0 <- rep(0,nrow(adj.m))
#add (adj.m) names to the vector for next step
names(h0) <- rownames(adj.m)
#set hot genes as 1, others as 0
#note: not all DEG is one of the genes in h0, after performing this step 50 genes are assigned to 1 in h0 vector.
h0[names(h0) %in% deg$ENTREZ_ID] <- 1
h0.dataframe <- as.data.frame(h0)
write.table(h0,"h0",quote = FALSE, row.names = TRUE,sep = "\t")

# computation of stationary distribution
ht <- as.data.frame(heat.diffusion(h0, adj.m))

#add rownames to the generated ht vector
rownames(ht) <- rownames(adj.m)

ht <- setDT(ht, keep.rownames = TRUE)[]
colnames(ht) <- c("node","heat")
#write generated heats to the file
write.table(ht,"heats", quote= FALSE, sep = '\t', row.names = FALSE)
```

## Visualization of heat result

``` r
#visualize only nodes that have a certain cutoff heat
#plot the dist see the dist normal? select cutoff based on the dist.

## Basic histogram from the vector "ht$V1". Each bin is .5 wide.
ggplot(ht, aes(x=heat)) + geom_histogram(binwidth=.05)
```

![](PancCanNet-workflow5_files/figure-markdown_github/visualization-1.png)

``` r
# Draw with black outline, white fill
ggplot(ht, aes(x=heat)) +geom_histogram(binwidth=.05, colour="black", fill="white")
```

![](PancCanNet-workflow5_files/figure-markdown_github/visualization-2.png)

``` r
# Density curve
ggplot(ht, aes(x=heat)) + geom_density() + xlim(-0.1, 1)
```

![](PancCanNet-workflow5_files/figure-markdown_github/visualization-3.png)

``` r
# Histogram overlaid with kernel density curve
ggplot(ht, aes(x=heat)) + 
    geom_histogram(aes(y=..density..),      # Histogram with density instead of count on y-axis
                   binwidth=.5,
                   colour="black", fill="white") +
    geom_density(alpha=.2, fill="#FF6666")  # Overlay with transparent density plot
```

![](PancCanNet-workflow5_files/figure-markdown_github/visualization-4.png)

## Graph Visualization

``` r
#selected threshold is 0.0511, which is approximately calculated by the mean of 
#all heat values plus the standard deviation of data.
threshold = 0.051

#select only nodes (wp and genes) that has heat greater than the threshold
ht <- ht [ht$heat>threshold,]

#scenario-1: select only interactions between selected nodes  (heat>threshold)
wpid2gene.2 <- wpid2gene[(wpid2gene$wpid %in% ht$node)&(wpid2gene$gene %in% ht$node),]
#scenario-2: select only interactions that contains at least one selected node (heat>threshold)
wpid2gene.2 <- wpid2gene[(wpid2gene$wpid %in% ht$node)|(wpid2gene$gene %in% ht$node),]

#regenerate graph with the new wpid2gene matrix
graph <- graph_from_edgelist(as.matrix(wpid2gene.2),directed = FALSE)

#check that cytoscape is connected
cytoscapePing()

# create igraph if not yet available
net <- RCy3::createNetworkFromIgraph(graph)
RCy3::loadTableData(data.key.column = "node", data = ht, table = "node",table.key.column = "name")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
RCy3::setNodeColorMapping("heat", c(0,1), c("#FFFFFF","#FF0000"), default.color = "#FFFACD")
```

    ## NULL
