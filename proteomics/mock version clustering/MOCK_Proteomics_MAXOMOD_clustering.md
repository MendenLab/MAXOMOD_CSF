---
title: "MOCK Proteomics MAXOMOD Clustering "
author: "Clara Meijs"
date: "2023-08-03"
output:
  html_document:
    df_print: paged
    keep_md: yes
    toc: true
    toc_float: true
    toc_collapsed: true
    toc_depth: 5
    theme: lumen
---

In this RMarkdown file, we will perform the clustering analysis on proteomics data. I generated fake data so that this file can be shared freely without privacy or confidentiality issues. 

## Libraries

Start with clearing environment and loading packages


```{.r .fold-hide}
set.seed(9)
rm(list=ls())
library(ggthemes)
library(pheatmap)
library(ggplot2)
library(matrixStats)
library(wesanderson)
library(clusterProfiler)
library(enrichplot)
library(msigdbr)
library(dichromat)
library(stringr)
library(dplyr)
library(ggrepel)
library(reshape2)
library(umap)
library(ggthemes)
library(cowplot)
library(DEP)
```

```
## Warning in fun(libname, pkgname): mzR has been built against a different Rcpp version (1.0.10)
## than is installed on your system (1.0.11). This might lead to errors
## when loading mzR. If you encounter such issues, please send a report,
## including the output of sessionInfo() to the Bioc support forum at 
## https://support.bioconductor.org/. For details see also
## https://github.com/sneumann/mzR/wiki/mzR-Rcpp-compiler-linker-issue.
```

```{.r .fold-hide}
library(naniar)
library(SummarizedExperiment)
library(data.table)
library(readr)
library(ggpubr)
library(RColorBrewer)
```

## Set working directories


```{.r .fold-hide}
# if you are using Rstudio run the following command, otherwise, set the working directory to the folder where this script is in
setwd(dirname(rstudioapi::getActiveDocumentContext()$path))

# create directory for results
dir.create(file.path(getwd(),'results/'), showWarnings = FALSE)
# create directory for plots
dir.create(file.path(getwd(),'plots/'), showWarnings = FALSE)
# create directory for final plots used for the paper
dir.create(file.path(getwd(),'plots/paper'), showWarnings = FALSE)
# create directory for data
dir.create(file.path(getwd(),'data/'), showWarnings = FALSE)
```

## Generate fake data

I make mock data with a pattern for three subgroups.


```r
#GENERATING FAKE ABUNDANCY DATA
    set.seed(9)
    
    #take the protein names from the actual dataset
    protein_names = readRDS(file = "data/protein_names.rds")
      
    # Number of proteins and patients
    num_proteins <- length(protein_names)
    num_patients <- 50
    
    # Generate fake abundancy data
    abundance_data <- matrix(runif(num_proteins * num_patients, min = 0, max = 1000), nrow = num_proteins, ncol = num_patients)
    
    # Adding some noise
    abundance_data <- abundance_data + rnorm(num_proteins * num_patients, mean = 0, sd = 0.5)
    
    #make into dataframe and give normal name
    abundance_data = as.data.frame(abundance_data)
    
    #colnames proteins and patients
    rownames(abundance_data) = protein_names
    colnames(abundance_data) = paste0("patient_", 1:num_patients)
    
    # Define cluster 1 and cluster 2 proteins
    #these proteins are associated with the same pathways, so we can get realistic pathway analysis
    #BP
    cluster1_proteins <- c("F2", "PLG", "KNG1", "VTN", "FGG", 
                           "FGB", "APOH", "SERPINF2", "CPB2", "FGA", "PROS1", "KLKB1", "HRG")
    cluster2_proteins <- c("NLGN1", "CHGA", "YWHAZ", "SLIT1")
    cluster3_proteins = c("LRRC4B", "CBLN1", "NRCAM", "NRXN2", "SPOCK2", "EFNB2", "PLXNB1", 
                          "LAMB2", "PTPRD", "SHISA6", "PRNP", "PTPRF", "CLSTN2", "APP", 
                          "NPTX1", "PCDH8", "CBLN4", "CBLN3", "NLGN1", "L1CAM", "YWHAZ", 
                          "HSPA8", "CDH2", "NRXN1", "CDH8", "NEGR1", "SPARCL1", "GAP43", 
                          "SLIT1", "PTPRS")

    #CC
    cluster1_proteins = c(cluster1_proteins,
                          "NTRK2", "CANX", "CNTN6", "LRRC4B", "CNTNAP4", "NRXN2", "EFNB2", 
                          "PTPRD", "PRNP", "APP", "PCDH8", "PARK7", "PTPRN2", "NLGN1", 
                          "HSPA8", "CDH2", "NRXN1", "CDH8", "YWHAG", "CNTN1", "PTPRN", 
                          "PTPRS")

    cluster2_proteins = c(cluster2_proteins, 
                          "IGHG4", "IGHG3", "IGKC", "JCHAIN", "IGHM", "IGHG2", "IGHA1", "IGLC2",
                          "F2", "PLG", "AMBP", "KNG1", "VTN", "ORM1", "ITIH2", "ORM2", "A1BG", "HPX", 
                          "FGG", "FGB", "ITIH4", "ITIH1", "APOH", "ATRN", "AHSG", "SERPINF2", "CPN2",
                          "SERPINA5", "SEMA3B", "SERPINC1", "FGA", "ANG", "IGFALS", "FBLN5", "F9", 
                          "HRG", "SPARC", "PSAP", "APOA1", "SOD3", "PCOLCE", "FBN1", "APOC3", "TIMP1",
                          "EFEMP1", "THBS4", "LTBP4", "SERPINA1", "F12", "EFEMP2", "APOA4", "LUM", 
                          "CLEC3B", "A2M", "LAMC1", "MGP", "LGALS3BP", "FCGBP", "CTSD")
    
    cluster3_proteins = c(cluster3_proteins, 
                          "LRRC4B", "NRCAM", "EFNB2", "SHISA6", "CLSTN2", "ADAM22", "DCC", "PTPRS")
    
    #MF
    cluster1_proteins = c(cluster1_proteins, 
                          "F2", "APOL1", "APOM", "GC", "C8G", "AFM", "APOH", "RBP4", "SERPINA5", 
                          "PON1", "SERPINA6", "APOA2", "CD81", "APOD", "PSAP", "APOA1", 
                          "JCHAIN", "APOC3", "IGHM", "LBP", "APOB", "APOA4", "APOC2", "AMBP" )
    cluster2_proteins = c(cluster2_proteins, "PLG", "IGHG4", "IGHG3", "IGKC", "JCHAIN", "IGHM")
    cluster3_proteins = c(cluster3_proteins, "ACTG1", "CALR", "YWHAE", "UBA52", "GPR37", "TPI1", 
                          "YWHAZ", "HSPA8", "PTPRN")
    
    #add random proteins to the proteins and make unique
    cluster1_proteins = unique(c(cluster1_proteins, sample(size = 30, x = protein_names)))
    cluster2_proteins = unique(c(cluster2_proteins, sample(size = 30, x = protein_names)))
    cluster3_proteins = unique(c(cluster3_proteins, sample(size = 30, x = protein_names)))
    
    # Create random subgroup indicators for patients
    cluster_index <- sample(1:3, num_patients, replace = TRUE)
    
    # Set random fold changes for subgroup 1 proteins
    fold_changes_cluster1 <- runif(length(cluster1_proteins), min = 2, max = 4)  # Adjust fold changes as needed
    
    # Set random fold changes for subgroup 2 proteins
    fold_changes_cluster2 <- runif(length(cluster2_proteins), min = 2, max = 4)  # Adjust fold changes as needed
    
    # Set random fold changes for subgroup 2 proteins
    fold_changes_cluster3 <- runif(length(cluster3_proteins), min = 2, max = 4)  # Adjust fold changes as needed
    
    # Apply fold changes to subgroup 1 proteins
    for (protein_name in cluster1_proteins) {
      protein_index <- which(protein_names == protein_name)
      abundance_data[protein_index, cluster_index == 1] <- abundance_data[protein_index, cluster_index == 1] * fold_changes_cluster1[match(protein_name, cluster1_proteins)]
    }
    
    # Apply fold changes to subgroup 2 proteins
    for (protein_name in cluster2_proteins) {
      protein_index <- which(protein_names == protein_name)
      abundance_data[protein_index, cluster_index == 2] <- abundance_data[protein_index, cluster_index == 2] * fold_changes_cluster2[match(protein_name, cluster2_proteins)]
    }
    
        # Apply fold changes to subgroup 2 proteins
    for (protein_name in cluster3_proteins) {
      protein_index <- which(protein_names == protein_name)
      abundance_data[protein_index, cluster_index == 3] <- abundance_data[protein_index, cluster_index == 3] * fold_changes_cluster3[match(protein_name, cluster3_proteins)]
    }
    
    # Adding more noise
    abundance_data <- abundance_data + rnorm(num_proteins * num_patients, mean = 0, sd = 2)
    
    # Change negative values to positive
    abundance_data = abs(abundance_data)
    
    assay = as.data.frame(t(abundance_data))
    
#GENERATING FAKE CLINICAL DATA
    
    clin = data.frame(
      patid = paste0("patient_", 1:num_patients),
      disease = as.factor(rep("als", num_patients)),
      sex = as.factor(sample(c("Male", "Female"), num_patients, replace = TRUE)),
      age = runif(num_patients, min = 40, max = 80), 
      neurofilaments = runif(num_patients, min = 0, max = 1000),
      genetics = as.factor(sample(c("negative", "not_performed", "C9orf72"), num_patients, replace = TRUE)), 
      onset = as.factor(sample(c("spinal", "bulbar"), num_patients, replace = TRUE)), 
      age_at_onset = runif(num_patients, min = 40, max = 80), 
      progression_rate = runif(num_patients, min = 0, max = 8), 
      slow_vital_capacity = runif(num_patients, min = 0, max = 10), 
      pNFh = runif(num_patients, min = 0, max = 1000),
      center = as.factor(sample(c("munich", "goettingen"), num_patients, replace = TRUE))
    )
    
    #create extra categorical variable for age based on median
      m = median(clin$age)
      clin$age_cat = rep(NA, length(clin$age))
      clin$age_cat[clin$age>=m] = "over_59"
      clin$age_cat[clin$age<m] = "under_59"
      clin$age_cat = as.factor(clin$age_cat)
      
      
    #make patient ids to rownames
      rownames(clin) = clin$patid
      
saveRDS(list(abundancy = abundance_data,
             clinical_data = clin),
        file = "data/mock_data.rds")
```

## Define colour palettes


```r
#all colourblind friendly options
 display.brewer.all(n=NULL, type="all", select=NULL, exact.n=TRUE, 
                    colorblindFriendly=TRUE)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/define colour palettes using Rcolorbrewer-1.png)<!-- -->

```r
#create a list with all colours for the final paper figures
#each list element will be a named vector
final_colours = list(
  #red and blue for volcano plot
  volcano_plot = brewer.pal(n=10, "RdYlBu")[c(2,9)],
  male_female = brewer.pal(n=8, "Set2")[c(1,4)],
  #heatmap_scale = rev(brewer.pal(n = 11, "RdYlBu")),
  heatmap_scale = rev(brewer.pal(n=11, "RdBu")),
  clustering = brewer.pal(n=8, "Set2")[c(2,3,5)],
  age_scale = brewer.pal(n = 9, "YlGn"),
  disease_progression_scale = brewer.pal(n = 9, "YlOrRd"),
  onset = c(brewer.pal(n=11, "RdYlBu")[c(4,5)], "#B3B3B3"),
  disease_status = c(brewer.pal(n=11, "RdYlBu")[2],"#B3B3B3"),
  genetic_testing = c("#B3B3B3", brewer.pal(n = 11, "PRGn")[c(3, 9)]),
  center = c("purple4", "orange3"),
  neurofilaments = brewer.pal(n = 9, "PuBu"),
  pNFh_scale = brewer.pal(n = 9, "Purples"),
  age_at_onset_scale = brewer.pal(n = 9, "Blues"),
  slow_vital_capacity_scale = brewer.pal(n = 9, "Reds")
)

#give the vectors with discrete variables names
names(final_colours$volcano_plot) =  c("up", "down")
names(final_colours$male_female) = c("Male", "Female")
names(final_colours$clustering) = c("alpha", "beta", "theta")
names(final_colours$disease_status) = c("als", "ctrl")
names(final_colours$onset) = c("spinal", "bulbar", "ctrl")
names(final_colours$genetic_testing) = c("not_performed", "negative", "C9orf72")
names(final_colours$center) = c("goettingen", "munich")
```

## Perform different types of clustering


```r
set.seed(9)

      library(cluster)
      
      #make empty matrix to save cluster assignments
      cluster_assignments = as.data.frame(clin$patid)
      colnames(cluster_assignments) = "patid" #now it is still only one column, which contains the patient id's
      silhouette_scores = TWSS_scores = AIC_scores = BIC_scores = as.data.frame(matrix()) #create empty dataframes for the different scores
      rownames(silhouette_scores) = rownames(TWSS_scores) = rownames(AIC_scores) = rownames(BIC_scores) = "hclust" #they only have one row and first row will be used for hierarchical clustering 

#function to calculate total within sum of squares
      calc_SS = function(df) sum(as.matrix(dist(df)^2)) / (2 * nrow(df))
      calc_TWSS = function(df, clusters){
        number_clusters = length(levels(as.factor(clusters)))
        sum_of_squares = c()
        for(i in 1:number_clusters){sum_of_squares[i] = calc_SS(df[clusters == i,])}
        return(sum(sum_of_squares))
      }

#function to calculate BIC and AIC
      BIC2 <- function(df, clusters){
      m = ncol(df)
      n = nrow(df)
      k = length(levels(as.factor(clusters)))
      D = calc_TWSS(df, clusters)
      return(data.frame(AIC = D + 2*m*k,
                        BIC = D + log(n)*m*k))
      }

      
#PERFORM ALL THE CLUSTERING      
      
      #perform the clustering with trying cluster numbers 2-10
      for(i in 2:10){

# Hierarchical Clustering:
    set.seed(9)
  #with hclust function
    # hclust: Performs hierarchical clustering.
    title = paste0("hclust_k=", i)
    #performing the clustering
    dist_mat <- dist(assay, method = 'euclidean')
    cl <- hclust(dist_mat, method = 'ward.D')
    cluster_assignments[,title] <- cutree(cl, k = i)
    
    
    #cluster fit measures
    ss = silhouette(cluster_assignments[,title], dist(assay))
    silhouette_scores["hclust", i] = mean(ss[, 3])
    TWSS_scores["hclust", i] = calc_TWSS(assay, cluster_assignments[,title])
    AIC_scores["hclust", i] = BIC2(assay, cluster_assignments[,title])[1,1]
    BIC_scores["hclust", i] = BIC2(assay, cluster_assignments[,title])[1,2]

# Model-Based Clustering:
# Mclust: Fits Gaussian finite mixture models for model-based clustering.
    set.seed(9)
    title = paste0("mclust_k=", i)
    
    #performing the clustering
    library(mclust)
    cl <- Mclust(assay, G = i)
    cluster_assignments[,title] <- cl$classification

    #cluster fit measures
    ss = silhouette(cluster_assignments[,title], dist(assay))
    silhouette_scores["mclust", i] = mean(ss[, 3])
    TWSS_scores["mclust", i] = calc_TWSS(assay, cluster_assignments[,title])
    AIC_scores["mclust", i] = BIC2(assay, cluster_assignments[,title])[1,1]
    BIC_scores["mclust", i] = BIC2(assay, cluster_assignments[,title])[1,2]    
    
# K-Means Clustering:
    set.seed(9)
    # kmeans: Performs k-means clustering.
    title = paste0("kmeans_k=", i)
    
    #performing the clustering
    cl <- kmeans(assay, centers = i)
    cluster_assignments[,title] = cl$cluster
    
    #cluster fit measures
    ss = silhouette(cluster_assignments[,title], dist(assay))
    silhouette_scores["kmeans", i] = mean(ss[, 3])
    TWSS_scores["kmeans", i] = calc_TWSS(assay, cluster_assignments[,title])
    AIC_scores["kmeans", i] = BIC2(assay, cluster_assignments[,title])[1,1]
    BIC_scores["kmeans", i] = BIC2(assay, cluster_assignments[,title])[1,2]

# Partitioning Around Medoids:
# pam: Performs Partitioning Around Medoids (PAM) clustering.
    set.seed(9)
    title = paste0("pam_k=", i)
    
    #performing the clustering
    cl <- pam(assay, k = i)
    cluster_assignments[,title] <- cl$clustering

    #cluster fit measures
    ss = silhouette(cluster_assignments[,title], dist(assay))
    silhouette_scores["pam", i] = mean(ss[, 3])
    TWSS_scores["pam", i] = calc_TWSS(assay, cluster_assignments[,title])
    AIC_scores["pam", i] = BIC2(assay, cluster_assignments[,title])[1,1]
    BIC_scores["pam", i] = BIC2(assay, cluster_assignments[,title])[1,2]
      }
```

```
## Package 'mclust' version 6.0.0
## Type 'citation("mclust")' for citing this R package in publications.
```

```r
      cluster_assignments[,2:ncol(cluster_assignments)] = cluster_assignments[,2:ncol(cluster_assignments)]-1

#clustering function for later use
      
      #this function is a more concise version of what we've done above, because we don't need all the performance measures all the time
      #this function only returns cluster assignments

      perform_clustering <- function(assay_data, seed, clin_labels) {
        set.seed(seed)
        
        cluster_results <- list()
        cluster_assignments <- as.data.frame(clin_labels)
        colnames(cluster_assignments) <- "patid"
        
        cluster_methods <- c("hclust", "mclust", "kmeans", "pam")
        
          for (i in 2:10) {
            
            # Perform clustering
              #hierarchical clustering
              title = paste0("hclust_k=", i)
              dist_mat <- dist(assay_data, method = 'euclidean')
              cl <- hclust(dist_mat, method = 'ward.D')
              cluster_assignments[, title] <- cutree(cl, k = i)
              #model based clustering
              title = paste0("mclust_k=", i)
              library(mclust)
              cl <- Mclust(assay_data, G = i)
              cluster_assignments[, title] <- cl$classification
              #kmeans clustering
              title = paste0("kmeans_k=", i)
              cl <- kmeans(assay_data, centers = i)
              cluster_assignments[, title] <- cl$cluster
              #partitioning around medoids
              title = paste0("pam_k=", i)
              cl <- pam(assay_data, k = i)
              cluster_assignments[, title] <- cl$clustering
            }
        
        cluster_assignments[, 2:ncol(cluster_assignments)] <- cluster_assignments[, 2:ncol(cluster_assignments)] - 1
        return(cluster_assignments)
      }      
```

## Clusterprofiler GSEA function


```r
library(clusterProfiler)
library(enrichplot)
library(ggplot2)
library(msigdbr)
library(dichromat)
library(stringr)
redblue<-colorRampPalette(c("red","blue"))

#function for performing GSEA with clusterprofiler package and creating the corresponding plots
clusterprofiler_gsea = function(data, ont, title, alpha = 0.05){
                  
      #perform gsea
          dg = sort(data, decreasing = TRUE)  #sort proteins on decreasing log-fold change (required for gsea)
        
      #create background according to the ontology used for the analysis    
          bg <- msigdbr(species = "Homo sapiens", category = "C5", subcategory = ont) %>% 
                dplyr::select(gs_name, gene_symbol)
          bg <- bg[bg$gene_symbol %in% names(dg), ]
        
        #the gsea analysis with cut-off   
        gse2 = GSEA(geneList=dg, #performing the gsea
             nPermSimple = 100000, 
             minGSSize = 3, #minimum gene set size
             maxGSSize = 800, #maximum gene set size
             pvalueCutoff = alpha, 
             verbose = TRUE, 
             TERM2GENE = bg, #background
             pAdjustMethod = "BH", #benjamini hochberg correction
             eps = 1e-5, 
             seed = 9)
        
        if(nrow(gse2@result)<2){
          plot = "no_significant_results"
          cnetplot = "no_significant_results"
          results = gse2@result
        }else{
        results = gse2@result
        x2 <- pairwise_termsim(gse2) 
        plot = emapplot(x2, showCategory = 800, cex.params = list(category_label = 0.5))  +
            ggtitle(paste0("fgsea gene ontology with \n", title))
        cnetplot = cnetplot(gse2, node_label = 'all',
                            cex.params = list(category_label = 0.5), showCategory = 1500, 
                            color.params = list(foldChange = dg))  +
            ggtitle(paste0("fgsea gene ontology with \n",title))
        }
        
      return(list(plot = plot, cnetplot = cnetplot, results = results)) #function returns emapplot, cnetplot, and result table

          }
```

## Performing DEx and GSEA analysis


```r
#settings for upcoming analyses
      cluster_assignments <- perform_clustering(assay, 9, clin_labels = clin$patid) #use clustering function to get cluster labels
      cluster_assignments = cluster_assignments[,1:9] #take only k=2 and k=3
      cluster_levels_k2 = c("alpha", "beta") #rename levels
      cluster_levels_k3 =  c("beta", "theta", "alpha")  #first is beta, second is theta, third is alpha
      set.seed(9)
      res = list() #create empty results list
      l = u = 1
      covariates = "age_sex_cov" #we only want to make a model with age and sex covariates
      covariates_f = ~0 + condition + age_cat + sex #the function to model these covariates correctly
      control = "beta" #set beta as the control, which sets the direction of the clustering comparison
      ontologies = c("BP", "CC", "MF")
      plots = list()
      cnetplots = list()
      results = list()
      alpha = 0.1 #FDR cut-off
      
      cluster_assignments_2 = cluster_assignments[,c("patid", "kmeans_k=2", "kmeans_k=3")] #select only columns with patid and kmeans labels

for(k in 2:ncol(cluster_assignments_2)){ #loop to perform the analyses for kmeans k=2 and k=3
        set.seed(9)
        title = colnames(cluster_assignments_2)[k]

#make summarized experiment, this time with cluster as condition
        #we need the summarized experiment data format to be able to use the DEx function from the DEP package
        clin$cluster = as.factor(cluster_assignments_2[,k])
        if(k == 2){levels(clin$cluster) = cluster_levels_k2}
        if(k == 3){levels(clin$cluster) = cluster_levels_k3}
        assay2 = as.data.frame(t(assay))
        assay2$ID = assay2$name = rownames(assay2)
        abundance.columns <- grep("patient", colnames(assay2)) # get abundance column numbers
        experimental.design = clin[, c("patid","cluster", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")]
        colnames(experimental.design) = c("label","condition", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")
        experimental.design$replicate = 1:nrow(experimental.design)
        se_abu_data_ALS <- make_se(assay2, abundance.columns, experimental.design)
      
#perform DEx
        d = se_abu_data_ALS
        t = test_diff(d, type = "all", control = control, #the DEx function from the DEP package
            test = NULL, design_formula = formula(covariates_f))
        res[[l]] = as.data.frame(t@elementMetadata@listData)
        pval_columns = grep("p.val", colnames(res[[l]]))
        pval_names = colnames(res[[l]])[pval_columns]
        fdr_names = gsub("p.val", "fdr", pval_names)
        if(length(pval_columns)==1){
          res[[l]]$fdr = p.adjust(res[[l]][,pval_columns], method="BH") #calculate fdr with BH adjustment
        }else{
          #if we have 3 clusters we have three different comparisons
          res[[l]]$fdr.1 = p.adjust(res[[l]][,pval_columns[1]], method="BH") 
          res[[l]]$fdr.2 = p.adjust(res[[l]][,pval_columns[2]], method="BH")
          res[[l]]$fdr.3 = p.adjust(res[[l]][,pval_columns[3]], method="BH")
          colnames(res[[l]])[(ncol(res[[l]])-2) : ncol(res[[l]])] = fdr_names
        }
        #remove p.adj and CI columns
        padj_columns = grep("p.adj", colnames(res[[l]])) #because we already have the FDR
        CI_columns = grep("CI", colnames(res[[l]])) #unnecessary information
        res[[l]] = res[[l]][,-c(padj_columns, CI_columns)]
        
        names(res)[l] = title
        
#GSEA analysis 
        if(length(pval_columns) == 1){ #if we have k = 2, basically, since k=3 would have multiple pval columns
        p = res[[l]]$fdr
        p = -log10(p) #make the p a -log10(FDR) instead of normal fdr
        logfc_name = colnames(res[[l]])[grep("diff", colnames(res[[l]]))] #find the name of the fold-change column
        p[res[[l]][,logfc_name]<0] = -p[res[[l]][,logfc_name]<0] #make the -log10(FDR) signed
        names(p) = res[[l]]$name #add the protein names to the signed -log10(FDR) vector
        for(j in 1:length(ontologies)){
          title2 = paste0("GSEA_",title, "_", ontologies[j]) #make title to note the setttings
          set.seed(9)
          r = clusterprofiler_gsea(data = p, ont = ontologies[j], title = title2, alpha = alpha) #our clusterprofiler GSEA function in action
          plots[[u]] = r[[1]] #save emapplot to plots list
          cnetplots[[u]] = r[[2]] #save cnetplot to cnetplot list
          results[[u]] = r[[3]] #save results table to results list
          #give list elements the appropriate names
          names(plots)[u] = title2
          names(cnetplots)[u] = title2
          names(results)[u] = title2
          u = u+1 #go to next list element 
          }
        }else{ #if we have k = 3
          for(m in 1:length(pval_columns)){ #a loop for all comparisons, because we have three comparisons when running k=3 DEx
            #first comparison
            p = res[[l]][,fdr_names[m]]
            p = -log10(p) #make fdr into -log10(FDR)
            logfc_name = colnames(res[[l]])[grep("diff", colnames(res[[l]]))][m] #grab the name of the logfc column
            print(logfc_name)
            p[res[[l]][,logfc_name]<0] = -p[res[[l]][,logfc_name]<0] #make the -log10(FDR) signed
            names(p) = res[[l]]$name #give names to this vector
            for(j in 1:length(ontologies)){
              title2 = paste0("GSEA_",title, "_", ontologies[j], "_", logfc_name) #make this title to know where we are in the loops
              set.seed(9)
              r = clusterprofiler_gsea(data = p, ont = ontologies[j], title = title2, alpha = alpha) #use our gsea function
              plots[[u]] = r[[1]] #save emapplot to plots list
              cnetplots[[u]] = r[[2]] #save cnetplot to cnetplot list
              results[[u]] = r[[3]] #save results table to results list
              #give list elements the appropriate names
              names(plots)[u] = title2
              names(cnetplots)[u] = title2
              names(results)[u] = title2
              u = u+1
            }
          }
        }
        l = l+1
}
```

```
## Tested contrasts: alpha_vs_beta
```

```
## preparing geneSet collections...
```

```
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (35.37% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## Warning in fgseaMultilevel(pathways = pathways, stats = stats, minSize =
## minSize, : For some pathways, in reality P-values are less than 1e-05. You can
## set the `eps` argument to zero for better estimation.
```

```
## leading edge analysis...
```

```
## done...
```

```
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.
## preparing geneSet collections...
## 
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (35.37% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.

## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : For some pathways, in reality P-values are less than 1e-05. You can set the `eps` argument to zero for better estimation.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (35.37% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.Tested contrasts: alpha_vs_theta, alpha_vs_beta, theta_vs_beta
```

```
## [1] "alpha_vs_beta_diff"
```

```
## preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (46.79% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.

## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : For some pathways, in reality P-values are less than 1e-05. You can set the `eps` argument to zero for better estimation.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (46.79% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.

## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : For some pathways, in reality P-values are less than 1e-05. You can set the `eps` argument to zero for better estimation.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (46.79% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.
```

```
## [1] "alpha_vs_theta_diff"
```

```
## preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (40.69% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## no term enriched under specific pvalueCutoff...
## preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (40.69% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## leading edge analysis...
## done...
## preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (40.69% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## leading edge analysis...
## done...
```

```
## [1] "theta_vs_beta_diff"
```

```
## preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (39.75% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## no term enriched under specific pvalueCutoff...
## preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (39.75% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.

## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : For some pathways, in reality P-values are less than 1e-05. You can set the `eps` argument to zero for better estimation.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.preparing geneSet collections...
## GSEA analysis...
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (39.75% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## leading edge analysis...
## done...
## Scale for size is already present.
## Adding another scale for size, which will replace the existing scale.
```

## Save results and plots for DEx and GSEA


```r
#save plots
      plots_k2 = plots[1:3] #the first three plots in the plots list are the k=2 plots
      plots_k3 = plots[4:length(plots)] #4 and above are the k=3 plots
      plots_k3_alpha_beta = plots_k3[grep("alpha_vs_beta", names(plots_k3))] #select the alpha vs beta plots
      plots_k3_alpha_theta = plots_k3[grep("alpha_vs_theta", names(plots_k3))]
      plots_k3_theta_beta = plots_k3[grep("theta_vs_beta", names(plots_k3))]
      
      ggarrange(plotlist = plots_k2, ncol = 3, nrow = 1) #we put three plots in one
```

```
## Warning: ggrepel: 18 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 56 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-1.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k2.pdf", width = 11*3, height = 8, units = "in") #here we save the plot
      
      a = length(plots_k3_alpha_beta)
      ggarrange(plotlist = plots_k3_alpha_beta, ncol = 3, nrow = a/3)
```

```
## Warning: ggrepel: 48 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 57 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-2.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k3_alpha_beta.pdf", width = 11*3, height = 8*(a/3), units = "in") 
      
      a = length(plots_k3_alpha_theta)
      ggarrange(plotlist = plots_k3_alpha_theta, ncol = 3, nrow = a/3)
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-3.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k3_alpha_theta.pdf", width = 11*3, height = 8*(a/3), units = "in") 
      
      a = length(plots_k3_theta_beta)
      ggarrange(plotlist = plots_k3_theta_beta, ncol = 3, nrow = a/3)
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

```
## Warning: ggrepel: 52 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-4.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k3_theta_beta.pdf", width = 11*3, height = 8*(a/3), units = "in") 
      
#save big version of the plots
      ggarrange(plotlist = plots_k2, ncol = 3, nrow = 1)
```

```
## Warning: ggrepel: 18 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 56 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-5.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k2_big.pdf", width = 11*3*2, height = 8*2, units = "in", limitsize = FALSE) 
      
      a = length(plots_k3_alpha_beta)
      ggarrange(plotlist = plots_k3_alpha_beta, ncol = 3, nrow = a/3)
```

```
## Warning: ggrepel: 48 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 57 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-6.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k3_alpha_beta_big.pdf", width = 11*3*2, height = 8*2*(a/3), units = "in", limitsize = FALSE) 
      
      a = length(plots_k3_alpha_theta)
      ggarrange(plotlist = plots_k3_alpha_theta, ncol = 3, nrow = a/3)
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-7.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k3_alpha_theta_big.pdf", width = 11*3*2, height = 8*2*(a/3), units = "in", limitsize = FALSE) 
      
      a = length(plots_k3_theta_beta)
      ggarrange(plotlist = plots_k3_theta_beta, ncol = 3, nrow = a/3)
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

```
## Warning: ggrepel: 52 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-8.png)<!-- -->

```r
      ggsave("plots/comparison_GSEA_kmeans_k3_theta_beta_big.pdf", width = 11*3*2, height = 8*2*(a/3), units = "in", limitsize = FALSE) 


for(i in 1:length(cnetplots)){
  if(!is.character(cnetplots[[i]])){ #only if there is a plot stored in the list element and not a character we make the plot
    print(cnetplots[[i]])
    ggsave(paste0("plots/cnetplots_",names(cnetplots)[i] ,".pdf"), width = 11*2, height = 8*2, units = "in") 
  }
}
```

```
## Warning: ggrepel: 38 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-9.png)<!-- -->

```
## Warning: ggrepel: 145 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-10.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-11.png)<!-- -->

```
## Warning: ggrepel: 75 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-12.png)<!-- -->

```
## Warning: ggrepel: 159 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-13.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-14.png)<!-- -->

```
## Warning: ggrepel: 129 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-15.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Save results and plots for GSEA-16.png)<!-- -->

```r
saveRDS(results, file = "results/GSEA_results_clustering_kmeans_.rds")     #save all results as an RDS file

library(writexl)
# Write the list to an Excel file
write_xlsx(res, path = "results/DEx_results_kmeans.xlsx") #save DEx results to an excel file
names(results) = gsub("GSEA_", "", names(results))
names(results) = gsub("_diff", "", names(results))
write_xlsx(results, path = "results/GSEA_results_kmeans.xlsx") #save GSEA results to an excel file
```


##Visualization 1: plot fit measures (including elbow plot)


```r
set.seed(9)

scores = list(AIC = AIC_scores, BIC = BIC_scores, silhouette = silhouette_scores, TWSS = TWSS_scores) #make list where I put all the fit measures
plots = list() #initiate empty plot list

for(i in 1:length(scores)){ #loop that iterates through all types of scores
  colnames(scores[[i]]) = 1:10
  scores[[i]]$method = rownames(scores[[i]])
  melt = reshape::melt(scores[[i]])
  
  plots[[i]] = ggplot(melt, aes(x=variable, y=value, group = method, colour = method)) +
    geom_line() +
    geom_point() +
    ggtitle(names(scores)[i]) +
    theme_few()
  
}
```

```
## Using method as id variables
## Using method as id variables
## Using method as id variables
## Using method as id variables
```

```r
names(plots) = names(scores)

allplots <- ggarrange(plotlist=plots,
                      labels = LETTERS[1:length(plots)],
                      ncol = 2, nrow = 2)
```

```
## Warning: Removed 4 rows containing missing values (`geom_line()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_line()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_line()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_line()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```r
ggsave("plots/fit_scores.pdf", width = 11, height = 8, units = "in") 
```
##Visualization 2: UMAP plots


```r
# set seed for reproducible results
      set.seed(9)
      library(RColorBrewer)
      library(ggpubr)
      colour_set = brewer.pal(10, "Set3")
      colour_set[2] = "lightgoldenrod2"
      cluster_assignments <- perform_clustering(assay, 9, clin_labels = clin$patid)

#FUNCTION FOR MAKING THE UMAPS
      #it takes 
      #data: an abundancy table, 
      #ggtitle: name for plot title
      #legend_name: title for the  legend, 
      #labels: categorical variable to colour the dots in the UMAP plots
      #colour_set: named vector to determine the colours in the UMAP
UMAP_density_plot = function(data, 
                             ggtitle = "UMAP with disease status labels", 
                             legend_name = "Disease status", 
                             labels = clin$Condition, 
                             colour_set = c("seagreen4", "slateblue1", "salmon")){
      # run umap function
      umap_out = umap::umap(data)
      umap_plot = as.data.frame(umap_out$layout)
      
      #add condition labels
      umap_plot$group = labels

      # plot umap
      p1 = ggplot(umap_plot) + geom_point(aes(x=V1, y=V2, color = as.factor(group))) +
        ggtitle(ggtitle) +
          theme_few() +
          scale_colour_few() +
          scale_color_manual(name = legend_name, 
                           labels = levels(as.factor(umap_plot$group)), 
                           values = colour_set) 
  
      xdens <- 
        axis_canvas(p1, axis = "x") + 
        geom_density(data = umap_plot, aes(x = V1, fill = group, colour = group), alpha = 0.3) +
        scale_fill_manual( values = colour_set) + 
        scale_colour_manual( values = colour_set)
      ydens <-
        axis_canvas(p1, axis = "y", coord_flip = TRUE) + 
        geom_density(data = umap_plot, aes(x = V2, fill = group, colour = group), alpha = 0.3) +
        coord_flip() +
        scale_fill_manual(values = colour_set) + 
        scale_colour_manual( values = colour_set)
      p1 %>%
        insert_xaxis_grob(xdens, grid::unit(1, "in"), position = "top") %>%
        insert_yaxis_grob(ydens, grid::unit(1, "in"), position = "right") %>%
        ggdraw()
      
      p2 = p1 + geom_text(label = rownames(umap_plot), x = umap_plot$V1, y = umap_plot$V2,
                     hjust = 0, nudge_x = 1, size = 1.5, colour = "grey")
      
      print(p1)
      return(list(plot_without_patid_labels = p1, plot_with_patid_labels = p2))
}
      
      UMAP_plots = UMAP_plots_labels = list() #initiate empty list for UMAP plots
      
      for(i in 2:ncol(cluster_assignments)){
        title = colnames(cluster_assignments)[i]
        labels = as.factor(cluster_assignments[,i])
        names(labels) = cluster_assignments[,1]
        colour_subset = colour_set[1:length(levels(labels))]
      
#perform plots with function      
      umaps = UMAP_density_plot(data = assay, #using our in-house function for the UMAP
                                ggtitle = paste0("UMAP with cluster labels\n", title), 
                                legend_name = "Cluster labels", 
                                labels = labels, 
                                colour_set = colour_subset)
      UMAP_plots[[i-1]] = umaps$plot_without_patid_labels #saving the plot to the plotlist
      names(UMAP_plots)[i-1] = title #giving the list element a name
      
      UMAP_plots_labels[[i-1]] = umaps$plot_with_patid_labels
      names(UMAP_plots_labels)[i-1] = title
      }
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-1.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-2.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-3.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-4.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-5.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-6.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-7.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-8.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-9.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-10.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-11.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-12.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-13.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-14.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-15.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-16.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-17.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-18.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-19.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-20.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-21.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-22.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-23.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-24.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-25.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-26.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-27.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-28.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
## no non-missing arguments to max; returning -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-29.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
## no non-missing arguments to max; returning -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-30.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-31.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-32.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
## no non-missing arguments to max; returning -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-33.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-34.png)<!-- -->

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-35.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 2: UMAP plots-36.png)<!-- -->

```r
      UMAP_plots_subset = UMAP_plots[1:(6*4)] #only take until k=7
      UMAP_plots_labels_subset = UMAP_plots_labels[1:(6*4)] #only take until k=7
      
      allplots <- ggarrange(plotlist=UMAP_plots_subset, #plot all UMAPs in one big plot
                            labels = 1:length(UMAP_plots_subset),
                            ncol = 4, nrow = 6)
      ggsave("plots/UMAPs.pdf", width = 11*2, height = 8*2, units = "in") 
      
      allplots <- ggarrange(plotlist=UMAP_plots_labels_subset, #plot all UMAPs in one big plot
                            labels = 1:length(UMAP_plots_labels_subset),
                            ncol = 4, nrow = 6)
      ggsave("plots/UMAPs_labeled.pdf", width = 11*2, height = 8*2, units = "in") 
```

##Visualization 2b: UMAPs for paper


```r
UMAP_density_plot = function(data, 
                             ggtitle = "UMAP with disease status labels", 
                             legend_name = "Disease status", 
                             labels = clin$Condition, 
                             file_location = "plots/UMAP_condition.pdf", 
                             file_location_labels = "plots/UMAP_condition_labels.pdf",
                             colour_set = c("seagreen4", "slateblue1", "salmon"), 
                             shape = rep(16, nrow(umap_plot)), 
                             shapeTF = F){
      # run umap function
      umap_out = umap::umap(data)
      umap_plot = as.data.frame(umap_out$layout)
      
      #add condition labels
      umap_plot$group = labels

      # plot umap
      p1 = ggplot(umap_plot) + 
        geom_point(aes(x=V1, y=V2, color = as.factor(group),), shape=shape, alpha = 0.75, size = 4) +
        ggtitle(ggtitle) +
          theme_few() +
          scale_colour_few() +
          scale_color_manual(name = legend_name, 
                           labels = levels(as.factor(umap_plot$group)), 
                           values = colour_set) + 
          scale_fill_manual(values=colour_set) +
        labs(x = "UMAP1", y = "UMAP2")

      #add shape argument if we want to change shapes
      if(shapeTF){
        p1 = p1 + scale_shape_manual(name = "Sex", 
                    labels = levels(as.factor(shape)), 
                    values=c(15, 17))
      }
  
      xdens <- 
        axis_canvas(p1, axis = "x") + 
        geom_density(data = umap_plot, aes(x = V1, fill = group, colour = group), alpha = 0.3) +
        scale_fill_manual( values = colour_set) + 
        scale_colour_manual( values = colour_set)
      ydens <-
        axis_canvas(p1, axis = "y", coord_flip = TRUE) + 
        geom_density(data = umap_plot, aes(x = V2, fill = group, colour = group), alpha = 0.3) +
        coord_flip() +
        scale_fill_manual(values = colour_set) + 
        scale_colour_manual( values = colour_set)
      p1 %>%
        insert_xaxis_grob(xdens, grid::unit(1, "in"), position = "top") %>%
        insert_yaxis_grob(ydens, grid::unit(1, "in"), position = "right") %>%
        ggdraw()
      
      p1
      # save umap
      ggsave(file_location, width = 11/2, height = 8/2, units = "in")
      
      p1 + geom_text(label = rownames(umap_plot), x = umap_plot$V1, y = umap_plot$V2,
                     hjust = 0, nudge_x = 1, size = 1.5, colour = "grey")
      
      # save umap with labels
      ggsave(file_location_labels, width = 11/2, height = 8/2, units = "in")
}

#make the two plots
        #k = 2
        title = "kmeans_k=2"
        labels = as.factor(cluster_assignments[,title])
        names(labels) = cluster_assignments[,1]
        levels(labels) = c("alpha", "beta")  
    UMAP_density_plot(data = assay, 
                          ggtitle = "UMAP with cluster labels (k-means k=2)", 
                          legend_name = "Cluster label", 
                          labels = labels, 
                          file_location = "plots/paper/UMAP_cluster_k2.pdf", 
                          file_location_labels = "plots/paper/UMAP_cluster_k2_labels.pdf", 
                          colour_set = final_colours$clustering)
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```r
        #k = 3
        title = "kmeans_k=3"
        labels = as.factor(cluster_assignments[,title])
        names(labels) = cluster_assignments[,1]
        levels(labels) = c("beta", "theta", "alpha")  #first is beta, second is theta, third is alpha
        labels <- ordered(labels, levels = c("alpha", "beta", "theta"))
    UMAP_density_plot(data = assay, 
                          ggtitle = "UMAP with cluster labels (k-means k=3)", 
                          legend_name = "Cluster label", 
                          labels = labels, 
                          file_location = "plots/paper/UMAP_cluster_k3.pdf", 
                          file_location_labels = "plots/paper/UMAP_cluster_k3_labels.pdf", 
                          colour_set = final_colours$clustering)
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

##Visualization 3: Sankey plots


```r
set.seed(9)

library(ggalluvial)
library(tidyverse)
```

```
## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
## ✔ forcats   1.0.0     ✔ tibble    3.2.1
## ✔ lubridate 1.9.2     ✔ tidyr     1.3.0
## ✔ purrr     1.0.2     
## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
## ✖ lubridate::%within%()    masks IRanges::%within%()
## ✖ data.table::between()    masks dplyr::between()
## ✖ IRanges::collapse()      masks dplyr::collapse()
## ✖ Biobase::combine()       masks BiocGenerics::combine(), dplyr::combine()
## ✖ dplyr::count()           masks matrixStats::count()
## ✖ IRanges::desc()          masks dplyr::desc()
## ✖ tidyr::expand()          masks S4Vectors::expand()
## ✖ dplyr::filter()          masks clusterProfiler::filter(), stats::filter()
## ✖ data.table::first()      masks S4Vectors::first(), dplyr::first()
## ✖ lubridate::hour()        masks data.table::hour()
## ✖ lubridate::isoweek()     masks data.table::isoweek()
## ✖ dplyr::lag()             masks stats::lag()
## ✖ data.table::last()       masks dplyr::last()
## ✖ purrr::map()             masks mclust::map()
## ✖ lubridate::mday()        masks data.table::mday()
## ✖ lubridate::minute()      masks data.table::minute()
## ✖ lubridate::month()       masks data.table::month()
## ✖ BiocGenerics::Position() masks ggplot2::Position(), base::Position()
## ✖ lubridate::quarter()     masks data.table::quarter()
## ✖ purrr::reduce()          masks GenomicRanges::reduce(), IRanges::reduce()
## ✖ S4Vectors::rename()      masks dplyr::rename(), clusterProfiler::rename()
## ✖ lubridate::second()      masks data.table::second(), S4Vectors::second()
## ✖ lubridate::second<-()    masks S4Vectors::second<-()
## ✖ purrr::simplify()        masks clusterProfiler::simplify()
## ✖ IRanges::slice()         masks dplyr::slice(), clusterProfiler::slice()
## ✖ lubridate::stamp()       masks cowplot::stamp()
## ✖ purrr::transpose()       masks data.table::transpose()
## ✖ lubridate::wday()        masks data.table::wday()
## ✖ lubridate::week()        masks data.table::week()
## ✖ lubridate::yday()        masks data.table::yday()
## ✖ lubridate::year()        masks data.table::year()
## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors
```

```r
## Sankey plots within certain number of clusters, all four methods compared

      number_of_clusters = as.character(2:10)
      plots = list()
      
      for(i in 1:length(number_of_clusters)){
        data = cluster_assignments[,c(1, grep(number_of_clusters[i], colnames(cluster_assignments)))]
        data = reshape2::melt(data)
        colnames(data) = c("patid", "method", "cluster")
        data = as.data.frame(lapply(data, as.factor))
      
      plots[[i]] = ggplot(data,
               aes(x = method, stratum = cluster, alluvium = patid,
                   fill = cluster, label = cluster)) +
          scale_fill_brewer(type = "qual", palette = "Set3") +
          geom_flow(stat = "alluvium", lode.guidance = "frontback",
                   color = "darkgray") +
          geom_stratum() +
          theme(legend.position = "bottom") +
          ggtitle(paste0("How patients are clustered in different clustering algorithms \nk=", number_of_clusters[i])) +
          theme_few()
      }
```

```
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
```

```r
      allplots <- ggarrange(plotlist=plots,
                            labels = 1:length(plots),
                            ncol = 3, nrow = 3)
      ggsave("plots/Sankey_plots_between_methods.pdf", width = 11*2, height = 8*2, units = "in") 


# Sankey plots within a method, different number of clusters

    methods = c("kmeans", "hclust", "mclust", "pam")
    plots = list()
      
      for(i in 1:length(methods)){
        data = cluster_assignments[,c(1, grep(methods[i], colnames(cluster_assignments)))]
        
        data = reshape2::melt(data, id = "patid")
        colnames(data) = c("patid", "method", "cluster")
        data = as.data.frame(lapply(data, as.factor))
      
      plots[[i]] = ggplot(data,
               aes(x = method, stratum = cluster, alluvium = patid,
                   fill = cluster, label = cluster)) +
          scale_fill_brewer(type = "qual", palette = "Set3") +
          geom_flow(stat = "alluvium", lode.guidance = "frontback",
                    color = "darkgray") +
          geom_stratum() +
          theme(legend.position = "bottom") +
          ggtitle(paste0("How patients are clustered within one clustering algorithm \nmethod = ", methods[i])) +
          theme_few() + 
          theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))
      }
      
      allplots <- ggarrange(plotlist=plots,
                            labels = 1:length(plots),
                            ncol = 2, nrow = 2)
      ggsave("plots/Sankey_plots_within_method.pdf", width = 11*2, height = 8*2, units = "in") 
```
##Visualization 4: Sankey plot pathways and within kmeans k2 and k3


```r
      pathways_in_k2 = do.call(rbind, results[grep("k=2", names(results))])
      pathways_in_k2$cluster = rep("alpha", nrow(pathways_in_k2))
      pathways_in_k2$cluster[pathways_in_k2$enrichmentScore<0] = "beta"
      pathways_in_k2$cluster = as.factor(pathways_in_k2$cluster)
      
      results_k3 = results[grep("k=3", names(results))]
      for(i in 1:length(results_k3)){
        name_a = str_split(names(results_k3)[i], "_")[[1]][4]
        name_b = str_split(names(results_k3)[i], "_")[[1]][6]
        results_k3[[i]]$cluster = rep(name_a, nrow(results_k3[[i]]))
        results_k3[[i]]$cluster[results_k3[[i]]$enrichmentScore<0] = name_b
        results_k3[[i]]$cluster = as.factor(results_k3[[i]]$cluster)
      }
      pathways_in_k3 = do.call(rbind, results_k3)
      
      pathways_in_k2$ID = as.factor(pathways_in_k2$ID)
      pathways_in_k3$ID = as.factor(pathways_in_k3$ID)
      
      #deal with duplicates
      duplicates <- duplicated(pathways_in_k2[, "ID"])
      pathways_in_k2 <- pathways_in_k2[!duplicates, ]
      duplicates <- duplicated(pathways_in_k3[, "ID"])
      pathways_in_k3 <- pathways_in_k3[!duplicates, ]
      
      
      pathways_in_k2$n_clusters = rep("2_clusters", nrow(pathways_in_k2))
      pathways_in_k3$n_clusters = rep("3_clusters", nrow(pathways_in_k3))
      
      pathways_sankey = as.data.frame(rbind(pathways_in_k2, pathways_in_k3))
      pathways_sankey$n_clusters = as.factor(pathways_sankey$n_clusters )
      pathways_sankey$ID = as.factor(pathways_sankey$ID )
      data = pathways_sankey[,c("ID", "cluster", "n_clusters")]
      
            
            ggplot(data,
                     aes(x = n_clusters, stratum = cluster, alluvium = ID,
                         fill = cluster, label = cluster)) +
                scale_fill_brewer(type = "qual", palette = "Set3") +
                geom_flow(stat = "alluvium", lode.guidance = "frontback",
                          color = "darkgray") +
                geom_stratum() +
                theme(legend.position = "bottom") +
                ggtitle("Sankey plot within K-means, from 2 clusters to 3 clusters \nPATHWAYS") +
                theme_few()
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 4: Sankey plot pathways and within kmeans k2 and k3-1.png)<!-- -->

```r
            ggsave("plots/Sankey_plots_kmeans_2_3_pathways.pdf", width = 11, height = 8, units = "in")
            
   data = pivot_longer(cluster_assignments_2, cols = 2:3)
   colnames(data) = c("patid", "k", "cluster")
   data$cluster = as.factor(data$cluster)
   levels(data$cluster) = c("alpha", "beta", "theta")
   
              ggplot(data,
                     aes(x = k, stratum = cluster, alluvium = patid,
                         fill = cluster, label = cluster)) +
                scale_fill_brewer(type = "qual", palette = "Set3") +
                geom_flow(stat = "alluvium", lode.guidance = "frontback",
                          color = "darkgray") +
                geom_stratum() +
                theme(legend.position = "bottom") +
                ggtitle("Patient Sankey plot within K-means, from 2 clusters to 3 clusters \nPATIENTS") +
                theme_few()
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 4: Sankey plot pathways and within kmeans k2 and k3-2.png)<!-- -->

```r
    ggsave("plots/Sankey_plot_kmeans_2_3_patients.pdf", width = 11, height = 8, units = "in")
```

## Visualization 4b: Sankey plots for paper


```r
cluster_assignments_2 = cluster_assignments[,c("patid", "kmeans_k=2", "kmeans_k=3", "kmeans_k=4")]

#set the levels for the clusters
      #levels k=2
      cluster_assignments_2$`kmeans_k=2` = as.factor(cluster_assignments_2$`kmeans_k=2`)
      levels(cluster_assignments_2$`kmeans_k=2`) = c("alpha", "beta")
      cluster_assignments_2$`kmeans_k=2` = ordered(cluster_assignments_2$`kmeans_k=2`, levels = c("alpha", "beta", "theta", "zeta"))
      
      #levels k=3
      cluster_assignments_2$`kmeans_k=3` = as.factor(cluster_assignments_2$`kmeans_k=3`)
      levels(cluster_assignments_2$`kmeans_k=3`) = c("beta", "theta", "alpha")  #first is beta, second is theta, third is alpha
      cluster_assignments_2$`kmeans_k=3` <- ordered(cluster_assignments_2$`kmeans_k=3`, levels = c("alpha", "beta", "theta", "zeta"))
      
      #levels k=4
      cluster_assignments_2$`kmeans_k=4` = as.factor(cluster_assignments_2$`kmeans_k=4`)
      levels(cluster_assignments_2$`kmeans_k=4`) = c("theta", "zeta", "beta", "alpha") 
      cluster_assignments_2$`kmeans_k=4` <- ordered(cluster_assignments_2$`kmeans_k=4`, levels = c("alpha", "beta", "theta", "zeta"))

      
      
#Sankey plot with k=2 and k=3
data = pivot_longer(cluster_assignments_2, cols = 2:4)
colnames(data) = c("patid", "k", "cluster")

data23 = data[data$k == "kmeans_k=2" | data$k == "kmeans_k=3",]

   
              ggplot(data23,
                     aes(x = k, stratum = cluster, alluvium = patid,
                         fill = cluster, label = cluster)) +
                scale_fill_manual(values = final_colours$clustering) +
                geom_flow(stat = "alluvium", lode.guidance = "frontback",
                          color = "darkgray") +
                geom_stratum() +
                theme(legend.position = "bottom") +
                ggtitle("Patient Sankey plot within K-means, from 2 clusters to 3 clusters \nPATIENTS") +
                theme_few()
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 4b: Sankey plots for paper-1.png)<!-- -->

```r
    ggsave("plots/paper/Sankey_plot_kmeans_2_3_patients.pdf", width = 11/2, height = 8/2, units = "in")

#Sankey plot with k2-4
   
              ggplot(data,
                     aes(x = k, stratum = cluster, alluvium = patid,
                         fill = cluster, label = cluster)) +
                scale_fill_manual(values = c(final_colours$clustering, zeta = "lightpink")) +
                geom_flow(stat = "alluvium", lode.guidance = "frontback",
                          color = "darkgray") +
                geom_stratum() +
                theme(legend.position = "bottom") +
                ggtitle("Patient Sankey plot within K-means, from 2 clusters to 4 clusters \nPATIENTS") +
                theme_few()
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 4b: Sankey plots for paper-2.png)<!-- -->

```r
    ggsave("plots/paper/Sankey_plot_kmeans_2_3_4_patients.pdf", width = 11/2, height = 8/2, units = "in")
```
## Visualization 5a: Volcano plots Dex results


```r
#make volcano plots
      volcano_plot <- function(data_res, alpha_sig, name_title){
        logFC = data_res[,grep("diff",colnames(data_res))]
        fdr = data_res[,grep("fdr",colnames(data_res))]
        df <- data.frame(x = logFC, 
                         y = -log10(fdr),
                         name = data_res$name)
        names(df) <- c("x","y","name")
        df <- df %>%
          mutate(omic_type = case_when(x >= 0 & y >= (-log10(alpha_sig)) ~ "up",
                                       x <= (0) & y >= (-log10(alpha_sig)) ~ "down",
                                       TRUE ~ "ns")) 
        cols <- c("up" = "#d4552b", "down" = "#26b3ff", "ns" = "grey") 
        sizes <- c("up" = 2, "down" = 2, "ns" = 1) 
        alphas <- c("up" = 0.7, "down" = 0.7, "ns" = 0.5)
        ggplot(data = df, aes(x,y)) + 
          geom_point(aes(colour = omic_type), 
                     alpha = 0.5, 
                     shape = 16,
                     size = 3) + 
          geom_hline(yintercept = -log10(alpha_sig),
                     linetype = "dashed") + 
          geom_vline(xintercept = 0,linetype = "dashed") +
          geom_point(data = filter(df, y >= (-log10(alpha_sig))),
                     aes(colour = omic_type), 
                     alpha = 0.5, 
                     shape = 16,
                     size = 4) + 
          #annotate(geom="text", x=-1.9, y= (-log10(alpha_sig)) + 0.15, label="FDR = 10%",size = 5) +
          geom_text_repel(data = filter(df, y >= (-log10(alpha_sig)) & y > 0),
                           aes(label = name),
                           force = 1,
                          #hjust = 1,
                           #nudge_x = - 0.3,
                          #nudge_y = 0.1,
                          #direction = "x",
                           max.overlaps = 10,
                          segment.size = 0.2,
                           size = 4) +
          geom_text_repel(data = filter(df, y >= (-log10(alpha_sig)) & y < 0),
                          aes(label = name),
                          force = 1,
                          #hjust = 0,
                          #nudge_x = 0.3,
                          #nudge_y = 0.1,
                          #direction = "y",
                          max.overlaps = 10,
                          size = 4) +
          scale_colour_manual(values = cols) + 
          scale_fill_manual(values = cols) + 
          scale_x_continuous(expand = c(0, 0), 
                             limits = c(-0.31, 0.31)) + 
          scale_y_continuous(expand = c(0, 0), limits = c(-0.1, NA)) +
          labs(title = name_title,
               x = "log2(fold change)",
               y = expression(-log[10] ~ "(adjusted p-value)"),
               colour = "Differential \nExpression") +
          theme_classic() + # Select theme with a white background  
          theme(axis.title.y = element_text(size = 14),
                axis.title.x = element_text(size = 14),
                axis.text = element_text(size = 12),
                plot.title = element_text(size = 15, hjust = 0.5),
                text = element_text(size = 14)) +
          annotate("text", x = 0.2, y = 0.5, label = paste0(sum(df$omic_type=="up"), " more abundant \n", sum(df$omic_type=="down"), " less abundant"))
      }
      
      plots_FDR0.05 = plots_FDR0.1 = list()
      l = 1
      
      for(i in 1:length(res)){
        if(i == 1){
          diff_name = colnames(res[[i]])[grep("diff", colnames(res[[i]]))]
          name = gsub("_diff", "", diff_name)
          plots_FDR0.05[[l]] = volcano_plot(res[[i]], 0.05 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", names(res)[i], "\n", name))
          plots_FDR0.1[[l]] = volcano_plot(res[[i]], 0.1 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", names(res)[i], "\n", name))
          l = l+1
        }else{
          diff_names = colnames(res[[i]])[grep("diff", colnames(res[[i]]))]
          fdr_names = colnames(res[[i]])[grep("fdr", colnames(res[[i]]))]
          names = gsub("_diff", "", diff_names)
          for(j in 1:length(names)){
            d = res[[i]][,c("name", diff_names[j], fdr_names[j])]
            plots_FDR0.05[[l]] = volcano_plot(d, 0.05 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", names(res)[i], "\n", names[j]))
            plots_FDR0.1[[l]] = volcano_plot(d, 0.1 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", names(res)[i], "\n", names[j]))
            l = l+1
          }
        }
      }
      
      allplots <- ggarrange(plotlist=plots_FDR0.05,
                                  ncol = 4, nrow = 1)
```

```
## Warning: Removed 417 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 74 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 74 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 416 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 103 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 103 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 376 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 93 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 93 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 425 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 81 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 81 rows containing missing values (`geom_text_repel()`).
```

```r
      ggsave("plots/volcano_plots_FDR0.05.pdf", width = 11*4, height = 8, units = "in") 
      
      allplots <- ggarrange(plotlist=plots_FDR0.1,
                                  ncol = 4, nrow = 1)
```

```
## Warning: Removed 417 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 91 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 91 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 416 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 119 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 119 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 376 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 106 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 106 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 425 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 108 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 108 rows containing missing values (`geom_text_repel()`).
```

```r
      ggsave("plots/volcano_plots_FDR0.1.pdf", width = 11*4, height = 8, units = "in")
```

## Visualization 5b: Barplots with top hits for Dex results


```r
#make barplot with FC and coloured with p-value, top n
      
      n = 30

      barplot_FC = function(data, title){ #enter a matrix with name, diff, and fdr
        plot = ggplot(data, aes(x = reorder(name, -diff, decreasing = T), y = diff, fill = fdr)) +
          geom_bar(stat = "identity", position = "identity") +
          coord_flip() +
          scale_fill_gradient(low = "red", high = "purple1") +
          labs(title = title,
               y = "Log2(Fold-Change)",
               x = "Name of protein", 
               colour = "FDR") +
          theme_few()
        return(plot)
      }
    
    important_protein_names = list()  
    all_important_protein_names = list()
      
    #k = 2
      barplots_k2 = list()
        #ALPHA: top 30 ALPHA associated proteins according to FDR
        d = res[[1]]
        d = d[d$alpha_vs_beta_diff>0,]
        d = d[d$fdr<=0.05,]
        all_important_protein_names$k2_alpha = d$name
        d <- d %>%
          arrange(fdr) %>%
          slice_head(n = n)
        colnames(d)[3] = "diff"
        title = paste0("kmeans_k=2, \ntop_", n, "_alpha")
        barplots_k2$alpha = barplot_FC(d, title)
        important_protein_names$k2_alpha = d$name
        
        #BETA: top 30 BETA associated proteins according to FDR
        d = res[[1]]
        d = d[d$alpha_vs_beta_diff<0,]
        d = d[d$fdr<=0.05,]
        all_important_protein_names$k2_beta = d$name
        d$alpha_vs_beta_diff = -d$alpha_vs_beta_diff
        d <- d %>%
          arrange(fdr) %>%
          slice_head(n = n)
        colnames(d)[3] = "diff"
        title = paste0("kmeans_k=2, \ntop_", n, "_beta")
        barplots_k2$beta = barplot_FC(d, title)
        important_protein_names$k2_beta = d$name
      
    #k = 3
      barplots_k3 = list()
      #ALPHA: top 30 alpha associated proteins according to FDR
        d = res[[2]]
        d = d[,c("name", colnames(d)[grep("alpha", colnames(d))])]
        d = d[d$alpha_vs_beta_diff>0 & d$alpha_vs_theta_diff>0,]
        d = d[d$alpha_vs_beta_fdr<=0.05 & d$alpha_vs_theta_fdr<=0.05,]
        all_important_protein_names$k3_alpha = d$name
        d$lowest_fdr = pmin(d$alpha_vs_beta_fdr, d$alpha_vs_theta_fdr)
        d$biggest_FC = pmax(d$alpha_vs_beta_diff, d$alpha_vs_theta_diff)
        d <- d %>%
          arrange(lowest_fdr) %>%
          slice_head(n = n)
        colnames(d)[8] = "fdr"
        colnames(d)[9] = "diff"
        title = paste0("kmeans_k=3, \ntop_", n, "_alpha")
        barplots_k3$alpha = barplot_FC(d, title)
        important_protein_names$k3_alpha = d$name
        
      #BETA: top 30 beta associated proteins according to FDR
        d = res[[2]]
        d = d[,c("name", colnames(d)[grep("beta", colnames(d))])]
        d = d[d$alpha_vs_beta_diff<0 & d$theta_vs_beta_diff<0,]
        d$alpha_vs_beta_diff = -d$alpha_vs_beta_diff
        d$theta_vs_beta_diff = -d$theta_vs_beta_diff
        d = d[d$alpha_vs_beta_fdr<=0.05 & d$theta_vs_beta_fdr<=0.05,]
        all_important_protein_names$k3_beta = d$name
        d$lowest_fdr = pmin(d$alpha_vs_beta_fdr, d$theta_vs_beta_fdr)
        d$biggest_FC = pmax(d$alpha_vs_beta_diff, d$theta_vs_beta_diff)
        d <- d %>%
          arrange(lowest_fdr) %>%
          slice_head(n = n)
        colnames(d)[8] = "fdr"
        colnames(d)[9] = "diff"
        title = paste0("kmeans_k=3, \ntop_", n, "_beta")
        barplots_k3$beta = barplot_FC(d, title)
        important_protein_names$k3_beta = d$name
        
      #THETA: top 30 theta associated proteins according to FDR
        d = res[[2]]
        d = d[,c("name", colnames(d)[grep("theta", colnames(d))])]
        d = d[d$alpha_vs_theta_diff<0 & d$theta_vs_beta_diff>0,]
        d = d[d$alpha_vs_theta_fdr<0.05 & d$theta_vs_beta_fdr<0.05,]
        all_important_protein_names$k3_theta = d$name
        d$lowest_fdr = pmin(d$alpha_vs_theta_fdr, d$theta_vs_beta_fdr)
        d$biggest_FC = pmax(d$alpha_vs_theta_diff, d$theta_vs_beta_diff)
        d <- d %>%
          arrange(lowest_fdr) %>%
          slice_head(n = n)
        colnames(d)[8] = "fdr"
        colnames(d)[9] = "diff"
        title = paste0("kmeans_k=3, \ntop_", n, "_theta")
        barplots_k3$theta = barplot_FC(d, title)
        important_protein_names$k3_theta = d$name
        
      allplots <- ggarrange(plotlist=barplots_k2,
                                        ncol = 2, nrow = 1)
      ggsave("plots/barplot_important_proteins_k2_FDR0.05.pdf", width = 6*2, height = 8, units = "in") 
      allplots <- ggarrange(plotlist=barplots_k3,
                                        ncol = 3, nrow = 1)
      ggsave("plots/barplot_important_proteins_k3_FDR0.05.pdf", width = 6*3, height = 8, units = "in")
```

## Visualization 5c: Violin plots with top hits for Dex results


```r
#make violin plot showing only the most important proteins

  #k = 2
    violin_plots_k2 = list()
    d = assay
    d$cluster = cluster_assignments_2$`kmeans_k=2`

      #alpha
      d_alpha = d[,c("cluster", important_protein_names$k2_alpha)]
      d_alpha = pivot_longer(d_alpha, !cluster)
      
        
      violin_plots_k2$alpha = ggviolin(d_alpha, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=2 \nALPHA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=colour_set[1:2]) +
                      theme_few() 
      
      #beta
      d_beta = d[,c("cluster", important_protein_names$k2_beta)]
      d_beta = pivot_longer(d_beta, !cluster)
      
        
      violin_plots_k2$beta = ggviolin(d_beta, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=2 \nBETA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=colour_set[1:2]) +
                      theme_few() 

    ggarrange(plotlist = violin_plots_k2, ncol = 2, nrow = 1)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 5c: Violin plots with top hits for Dex results-1.png)<!-- -->

```r
    ggsave(filename = "plots/violin_plots_kmeans_k2.pdf", width = 12*2, height = 12*1.5, units = "in")
    
  #k = 3
    violin_plots_k3 = list()
    d = assay
    d$cluster = cluster_assignments_2$`kmeans_k=3`
    
      #alpha
      d_alpha = d[,c("cluster", important_protein_names$k3_alpha)]
      d_alpha = pivot_longer(d_alpha, !cluster)
      
        
      violin_plots_k3$alpha = ggviolin(d_alpha, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=3 \nALPHA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=colour_set[1:3]) +
                      theme_few() 
      
      #beta
      d_beta = d[,c("cluster", important_protein_names$k3_beta)]
      d_beta = pivot_longer(d_beta, !cluster)
      
        
      violin_plots_k3$beta = ggviolin(d_beta, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=3 \nBETA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=colour_set[1:3]) +
                      theme_few()
      
      #theta
      d_theta = d[,c("cluster", important_protein_names$k3_theta)]
      d_theta = pivot_longer(d_theta, !cluster)
      
        
      violin_plots_k3$theta = ggviolin(d_theta, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=3 \nTHETA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=colour_set[1:3]) +
                      theme_few()

    ggarrange(plotlist = violin_plots_k3, ncol = 3, nrow = 1)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 5c: Violin plots with top hits for Dex results-2.png)<!-- -->

```r
    ggsave(filename = "plots/violin_plots_kmeans_k3.pdf", width = 12*3, height = 12*1.5, units = "in")
    
for(i in 1:length(important_protein_names)){
  important_protein_names[[i]] = as.data.frame(important_protein_names[[i]] )
  all_important_protein_names[[i]] = as.data.frame(all_important_protein_names[[i]] )
}    
write_xlsx(important_protein_names, path = "results/top_30_important_protein_names_clusters.xlsx")
write_xlsx(all_important_protein_names, path = "results/all_important_protein_names_clusters.xlsx")
```

## Visualization 5d: Volcano plots for the paper


```r
labels = res[[1]]$name[1:100]

#PLOT OF MINPROB, ALS vs CTRL, NOT STRATIFIED, age corrected

volcano_plot_paper <- function(data_res, alpha_sig, name_title, col_up, col_down){
  logFC = data_res[,grep("diff",colnames(data_res))]
  fdr = data_res[,grep("fdr",colnames(data_res))]
  df <- data.frame(x = logFC, 
                   y = -log10(fdr),
                   name = data_res$name)
  names(df) <- c("x","y","name")
  df <- df %>%
    mutate(omic_type = case_when(x >= 0 & y >= (-log10(alpha_sig)) ~ "up",
                                 x <= (0) & y >= (-log10(alpha_sig)) ~ "down",
                                 TRUE ~ "ns")) 
  cols <- c("up" = col_up, "down" = col_down, "ns" = "grey") 
  sizes <- c("up" = 2, "down" = 2, "ns" = 1) 
  alphas <- c("up" = 0.7, "down" = 0.7, "ns" = 0.5)
  ggplot(data = df, aes(x,y)) + 
    geom_point(aes(colour = omic_type), 
               alpha = 0.5, 
               shape = 16,
               size = 2) + 
    geom_hline(yintercept = -log10(alpha_sig),
               linetype = "dashed") + 
    geom_vline(xintercept = 0,linetype = "dashed") +
    geom_point(data = filter(df, y >= (-log10(alpha_sig))),
               aes(colour = omic_type), 
               alpha = 0.5, 
               shape = 16,
               size = 4) + 
    #annotate(geom="text", x=-1.9, y= (-log10(alpha_sig)) + 0.15, label="FDR = 10%",size = 5) +
    geom_text_repel(data = filter(df, y >= (-log10(alpha_sig)) & y > 0 & name %in% labels),
                     aes(label = name),
                     force = 1,
                    hjust = 1,
                    nudge_x = -0.05,
                    nudge_y = 0.1,
                    direction = "both",
                    max.overlaps = 5,
                     size = 4) +
    geom_text_repel(data = filter(df, y >= (-log10(alpha_sig)) & y < 0 & name %in% labels),
                    aes(label = name),
                    force = 1,
                    hjust = 0,
                    nudge_x = 0.05,
                    nudge_y = 0.1,
                    direction = "both",
                    max.overlaps = 5,
                    size = 4) +
    scale_colour_manual(values = cols) + 
    scale_fill_manual(values = cols) + 
    scale_x_continuous(expand = expansion(mult = .05), limits = c(-1.05*(max(abs(df$x))),
                                                                  1.05*(max(abs(df$x))))) + 
    scale_y_continuous(expand = expansion(mult = .05), limits = c(-0.1, NA)) +
    labs(title = name_title,
         x = "log2(fold change)",
         y = expression(-log[10] ~ "(adjusted p-value)"),
         colour = "Differential \nAbundancy") +
    theme_few() + # Select theme with a white background  
    theme(axis.title.y = element_text(size = 14),
          axis.title.x = element_text(size = 14),
          axis.text = element_text(size = 12),
          plot.title = element_text(size = 15, hjust = 0.5),
          text = element_text(size = 14)) +
    annotate("text", x = 0.1, y = 0.5, label = paste0(sum(df$omic_type=="up"), " more abundant \n", sum(df$omic_type=="down"), " less abundant"))
}

      plots_FDR0.05 = plots_FDR0.1 = list()
      l = 1
      
      for(i in 1:length(res)){
        if(i == 1){
          #k = 2
          diff_name = colnames(res[[i]])[grep("diff", colnames(res[[i]]))]
          name = gsub("_diff", "", diff_name)
          comparison = str_split(name, pattern = "_")[[1]]
          col_up = final_colours$clustering[comparison[1]]
          col_down = final_colours$clustering[comparison[3]]
          names(col_up) = NULL
          names(col_down) = NULL
          
          plots_FDR0.05[[l]] = volcano_plot_paper(data_res = res[[i]], 
                                            alpha_sig = 0.05 , 
                                            col_up = col_up,
                                            col_down = col_down,
                                            name_title = paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", 
                                                                names(res)[i], 
                                                                "\n", 
                                                                name))
          plots_FDR0.1[[l]] = volcano_plot_paper(data_res = res[[i]], 
                                            alpha_sig = 0.1 , 
                                            col_up = col_up,
                                            col_down = col_down,
                                            name_title = paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", 
                                                                names(res)[i], 
                                                                "\n", 
                                                                name))
          l = l+1
        }else{
          #k = 3
          diff_names = colnames(res[[i]])[grep("diff", colnames(res[[i]]))]
          fdr_names = colnames(res[[i]])[grep("fdr", colnames(res[[i]]))]
          names = gsub("_diff", "", diff_names)
          for(j in 1:length(names)){
            name = names[j]
            comparison = str_split(name, pattern = "_")[[1]]
            col_up = final_colours$clustering[comparison[1]]
            col_down = final_colours$clustering[comparison[3]]
            names(col_up) = NULL
            names(col_down) = NULL
            d = res[[i]][,c("name", diff_names[j], fdr_names[j])]
            
            plots_FDR0.05[[l]] = volcano_plot_paper(data_res = d, 
                                            alpha_sig = 0.05 , 
                                            col_up = col_up,
                                            col_down = col_down,
                                            name_title = paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", 
                                                                names(res)[i], 
                                                                "\n", 
                                                                name))
            plots_FDR0.1[[l]] = volcano_plot_paper(data_res = d, 
                                            alpha_sig = 0.1 , 
                                            col_up = col_up,
                                            col_down = col_down,
                                            name_title = paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", 
                                                                names(res)[i], 
                                                                "\n", 
                                                                name))
            l = l+1
          }
        }
      }
      
      allplots <- ggarrange(plotlist=plots_FDR0.05,
                                  ncol = 4, nrow = 1)
      ggsave("plots/paper/volcano_plots_FDR0.05.pdf", width = 5.5*4, height = 4, units = "in") 
```

```
## Warning: ggrepel: 3 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 12 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 11 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 2 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```r
      allplots <- ggarrange(plotlist=plots_FDR0.1,
                                  ncol = 4, nrow = 1)
      ggsave("plots/paper/volcano_plots_FDR0.1.pdf", width = 5.5*4, height = 4, units = "in") 
```

```
## Warning: ggrepel: 7 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 16 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 11 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

```
## Warning: ggrepel: 4 unlabeled data points (too many overlaps). Consider
## increasing max.overlaps
```

## Visualization 5e: Violin plots for the paper


```r
#add fake controls
assay_ctrl = assay
#add a lot of noise
assay_ctrl <- assay_ctrl + rnorm(num_proteins * num_patients, mean = 0, sd = 2)
assay_ctrl = abs(assay_ctrl)
rownames(assay_ctrl) = gsub("patient", "ctrl", rownames(assay_ctrl))
assay_all = rbind(assay, assay_ctrl)

#which proteins do we want to highlight:
high_prot = c(
  k2_alpha = "COL15A1",
  k2_beta = "CDH8",
  k3_alpha = "YWHAE",
  k3_beta = "A1BG",
  k3_theta = "MSN"
)

#make violin plot showing only the most important proteins

  #k = 2
    violin_plots_k2 = list()
    d = assay_all
    d$cluster = as.factor(rep("ctrl", nrow(d)))
    d$cluster <- ordered(d$cluster, levels = c("alpha", "beta", "theta", "ctrl"))
    d[cluster_assignments_2$patid, "cluster"] = cluster_assignments_2$`kmeans_k=2`

      #alpha
      d_alpha = d[,c("cluster", high_prot["k2_alpha"])]
      d_alpha = pivot_longer(d_alpha, !cluster)
      
        
      violin_plots_k2$alpha = ggviolin(d_alpha, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=2 \nALPHA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=c(final_colours$clustering, ctrl = "grey")) +
                      theme_few() 
      
      #beta
      d_beta = d[,c("cluster", high_prot["k2_beta"])]
      d_beta = pivot_longer(d_beta, !cluster)
      
        
      violin_plots_k2$beta = ggviolin(d_beta, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=2 \nBETA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=c(final_colours$clustering, ctrl = "grey")) +
                      theme_few() 

    ggarrange(plotlist = violin_plots_k2, ncol = 2, nrow = 1)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 5e: Violin plots for the paper-1.png)<!-- -->

```r
    ggsave(filename = "plots/paper/violin_plots_kmeans_k2.pdf", width = 8, height = 3, units = "in")
    
  #k = 3
    violin_plots_k3 = list()
    d = assay_all
    d$cluster = as.factor(rep("ctrl", nrow(d)))
    d$cluster <- ordered(d$cluster, levels = c("alpha", "beta", "theta", "ctrl"))
    d[cluster_assignments_2$patid, "cluster"] = cluster_assignments_2$`kmeans_k=3`
    
      #alpha
      d_alpha = d[,c("cluster", high_prot["k3_alpha"])]
      d_alpha = pivot_longer(d_alpha, !cluster)
      
        
      violin_plots_k3$alpha = ggviolin(d_alpha, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=3 \nALPHA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=c(final_colours$clustering, ctrl = "grey")) +
                      theme_few() 
      
      #beta
      d_beta = d[,c("cluster",  high_prot["k3_beta"])]
      d_beta = pivot_longer(d_beta, !cluster)
      
        
      violin_plots_k3$beta = ggviolin(d_beta, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=3 \nBETA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=c(final_colours$clustering, ctrl = "grey")) +
                      theme_few()
      
      #theta
      d_theta = d[,c("cluster", high_prot["k3_theta"])]
      d_theta = pivot_longer(d_theta, !cluster)
      
        
      violin_plots_k3$theta = ggviolin(d_theta, x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "name") +
                      labs(title = "Boxplot of Protein Expression by Cluster for K-means k=3 \nTHETA",
                           x = "Cluster",
                           y = "Abundancy") +
                      scale_fill_manual(values=c(final_colours$clustering, ctrl = "grey")) +
                      theme_few()

    ggarrange(plotlist = violin_plots_k3, ncol = 3, nrow = 1)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 5e: Violin plots for the paper-2.png)<!-- -->

```r
    ggsave(filename = "plots/paper/violin_plots_kmeans_k3.pdf", width = 12, height = 3, units = "in")
```

##Visualization 6a: Heatmap for our own clustering


```r
#select only the important proteins
      k2_proteins = rbind(
        cbind(protein = all_important_protein_names[["k2_alpha"]][,1], cluster = rep("alpha", nrow(all_important_protein_names[["k2_alpha"]]))),
        cbind(protein = all_important_protein_names[["k2_beta"]][,1], cluster = rep("beta", nrow(all_important_protein_names[["k2_beta"]])))
      )
      k2_proteins = as.data.frame(as.matrix(k2_proteins))
      k2_proteins$cluster = as.factor(k2_proteins$cluster)
      k3_proteins = rbind(
        cbind(protein = all_important_protein_names[["k3_alpha"]][,1], cluster = rep("alpha", nrow(all_important_protein_names[["k3_alpha"]]))),
        cbind(protein = all_important_protein_names[["k3_beta"]][,1], cluster = rep("beta", nrow(all_important_protein_names[["k3_beta"]]))),
        cbind(protein = all_important_protein_names[["k3_theta"]][,1], cluster = rep("theta", nrow(all_important_protein_names[["k3_theta"]])))
      )
      k3_proteins = as.data.frame(as.matrix(k3_proteins))
      k3_proteins$cluster = as.factor(k3_proteins$cluster)
      
      assay_k2 = assay[,k2_proteins$protein]
      assay_k3 = assay[,k3_proteins$protein]
      
      cluster_assignments_2$`kmeans_k=2` = as.factor(cluster_assignments_2$`kmeans_k=2`)
      cluster_assignments_2$`kmeans_k=3` = as.factor(cluster_assignments_2$`kmeans_k=3`)
      
      ordered_k2 = cluster_assignments_2[order(cluster_assignments_2$`kmeans_k=2`),]
      ordered_k3 = cluster_assignments_2[order(cluster_assignments_2$`kmeans_k=3`),]
      
      assay_k2 = assay_k2[ordered_k2$patid,]
      assay_k3 = assay_k3[ordered_k3$patid,]
      
#make the heatmap
      library(pheatmap)

      save_pheatmap_pdf <- function(x, filename, width=7, height=7) {
         stopifnot(!missing(x))
         stopifnot(!missing(filename))
         pdf(filename, width=width, height=height)
         grid::grid.newpage()
         grid::grid.draw(x$gtable)
         dev.off()
      }

      set.seed(9)

      assay_k2_scaled = scale(assay_k2)
      assay_k3_scaled = scale(assay_k3)
      
      assay_k2 = as.data.frame(t(assay_k2))
      assay_k2_scaled = as.data.frame(t(assay_k2_scaled))
      assay_k2_capped = assay_k2_scaled
      assay_k2_capped[assay_k2_capped>3] = 3
      assay_k2_capped[assay_k2_capped<(-3)] = -3
      
      assay_k3 = as.data.frame(t(assay_k3))
      assay_k3_scaled = as.data.frame(t(assay_k3_scaled))
      assay_k3_capped = assay_k3_scaled
      assay_k3_capped[assay_k3_capped>3] = 3
      assay_k3_capped[assay_k3_capped<(-3)] = -3
      
      heatmap_data = list(
          datasets = list(assay_k2, 
                          assay_k2_scaled, 
                          assay_k2_capped,
                          assay_k3, 
                          assay_k3_scaled,
                          assay_k3_capped),
        patids = list(ordered_k2$patid, 
                      ordered_k2$patid, 
                      ordered_k2$patid,
                      ordered_k3$patid,
                      ordered_k3$patid,
                      ordered_k3$patid),
        patient_cluster = list(ordered_k2$`kmeans_k=2`, 
                               ordered_k2$`kmeans_k=2`, 
                               ordered_k2$`kmeans_k=2`, 
                               ordered_k3$`kmeans_k=3`, 
                               ordered_k3$`kmeans_k=3`, 
                               ordered_k3$`kmeans_k=3`),
        proteins = list(k2_proteins$protein, 
                        k2_proteins$protein, 
                        k2_proteins$protein, 
                        k3_proteins$protein, 
                        k3_proteins$protein, 
                        k3_proteins$protein),
        protein_cluster = list(k2_proteins$cluster, 
                               k2_proteins$cluster, 
                               k2_proteins$cluster, 
                               k3_proteins$cluster,
                               k3_proteins$cluster, 
                               k3_proteins$cluster),
        titles = c("k2", 
                   "k2_scaled", 
                   "k2_capped", 
                   "k3", 
                   "k3_scaled", 
                   "k3_capped")
      )
      
      for(i in 1:length(heatmap_data$datasets)){
          # Create row and column annotations for k2
          col_annotations <- data.frame(
            Cluster_assignment = as.character(heatmap_data$patient_cluster[[i]])
          )
          rownames(col_annotations) = heatmap_data$patids[[i]]
          
          row_annotations <- data.frame(
            Cluster_subtypes = heatmap_data$protein_cluster[[i]]
          )
          
          ann_colors = list(
            Cluster_assignment = final_colours$clustering,
            Cluster_subtypes = final_colours$clustering)
          
          rownames(row_annotations) = heatmap_data$proteins[[i]]
          title = heatmap_data$titles[[i]]
          
          p = pheatmap(
            heatmap_data$datasets[[i]],
            annotation_row = row_annotations,
            annotation_col = col_annotations,
            annotation_colors = ann_colors,
            cluster_rows = FALSE,
            cluster_cols = FALSE,
            fontsize = 5,  # Adjust the font size if needed
            border_color = NA, 
            main = title
          )
          save_pheatmap_pdf(p, paste0("plots/heatmap_clustering_kmeans_",title,"_.pdf"))
      }
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6a: Heatmap for our own clustering-1.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6a: Heatmap for our own clustering-2.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6a: Heatmap for our own clustering-3.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6a: Heatmap for our own clustering-4.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6a: Heatmap for our own clustering-5.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6a: Heatmap for our own clustering-6.png)<!-- -->
## Visualization 6b: heatmap for paper


```r
# change size of the figure to relatively small
      save_pheatmap_pdf <- function(x, filename, width=4, height=2.5) {
         stopifnot(!missing(x))
         stopifnot(!missing(filename))
         pdf(filename, width=width, height=height)
         grid::grid.newpage()
         grid::grid.draw(x$gtable)
         dev.off()
      }
#choose only capped

indices = c(3,6)

for(i in indices){
     # Create row and column annotations for k2
          col_annotations <- data.frame(
            Cluster_assignment = as.character(heatmap_data$patient_cluster[[i]])
          )
          rownames(col_annotations) = heatmap_data$patids[[i]]
          
          row_annotations <- data.frame(
            Cluster_subtypes = heatmap_data$protein_cluster[[i]]
          )
          
          ann_colors = list(
            Cluster_assignment = final_colours$clustering,
            Cluster_subtypes = final_colours$clustering)
          
          rownames(row_annotations) = heatmap_data$proteins[[i]]
          title = heatmap_data$titles[[i]]
        
          p = pheatmap(
            heatmap_data$datasets[[i]],
            annotation_row = row_annotations,
            annotation_col = col_annotations,
            annotation_colors = ann_colors,
            annotation_names_row = F, # no column labels or row labels
            annotation_names_col = F, # no column labels or row labels
            color = final_colours$heatmap_scale, # change heatmap colour scale
            cluster_rows = FALSE,
            cluster_cols = FALSE,
            fontsize = 5,  # Adjust the font size if needed
            border_color = NA, 
            main = title
          )
          save_pheatmap_pdf(p, paste0("plots/paper/heatmap_clustering_kmeans_",title,"_.pdf"))
  }
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6b: heatmap for paper-1.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/Visualization 6b: heatmap for paper-2.png)<!-- -->
## Visualization 7a: pathway barplot version 1


```r
#create fake pathway annotation

results_annotated = results
for(i in 1:length(results_annotated)){
  #create annotation
  results_annotated[[i]]$annotation = as.factor(sample(c(
    "inflammatory pathway", 
    "cytoskeleton pathways",
    "aging pathway"), 
    nrow(results_annotated[[i]]), replace = TRUE))
  #make the p-adjust a signed -log10(FDR)
  results_annotated[[i]]$log10 = -log10(results_annotated[[i]]$p.adjust)
  results_annotated[[i]]$log10[results_annotated[[i]]$enrichmentScore <0] = -results_annotated[[i]]$log10[results_annotated[[i]]$enrichmentScore <0]
}

for(i in 1:length(results_annotated)){
  
      
      limit_words <- function(text, max_words) {
        words <- strsplit(text, "\\s+")
        filtered_words <- lapply(words, function(word_list) {
          word_list[1:min(length(word_list), max_words)]
        })
        filtered_text <- sapply(filtered_words, function(word_list) {
          paste(word_list, collapse = " ")
        })
        return(filtered_text)
      }
      
      results_annotated[[i]]$ID <- sapply(results_annotated[[i]]$ID, function(text) {
        limit_words(text, 7)
      })
      
      # Create a custom color palette with 7 distinct colors
      colour_subset <- colour_set[1:length(levels(results_annotated[[i]]$annotation))]
      
      results_annotated[[i]]$ID = as.factor(results_annotated[[i]]$ID)
      desired_order = rev(unique(results_annotated[[i]]$ID))
      # Set the category variable as an ordered factor with desired order
      results_annotated[[i]]$ID <- factor(results_annotated[[i]]$ID, levels = desired_order, ordered = TRUE)
      
      # Create a bar plot with ggplot2
      ggplot(results_annotated[[i]], aes(x = ID, y = log10, fill = annotation)) +
        geom_bar(stat = "identity", position = "identity") +
        coord_flip() + scale_y_reverse() +
        
        # Set zero as the middle point on the x-axis
        #scale_y_continuous(limits = c(-max(abs(GSEA_results[[i]]$log10)), max(abs(GSEA_results[[i]]$log10)))) +
        # Add labels and customize the plot
        labs(title = paste0("Bar Plot of signed log10(FDR)\n", names(results_annotated)[i]),
             x = "ID",
             y = "Signed log10(FDR)") +
        theme_few() +
        # Customize colors using the custom color palette
        scale_fill_manual(values = colour_subset) +
        # Adjust x-axis label font size
        theme(axis.text.y = element_text(size = 7))
            
      ggsave(paste0("plots/barplot_GSEA_annotation_", names(results_annotated)[i],"version1.pdf"),
             width = 8, 
             height = 1 + nrow(results_annotated[[i]])*0.1,
             units = "in")

}
```

## Visualization 7b: pathway barplot version 2


```r
for(i in 1:length(results_annotated)){
  
  if(nrow(results_annotated[[i]]) > 0){
  
  order = as.character(unique(results_annotated[[i]]$annotation))
  results_annotated[[i]]$annotation = factor(results_annotated[[i]]$annotation, levels = order)
  
  comparison = names(results_annotated)[i]
  comparison = str_split(comparison, pattern = "_")[[1]]
  up = comparison[1]
  down = comparison[3]
  results_annotated[[i]]$comparison = rep(up, nrow(results_annotated[[i]]))
  results_annotated[[i]]$comparison[results_annotated[[i]]$enrichmentScore < 0] = down


# Create a bar plot with ggplot2

  #WHITE FACETGRID
      ggplot(results_annotated[[i]], aes(x = ID, y = log10, fill = comparison)) +
        geom_bar(stat = "identity", position = "identity", color = "white") +
        coord_flip() + 
        # Add labels and customize the plot
        labs(title = paste0("Bar Plot of signed log10(FDR)\n", names(results_annotated)[i]),
             x = "ID",
             y = "Signed log10(FDR)") +
        theme_few() +
        # Customize colors using the custom color palette
        scale_fill_manual(values = final_colours$clustering, aesthetics = "fill") +
        # Adjust x-axis label font size
        #theme(axis.text.y = element_text(size = 7))
        theme(axis.text.y = element_blank()) +
        facet_grid(annotation ~ ., scales = "free", space = "free") +
        theme(strip.text.y = element_text(angle = 0)) + 
        theme(panel.spacing = unit(0.1, "lines"))
            
      ggsave(paste0("plots/paper/barplot_GSEA_annotation_", names(results_annotated)[i],"_facetgrid_white.pdf"),
             width = 5, 
             height = 1 + nrow(results_annotated[[i]])*0.06,
             units = "in")
      
  #BLACK FACETGRID    
      ggplot(results_annotated[[i]], aes(x = ID, y = log10, fill = comparison)) +
        geom_bar(stat = "identity", position = "identity", color = "black") +
        coord_flip() + 
        # Add labels and customize the plot
        labs(title = paste0("Bar Plot of signed log10(FDR)\n", names(results_annotated)[i]),
             x = "ID",
             y = "Signed log10(FDR)") +
        theme_few() +
        # Customize colors using the custom color palette
        scale_fill_manual(values = final_colours$clustering, aesthetics = "fill") +
        # Adjust x-axis label font size
        #theme(axis.text.y = element_text(size = 7))
        theme(axis.text.y = element_blank()) +
        facet_grid(annotation ~ ., scales = "free", space = "free") +
        theme(strip.text.y = element_text(angle = 0)) + 
        theme(panel.spacing = unit(0.1, "lines"))
            
      ggsave(paste0("plots/paper/barplot_GSEA_annotation_", names(results_annotated)[i],"_facetgrid_black.pdf"),
             width = 5, 
             height = 1 + nrow(results_annotated[[i]])*0.06,
             units = "in")
     
    #NO FACETGRID   
            ggplot(results_annotated[[i]], aes(x = ID, y = log10, fill = comparison)) +
        geom_bar(stat = "identity", position = "identity") +
        coord_flip() + 
        # Add labels and customize the plot
        labs(title = paste0("Bar Plot of signed log10(FDR)\n", names(results_annotated)[i]),
             x = "ID",
             y = "Signed log10(FDR)") +
        theme_few() +
        # Customize colors using the custom color palette
        scale_fill_manual(values = final_colours$clustering) +
        # Adjust x-axis label font size
        #theme(axis.text.y = element_text(size = 7))
        theme(axis.text.y = element_blank())
            
      ggsave(paste0("plots/paper/barplot_GSEA_annotation_", names(results_annotated)[i],".pdf"),
             width = 4, 
             height = 1 + nrow(results_annotated[[i]])*0.04,
             units = "in")
  }
}
```

## Protein Family/Domain enrichment analysis


```r
#create an annotation file with proteins and corresponding protein domains
    library(httr)
```

```
## 
## Attaching package: 'httr'
```

```
## The following object is masked from 'package:Biobase':
## 
##     content
```

```r
    # Function to query UniProt API
    get_uniprot_info <- function(accession_numbers) {
      base_url <- "https://www.uniprot.org/uniprot/"
      response <- GET(paste0(base_url, accession_numbers, ".txt"))
      info <- content(response, "text", encoding = "UTF-8")
      return(info)
    }
    
    # UniProt accession numbers (replace with your own)
    uniprot_to_genename = readRDS(file = "data/uniprot_to_genename.rds")
    uniprot_to_genename$better_gene_name[is.na(uniprot_to_genename$better_gene_name)] = uniprot_to_genename$gene_name[is.na(uniprot_to_genename$better_gene_name)]
    uniprot_accession_numbers = uniprot_to_genename[uniprot_to_genename$better_gene_name %in% colnames(assay), c("uniprot_accession", "better_gene_name")] 
    
    # Get information about proteins
    protein_info <- lapply(uniprot_accession_numbers$uniprot_accession, get_uniprot_info)
    
    protein_domains = data.frame(matrix(ncol = 3))
    colnames(protein_domains) = c("uniprot", "gene_name", "domain")
    
    for(i in 1:length(protein_info)){
        text = protein_info[[i]]
        # Replace "\\n" with actual line breaks
        formatted_text <- gsub("\\\\n", "\n", text)
        # Find the first line that starts with "DR   Pfam"
        selected_lines <- grep("^DR\\s+Pfam", strsplit(formatted_text, "\n")[[1]], value = TRUE)
        if(length(selected_lines)>=1){
          for(j in 1:length(selected_lines)){
          selected_line = selected_lines[j]
          new_row = data.frame(
            domain = str_split_i(string = selected_line, pattern = ";", i = 3),
            uniprot = uniprot_accession_numbers$uniprot_accession[i],
            gene_name = uniprot_accession_numbers$better_gene_name[i]
          )
          protein_domains = rbind(protein_domains, new_row)
          }
        }
    }

    protein_and_domain_list <- split(protein_domains$gene_name, protein_domains$domain)

    #load DEx results
    excel_file_path <- "results/DEx_results_kmeans.xlsx"
    all_sheets <- readxl::excel_sheets(excel_file_path)
    res <- lapply(all_sheets, function(sheet) {
      readxl::read_excel(excel_file_path, sheet = sheet)
    })
    names(res) = all_sheets
        
    library(fgsea)
    gsea_result = list()
    

    # PROTEIN GROUP GSEA TEST IN K = 2
    d = as.data.frame(res[[1]])
    
    d$log10fdr = -log10(d$fdr)
    d$log10fdr[d$alpha_vs_beta_diff<0] = -d$log10fdr[d$alpha_vs_beta_diff<0]
    
    d = d[order(d$log10fdr, decreasing = T),]
    d2 = d$log10fdr
    d2 = setNames(d2, d$name)
    
    # Run GSEA
    gsea_result[[1]] <- fgsea(pathways = protein_and_domain_list, stats = d2, minSize = 3)
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (35.37% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```r
    names(gsea_result)[1] = "alpha_vs_beta_k2"
    
    # Perform Benjamini-Hochberg correction
    gsea_result[[1]]$fdr <- p.adjust(gsea_result[[1]]$pval, method = "BH")
    
    # PROTEIN GROUP GSEA TEST IN K = 3
    d = as.data.frame(res[[2]])
    comparisons_fdr = colnames(d)[grep("fdr", colnames(d))]
    comparisons_diff = gsub("_fdr", "_diff", comparisons_fdr)
    comparisons = gsub("_fdr", "", comparisons_fdr)
    
    for(i in 1:length(comparisons)){
        
        d$log10fdr = -log10(d[,comparisons_fdr[i]])
        d$log10fdr[d[,comparisons_diff[i]]<0] = -d$log10fdr[d[,comparisons_diff[i]]<0]
        
        d = d[order(d$log10fdr, decreasing = T),]
        d2 = d$log10fdr
        d2 = setNames(d2, d$name)
        
        # Run GSEA
        gsea_result[[i+1]] <- fgsea(pathways = protein_and_domain_list, stats = d2, minSize = 3)
        names(gsea_result)[i+1] = paste0(comparisons[i], "_k3")
        
        # Perform Benjamini-Hochberg correction
        gsea_result[[i+1]]$fdr <- p.adjust(gsea_result[[i+1]]$pval, method = "BH")
    }
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (46.79% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (40.69% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (39.75% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```r
    #Plot the results
    plot_results = list()
    for(i in 1:length(gsea_result)){
      
        d = gsea_result[[i]]
        name = names(gsea_result)[i]
        d = d[d$fdr<=0.2,]
        # Concatenate sentences into a character vector
        d$leadingEdge <- sapply(d$leadingEdge, function(sentence) paste(sentence, collapse = " "))
        d = d[order(d$ES, decreasing = T),]
        # make hits/size variable
        d$hits = str_count(d$leadingEdge, "\\S+")
        d$hits_and_size =  paste0(d$hits, "/", d$size)
        
        # Create the barplot
        ggplot(d, aes(x = ES, y = reorder(pathway, ES), fill = fdr, label = hits_and_size)) +
          geom_bar(stat = "identity") +
          labs(title = paste0("Enrichment Scores for Protein Domains\nFDR 20% cut-off\n", name),
               x = "Enrichment Score",
               y = "Protein Domain") +
          scale_fill_gradient(limits = c(0,0.2)) +
          theme_few() +
          geom_text(size = 4,  position = position_stack(vjust = 0.5), colour = "white")  # Adjust the text position if needed
          
              ggsave(paste0("plots/barplot_protein_domain_enrichment_", name,".pdf"),
                 width = 5, 
                 height = 1 + nrow(d)*0.2,
                 units = "in")
      
          plot_results[[i]] = d
          names(plot_results)[i] = name
    
    }
    
    write_xlsx(plot_results, path = "results/barplot_protein_domain_enrichment_results.xlsx")
    write_xlsx(gsea_result, path = "results/protein_domain_enrichment_results_all.xlsx")
    
    
#REPEAT WITH SETSIZE 1
    
    # PROTEIN GROUP GSEA TEST IN K = 2
    d = as.data.frame(res[[1]])
    
    d$log10fdr = -log10(d$fdr)
    d$log10fdr[d$alpha_vs_beta_diff<0] = -d$log10fdr[d$alpha_vs_beta_diff<0]
    
    d = d[order(d$log10fdr, decreasing = T),]
    d2 = d$log10fdr
    d2 = setNames(d2, d$name)
    
    # Run GSEA
    gsea_result[[1]] <- fgsea(pathways = protein_and_domain_list, stats = d2, minSize = 1)
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (35.37% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```r
    names(gsea_result)[1] = "alpha_vs_beta_k2"
    
    # Perform Benjamini-Hochberg correction
    gsea_result[[1]]$fdr <- p.adjust(gsea_result[[1]]$pval, method = "BH")
    
    # PROTEIN GROUP GSEA TEST IN K = 3
    d = as.data.frame(res[[2]])
    comparisons_fdr = colnames(d)[grep("fdr", colnames(d))]
    comparisons_diff = gsub("_fdr", "_diff", comparisons_fdr)
    comparisons = gsub("_fdr", "", comparisons_fdr)
    
    for(i in 1:length(comparisons)){
        
        d$log10fdr = -log10(d[,comparisons_fdr[i]])
        d$log10fdr[d[,comparisons_diff[i]]<0] = -d$log10fdr[d[,comparisons_diff[i]]<0]
        
        d = d[order(d$log10fdr, decreasing = T),]
        d2 = d$log10fdr
        d2 = setNames(d2, d$name)
        
        # Run GSEA
        gsea_result[[i+1]] <- fgsea(pathways = protein_and_domain_list, stats = d2, minSize = 1)
        names(gsea_result)[i+1] = paste0(comparisons[i], "_k3")
        
        # Perform Benjamini-Hochberg correction
        gsea_result[[i+1]]$fdr <- p.adjust(gsea_result[[i+1]]$pval, method = "BH")
    }
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (46.79% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (40.69% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```
## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (39.75% of the list).
## The order of those tied genes will be arbitrary, which may produce unexpected results.
```

```r
    #Plot the results
    plot_results = list()
    for(i in 1:length(gsea_result)){
      
        d = gsea_result[[i]]
        name = names(gsea_result)[i]
        d = d[d$fdr<=0.2,]
        # Concatenate sentences into a character vector
        d$leadingEdge <- sapply(d$leadingEdge, function(sentence) paste(sentence, collapse = " "))
        d = d[order(d$ES, decreasing = T),]
        # make hits/size variable
        d$hits = str_count(d$leadingEdge, "\\S+")
        d$hits_and_size =  paste0(d$hits, "/", d$size)
        
        # Create the barplot
        ggplot(d, aes(x = ES, y = reorder(pathway, ES), fill = fdr, label = hits_and_size)) +
          geom_bar(stat = "identity") +
          labs(title = paste0("Enrichment Scores for Protein Domains\nFDR 20% cut-off\n", name),
               x = "Enrichment Score",
               y = "Protein Domain") +
          scale_fill_gradient(limits = c(0,0.2)) +
          theme_few() +
          geom_text(size = 4,  position = position_stack(vjust = 0.5), colour = "white")  # Adjust the text position if needed
          
              ggsave(paste0("plots/barplot_protein_domain_enrichment_", name,"_setsize1.pdf"),
                 width = 5, 
                 height = 1 + nrow(d)*0.2,
                 units = "in")
      
          plot_results[[i]] = d
          names(plot_results)[i] = name
    
    }
    
    write_xlsx(plot_results, path = "results/barplot_protein_domain_enrichment_results_setsize1.xlsx")
    write_xlsx(gsea_result, path = "results/protein_domain_enrichment_results_all_setsize1.xlsx")
```

## Investigate clinical variables between clusters

### TEST ASSUMPTIONS: are the clinical variables normally distributed?


```r
hist(clin$age)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK ASSUMPTIONS: are the clinical variables normally distributed?-1.png)<!-- -->

```r
hist(clin$neurofilaments)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK ASSUMPTIONS: are the clinical variables normally distributed?-2.png)<!-- -->

```r
hist(clin$progression_rate)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK ASSUMPTIONS: are the clinical variables normally distributed?-3.png)<!-- -->

### Perform DEx with sex stratification


```r
#settings for upcoming analyses
cluster_assignments <- perform_clustering(assay, 9, clin_labels = clin$patid)
cluster_assignments = cluster_assignments[,1:9]
cluster_levels = c("alpha", "beta", "theta")
set.seed(9)
res = list()
l = u = 1
covariates = "age_cov"
covariates_f = ~0 + condition + age_cat
control = "beta"
ontologies = c("BP", "CC", "MF")
plots = list()
cnetplots = list()
results = list()
alpha = 0.1

cluster_assignments_2 = cluster_assignments[,c("patid", "kmeans_k=2", "kmeans_k=3")]

for(k in 2:ncol(cluster_assignments_2)){
        set.seed(9)

#make summarized experiment, this time with cluster as condition, MALE
        clin$cluster = as.factor(cluster_assignments_2[,k])
        clin_male = clin[clin$sex == "Male",]
        assay_male = assay[clin$patid[clin$sex == "Male"],]
        levels(clin_male$cluster) = cluster_levels[1:length(levels(clin_male$cluster))]
        assay2 = as.data.frame(t(assay_male))
        assay2$ID = assay2$name = rownames(assay2)
        abundance.columns <- grep("patient", colnames(assay2)) # get abundance column numbers
        experimental.design = clin_male[, c("patid","cluster", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")]
        colnames(experimental.design) = c("label","condition", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")
        experimental.design$replicate = 1:nrow(experimental.design)
        se_abu_data_ALS_male <- make_se(assay2, abundance.columns, experimental.design)
        
        #make summarized experiment, this time with cluster as condition, FEMALE
        clin$cluster = as.factor(cluster_assignments_2[,k])
        clin_female = clin[clin$sex == "Female",]
        assay_female = assay[clin$patid[clin$sex == "Female"],]
        levels(clin_female$cluster) = cluster_levels[1:length(levels(clin_female$cluster))]
        assay2 = as.data.frame(t(assay_female))
        assay2$ID = assay2$name = rownames(assay2)
        abundance.columns <- grep("patient", colnames(assay2)) # get abundance column numbers
        experimental.design = clin_female[, c("patid","cluster", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")]
        colnames(experimental.design) = c("label","condition", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")
        experimental.design$replicate = 1:nrow(experimental.design)
        se_abu_data_ALS_female <- make_se(assay2, abundance.columns, experimental.design)
        
        se_sex_strata = list(se_abu_data_ALS_female = se_abu_data_ALS_female, 
                             se_abu_data_ALS_male = se_abu_data_ALS_male)
        
        for(i in 1:length(se_sex_strata)){
            #perform DEx
          title = paste0(colnames(cluster_assignments_2)[k], "_", names(se_sex_strata)[i])
          d = se_sex_strata[[i]]
          t = test_diff(d, type = "all", control = control,
              test = NULL, design_formula = formula(covariates_f))
          res[[l]] = as.data.frame(t@elementMetadata@listData)
          pval_columns = grep("p.val", colnames(res[[l]]))
          pval_names = colnames(res[[l]])[pval_columns]
          fdr_names = gsub("p.val", "fdr", pval_names)
          if(length(pval_columns)==1){
            res[[l]]$fdr = p.adjust(res[[l]][,pval_columns], method="BH")
          }else{
            res[[l]]$fdr.1 = p.adjust(res[[l]][,pval_columns[1]], method="BH")
            res[[l]]$fdr.2 = p.adjust(res[[l]][,pval_columns[2]], method="BH")
            res[[l]]$fdr.3 = p.adjust(res[[l]][,pval_columns[3]], method="BH")
            colnames(res[[l]])[(ncol(res[[l]])-2) : ncol(res[[l]])] = fdr_names
          }
          #remove p.adj and CI columns
          padj_columns = grep("p.adj", colnames(res[[l]]))
          CI_columns = grep("CI", colnames(res[[l]]))
          res[[l]] = res[[l]][,-c(padj_columns, CI_columns)]
          names(res)[l] = title
          l = l+1
        }

}
```

```
## Tested contrasts: alpha_vs_beta
## Tested contrasts: alpha_vs_beta
```

```
## Tested contrasts: theta_vs_beta, theta_vs_alpha, alpha_vs_beta
```

```
## Tested contrasts: theta_vs_beta, alpha_vs_beta, theta_vs_alpha
```

```r
library(writexl)
# Write the list to an Excel file
names(res) = gsub("_se_abu_data", "", names(res))
write_xlsx(res, path = "results/DEx_results_kmeans_sex_stratified.xlsx")
```

### Volcano plots DEx with sex stratification


```r
      plots_FDR0.05 = plots_FDR0.1 = list()
      l = 1
      
      for(i in 1:length(res)){
        if(i <= 2){
          diff_name = colnames(res[[i]])[grep("diff", colnames(res[[i]]))]
          name = gsub("_diff", "", diff_name)
          plots_FDR0.05[[l]] = volcano_plot(res[[i]], 0.05 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", names(res)[i], "\n", name))
          plots_FDR0.1[[l]] = volcano_plot(res[[i]], 0.1 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", names(res)[i], "\n", name))
          l = l+1
        }else{
          diff_names = colnames(res[[i]])[grep("diff", colnames(res[[i]]))]
          fdr_names = colnames(res[[i]])[grep("fdr", colnames(res[[i]]))]
          names = gsub("_diff", "", diff_names)
          for(j in 1:length(names)){
            d = res[[i]][,c("name", diff_names[j], fdr_names[j])]
            plots_FDR0.05[[l]] = volcano_plot(d, 0.05 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", names(res)[i], "\n", names[j]))
            plots_FDR0.1[[l]] = volcano_plot(d, 0.1 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", names(res)[i], "\n", names[j]))
            l = l+1
          }
        }
      }
      
      allplots <- ggarrange(plotlist=plots_FDR0.05,
                                  ncol = 4, nrow = 2)
```

```
## Warning: Removed 431 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 18 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 18 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 456 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 28 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 28 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 454 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 21 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 21 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 470 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 481 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 2 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 2 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 449 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 24 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 24 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 453 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 29 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 29 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 440 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 51 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 51 rows containing missing values (`geom_text_repel()`).
```

```r
      ggsave("plots/volcano_plots_FDR0.05_sex_stratified.pdf", width = 11*4, height = 8*2, units = "in") 
      
      allplots <- ggarrange(plotlist=plots_FDR0.1,
                                  ncol = 4, nrow = 2)
```

```
## Warning: Removed 431 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 25 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 25 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 456 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 39 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 39 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 454 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 35 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 35 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 470 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 14 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 14 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 481 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 12 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 12 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 449 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 53 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 53 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 453 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 60 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 60 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 440 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 70 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 70 rows containing missing values (`geom_text_repel()`).
```

```r
      ggsave("plots/volcano_plots_FDR0.1_sex_stratified.pdf", width = 11*4, height = 8*2, units = "in") 
      
#plot male vs female in one scatterplot
      # takes datatable with "x" for women, "y" for men, and "name" variables
    scatterplot_FDR_male_female = function(data,
                                           cut_off = -log10(0.1),
                                           q = 0.95, 
                                           main_title, 
                                           max.overlaps = 10, 
                                           lab_x = "signed -log10(FDR) for females",
                                           lab_y = "signed -log10(FDR) for males",
                                           labels_T_F = T,
                                           annotate_YN = T){
      data$omic_type = rep("ns", nrow(data))
      text_y = paste0("significant in males")
      text_x = paste0("significant in females")
      data$omic_type[abs(data$y) >= cut_off] = text_y
      data$omic_type[abs(data$x) >= cut_off] = text_x
      data$omic_type[(abs(data$x) >= cut_off) & (abs(data$y) >= cut_off)] = "significant in both"
      cols <- c("x" = "salmon", "y" = "#26b3ff", "ns" = "grey", "significant in both" = "mediumpurple1") 
      attributes(cols)$names[1] = text_x
      attributes(cols)$names[2] = text_y
      
      quantile_y = quantile(abs(data$y),na.rm = T, probs = q)
      quantile_x = quantile(abs(data$x),na.rm = T, probs = q)
    
     plot = ggplot(data, aes(x,y)) +
      geom_point(aes(colour = omic_type),
                   alpha = 0.5,
                   shape = 16,
                   size = 2) +
      geom_point(data = filter(data, abs(y) >= cut_off | abs(x) >= cut_off),
                   aes(colour = omic_type), 
                   alpha = 0.5, 
                   shape = 16,
                   size = 3) + 
      geom_hline(yintercept = cut_off, linetype = "dashed", colour = "grey40") +
      geom_hline(yintercept = -cut_off, linetype = "dashed", colour = "grey40") +
      geom_vline(xintercept = cut_off, linetype = "dashed", colour = "grey40") +
      geom_vline(xintercept = -cut_off, linetype = "dashed", colour = "grey40") +
      geom_hline(yintercept = 0, linetype = "dashed", colour = "grey80") +
      geom_vline(xintercept = 0, linetype = "dashed", colour = "grey80") +
         geom_text_repel(data = filter(data, abs(y) >= quantile_y | abs(x) >= quantile_x),
                         aes(label = name),
                         force = 1,
                        hjust = 1,
                        #nudge_x = - 0.05,
                        #nudge_y = 0.05,
                        #direction = "y",
                        max.overlaps = max.overlaps,
                        segment.size = 0.2,
                         size = 2)  +
        scale_colour_manual(values = cols) + 
        scale_fill_manual(values = cols) +
      labs(title = main_title,
             x = lab_x,
             y = lab_y,
             colour = "Differential \nExpression") +
        theme_classic() + # Select theme with a white background  
        theme(axis.title.y = element_text(size = 14),
              axis.title.x = element_text(size = 14),
              axis.text = element_text(size = 12),
              plot.title = element_text(size = 15, hjust = 0.5),
              text = element_text(size = 14)) +
       if(annotate_YN){
          annotate("text", x = 0.75, y = -2, label = 
                   paste0(sum(data$omic_type==text_y), " ", text_y,"\n", 
                          sum(data$omic_type==text_x), " ", text_x,"\n", 
                          sum(data$omic_type=="significant in both"), " significant in both"))
       }
       
     return(plot)}
    
    #ALPHA VS BETA K2
    
    plots = list()
    #MAKE PLOT WITH FDR      
    sex_m_k2 = as.data.table(cbind(res$`kmeans_k=2_ALS_female`$name, 
                                                  res$`kmeans_k=2_ALS_female`$fdr, 
                                                  res$`kmeans_k=2_ALS_male`$fdr))
    colnames(sex_m_k2) = c("name", "x", "y")
    sex_m_k2$x = as.numeric(sex_m_k2$x)
    sex_m_k2$y = as.numeric(sex_m_k2$y)
    sex_m_k2$x = -log10(sex_m_k2$x)
    sex_m_k2$y = -log10(sex_m_k2$y)
    sex_m_k2$x[res$`kmeans_k=2_ALS_female`$alpha_vs_beta_diff <0] = -sex_m_k2$x[res$`kmeans_k=2_ALS_female`$alpha_vs_beta_diff <0]
    sex_m_k2$y[res$`kmeans_k=2_ALS_male`$alpha_vs_beta_diff <0] = -sex_m_k2$y[res$`kmeans_k=2_ALS_male`$alpha_vs_beta_diff <0]

    plots[[1]] = scatterplot_FDR_male_female(data = sex_m_k2, 
                                             q = 0.90, 
                                              max.overlaps = 15, 
                                             main_title = "FDR scatterplot DEx results \nalpha vs beta \nsex stratified \nFDR 0.1 cut-off")
    
    #MAKE PLOT WITH log-fold-change     
    sex_m_k2 = as.data.table(cbind(res$`kmeans_k=2_ALS_female`$name, 
                                                  res$`kmeans_k=2_ALS_female`$alpha_vs_beta_diff, 
                                                  res$`kmeans_k=2_ALS_male`$alpha_vs_beta_diff))
    colnames(sex_m_k2) = c("name", "x", "y")
    sex_m_k2$x = as.numeric(sex_m_k2$x)
    sex_m_k2$y = as.numeric(sex_m_k2$y)
    plots[[2]] = scatterplot_FDR_male_female(data = sex_m_k2, 
                                             main_title = "Fold-Change scatterplot DEx results \nalpha vs beta \nkmeans k=2, sex stratified \nFC 0.05 cut-off", 
                                             cut_off = 0.05, 
                                             annotate_YN = F,
                                             lab_x = "FC for females",
                                             lab_y = "FC for males")
    
    allplots <- ggarrange(plotlist=plots,
                                  ncol = 2, nrow = 1)
    ggsave("plots/scatterplots_FDR_and_FC_sex_stratified_k2.pdf", width = 11*2, height = 8, units = "in") 
    
    #K3
    
    comparisons = c("alpha_vs_beta", "theta_vs_alpha", "theta_vs_beta")
    plots = list()
    l = 1
    
    for(i in 1:length(comparisons)){

        #MAKE PLOT WITH FDR
        fdr_name = paste0(comparisons[i], "_fdr")
        fc_name = paste0(comparisons[i], "_diff")
        sex_m_k3 = as.data.table(cbind(res$`kmeans_k=3_ALS_female`$name, 
                                                      res$`kmeans_k=3_ALS_female`[,fdr_name], 
                                                      res$`kmeans_k=3_ALS_male`[,fdr_name]))
        colnames(sex_m_k3) = c("name", "x", "y")
        sex_m_k3$x = as.numeric(sex_m_k3$x)
        sex_m_k3$y = as.numeric(sex_m_k3$y)
        sex_m_k3$x = -log10(sex_m_k3$x)
        sex_m_k3$y = -log10(sex_m_k3$y)
        sex_m_k3$x[res$`kmeans_k=3_ALS_female`[,fc_name] <0] = -sex_m_k3$x[res$`kmeans_k=3_ALS_female`[,fc_name] <0]
        sex_m_k3$y[res$`kmeans_k=3_ALS_male`[,fc_name] <0] = -sex_m_k3$y[res$`kmeans_k=3_ALS_male`[,fc_name] <0]
    
        plots[[l]] = scatterplot_FDR_male_female(data = sex_m_k3, 
                                                 q = 0.90, 
                                                  max.overlaps = 15, 
                                                 main_title = paste0(
                                                   "FDR scatterplot DEx results \n", comparisons[i], "\nkmeans k=3, sex stratified \nFDR 0.1 cut-off"))
        names(plots)[l] = paste0(comparisons[i], "_FDR")
        l = l+1
        
        #MAKE PLOT WITH log-fold-change     
        sex_m_k3 = as.data.table(cbind(res$`kmeans_k=3_ALS_female`$name, 
                                                      res$`kmeans_k=3_ALS_female`[,fc_name], 
                                                      res$`kmeans_k=3_ALS_male`[,fc_name]))
        colnames(sex_m_k3) = c("name", "x", "y")
        sex_m_k3$x = as.numeric(sex_m_k3$x)
        sex_m_k3$y = as.numeric(sex_m_k3$y)
        plots[[l]] = scatterplot_FDR_male_female(data = sex_m_k3, 
                                                 main_title = paste0("Fold-Change scatterplot DEx results \n", 
                                                                     comparisons[i], 
                                                                     "\nkmeans k=3, sex stratified \nFC 0.05 cut-off"), 
                                                 cut_off = 0.05, 
                                                 annotate_YN = F,
                                                 lab_x = "FC for females",
                                                 lab_y = "FC for males")
        names(plots)[l] = paste0(comparisons[i], "_FC")
        l = l+1
    }
    
    allplots <- ggarrange(plotlist=plots,
                                  ncol = 2, nrow = 3)
    ggsave("plots/scatterplots_FDR_and_FC_sex_stratified_k3.pdf", width = 11*2, height = 8*3, units = "in") 
```

### Test difference clinical characteristics between clusters


```r
set.seed(9)
library(nnet)
library(stats)
library(dunn.test)


#for each clustering algorithm, for each number of clusters, test if there are differences in the clinical variables
clin_vars = c("age", "sex", "neurofilaments", "age_at_onset", "progression_rate", "onset")
pval_clin = c()
models_clin = list()
l = 1

for(i in 2:ncol(cluster_assignments)){
  for(j in 1:length(clin_vars)){
     var = clin_vars[j]
     title = paste0(colnames(cluster_assignments)[i],"_",var)
     
        if(grepl("k=2", title, fixed = TRUE)){
          if(var == "neurofilaments" | var == "progression_rate"){
          #for k = 2
          model <- kruskal.test(cluster_assignments[,i] ~ clin[,var])
          d = as.data.frame(rbind(model[[2]], round(model[[3]],2)))
          rownames(d) = c("degrees of freedom", "p-value")
          colnames(d) = "Kruskal-wallis test"
          models_clin[[l]] = d
          pval_clin[l] = model[["p.value"]]
          }else{
          #for k = 2
          model <- glm(cluster_assignments[,i] ~ clin[,var], family = "binomial")
          models_clin[[l]] = summary(model)
          pval_clin[l] = models_clin[[l]]$coefficients[2,"Pr(>|z|)"]
          }
          
        }else{
           if(var == "neurofilaments" | var == "progression_rate"){
          #for k = 2
          model <- kruskal.test(cluster_assignments[,i] ~ clin[,var])
          d = as.data.frame(rbind(model[[2]], round(model[[3]],2)))
          rownames(d) = c("degrees of freedom", "p-value")
          colnames(d) = "Kruskal-wallis test"
          models_clin[[l]] = d
          pval_clin[l] = model[["p.value"]]
          }else{
          # for k > 2
          model <- multinom(cluster_assignments[,i] ~ clin[,var])
          models_clin[[l]] = summary(model)
          z <- summary(model)$coefficients/summary(model)$standard.errors
          p <- (1 - pnorm(abs(z), 0, 1)) * 2
          pval_clin[l] = min(p[,2])
          }
        }
        names(pval_clin)[l] = title
        names(models_clin)[l] = title
        l = l+1
  }
}
```

```
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.845760 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.014961 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## iter  10 value 53.107446
## iter  10 value 53.107446
## final  value 53.107446 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 52.444486 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.845760 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.014961 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## iter  10 value 53.107446
## iter  10 value 53.107446
## final  value 53.107446 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 52.444486 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## iter  10 value 53.845760
## iter  10 value 53.845760
## final  value 53.845760 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.014961 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## iter  10 value 53.107447
## iter  10 value 53.107447
## final  value 53.107447 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 52.444486 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.845760 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 53.014961 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## iter  10 value 53.107446
## iter  10 value 53.107446
## final  value 53.107446 
## converged
## # weights:  9 (4 variable)
## initial  value 54.930614 
## final  value 52.444486 
## converged
```




### Plot clinical variables between clusters, including results tests


```r
library(gridExtra)
```

```
## 
## Attaching package: 'gridExtra'
```

```
## The following object is masked from 'package:Biobase':
## 
##     combine
```

```
## The following object is masked from 'package:BiocGenerics':
## 
##     combine
```

```
## The following object is masked from 'package:dplyr':
## 
##     combine
```

```r
tt2 <- ttheme_minimal()

#for each clustering algorithm, for each number of clusters, test if there are differences in the clinical variables
clin_vars = c("age", "sex", "neurofilaments", "progression_rate", "onset")
colours = list(
  age = "darkgreen",
  sex = c(Female = "lightpink", Male ="lightblue3"),
  neurofilaments = "royalblue",
  progression_rate = "orange3",
  onset = c(spinal = "mediumpurple1", bulbar = "mediumaquamarine")
)
plots = list()

l = 0
for(i in 2:ncol(cluster_assignments)){
  for(j in 1:length(clin_vars)){
    l = l+1
    var = clin_vars[j]
    title = paste0(colnames(cluster_assignments)[i],"_",var)
  
    #how many clusters?
    if(grepl("k=2", colnames(cluster_assignments)[i])){
      if(var == "neurofilaments" | var == "progression_rate"){
        coeff = models_clin[[title]]}else{
          #with k=2
        coeff = round(models_clin[[title]]$coefficients,2)
        }
    }else{
      if(var == "neurofilaments" | var == "progression_rate"){
        coeff = models_clin[[title]]
      }else{
        #with k>2
        z <- models_clin[[title]]$coefficients/models_clin[[title]]$standard.errors
        p <- (1 - pnorm(abs(z), 0, 1)) * 2
        coeff = as.data.frame(models_clin[[title]]$coefficients)
        coeff$pvalue = p
        coeff = round(coeff, 2)
        coeff = as.data.frame(as.matrix(coeff))
        colnames(coeff) = c("Intercept", var, "p-value intercept", paste0("p-value ", var))
      }
       
    }
    linesize = ifelse(pval_clin[title]<=0.05, 2, 0)
  
  #is it a factor?
  if(is.factor(clin[,var])){

    #factor variables
    df = data.frame(clin[,var], as.factor(cluster_assignments[,i]))
    colnames(df) = c("variable", "cluster")
    df = as.data.frame(proportions(table(df), margin = 2))

    p = ggplot(df, aes(fill=variable, y=Freq, x=cluster)) + 
            geom_bar(position="fill", stat="identity") + 
          annotation_custom(tableGrob(coeff, theme= ttheme_minimal(base_size = 8)), 
                            #xmin = 1.8, xmax = 2, 
                            ymin = 0, ymax = 0.2) +
          scale_fill_manual(values=colours[[var]]) +
          labs(title = title,
           x = "cluster",
           y = paste0("proportions of ", var),
           fill = var) +
          theme_few() +
          theme(panel.background = element_rect(colour = "red", size=linesize))
    plots[[l]] = p
    names(plots)[l] = title
    
  }else{
    #numeric variables
    df = data.frame(clin[,var], as.factor(cluster_assignments[,i]))
    colnames(df) = c("variable", "cluster")
    df = na.omit(df)

    p = ggplot(df, aes(y=variable, x=cluster)) + 
            geom_boxplot(fill = colours[[var]]) + 
          labs(title = title,
           x = "cluster",
           y = var) +
          guides(fill="none") +
          theme_few() +
          theme(panel.background = element_rect(colour = "red", size=linesize)) +
          annotation_custom(tableGrob(coeff, theme = ttheme_minimal(base_size = 8)), 
                            #xmin = length(levels(df$cluster))-0.5, 
                            #xmax = length(levels(df$cluster)), 
                            ymin = min(df$variable), 
                            ymax = min(df$variable)+10) 
    plots[[l]] = p
    names(plots)[l] = title
  }
    }}
```

```
## Warning: The `size` argument of `element_rect()` is deprecated as of ggplot2 3.4.0.
## ℹ Please use the `linewidth` argument instead.
## This warning is displayed once every 8 hours.
## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
## generated.
```

```r
#age for 2, 3, and 4 clusters
plots234 = plots[c(
    grep("k=2", names(plots)),
    grep("k=3", names(plots)),
    grep("k=4", names(plots)))]

plots_age = plots234[grep("age", names(plots234))]
plots_sex = plots234[grep("sex", names(plots234))]
plots_neurofilaments = plots234[grep("neuro", names(plots234))]
plots_progression = plots234[grep("progr", names(plots234))]
plots_onset = plots234[grep("onset", names(plots234))]

ggarrange(plotlist = plots_age, nrow = length(plots_age)/4, ncol = 4) 
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK CLINICAL DATA: plot clinical variables, stratified by cluster-1.png)<!-- -->

```r
ggsave("plots/clinical_variables_clusters_age.pdf", width = 11*2, height = 8*2, units = "in") 
ggarrange(plotlist = plots_sex, nrow = length(plots_sex)/4, ncol = 4) 
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK CLINICAL DATA: plot clinical variables, stratified by cluster-2.png)<!-- -->

```r
ggsave("plots/clinical_variables_clusters_sex.pdf", width = 11*2, height = 8*2, units = "in") 
ggarrange(plotlist = plots_neurofilaments, nrow = length(plots_neurofilaments)/4, ncol = 4) 
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK CLINICAL DATA: plot clinical variables, stratified by cluster-3.png)<!-- -->

```r
ggsave("plots/clinical_variables_clusters_neurofilaments.pdf", width = 11*2, height = 8*2, units = "in") 
ggarrange(plotlist = plots_progression, nrow = length(plots_progression)/4, ncol = 4) 
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK CLINICAL DATA: plot clinical variables, stratified by cluster-4.png)<!-- -->

```r
ggsave("plots/clinical_variables_clusters_progression.pdf", width = 11*2, height = 8*2, units = "in") 
ggarrange(plotlist = plots_onset, nrow = length(plots_onset)/4, ncol = 4) 
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/CHECK CLINICAL DATA: plot clinical variables, stratified by cluster-5.png)<!-- -->

```r
ggsave("plots/clinical_variables_clusters_onset.pdf", width = 11*2, height = 8*2, units = "in") 

pval_clin_adjust = p.adjust(pval_clin[1:73], method="BH")
write_csv(as.data.frame(pval_clin_adjust), "results/testing_clin_vars_padjust.csv")
```


## Sensitivity analyses 

With clustering, there is always the issue of the lack of a ground truth. Therefore, you don't know if you're modelling noise or if you're modelling actual biological processes. As a result, we have to try to figure out if the models are stable and robust, which would be an indication we're not just modeling noise. Below, I have performed a few sensitivity analyses, among them are Randomized Initialization analysis and Leave One Out Cross Validation

### Randomized initialization sensitivity analysis

Within some clustering algorithms, there is a randomization step involved. By setting the seed we can make sure that this randomization can be replicated every time we rerun the algorithm. However, choosing a different randomization initialization allows us to investigate how stable our models are. Here, I perform the clustering 10 times, with each time using a different seed. Through Sankey plots, we can compare how stable the models are across the 10 different randomizations.


```r
# Example usage:
# Replace 'assay_age_adj' and 'clin' with your actual data
# Replace 'YourSeed' with the desired seed value
results = list()
for(i in 1:10){
  results[[i]] <- perform_clustering(assay, i, clin_labels = clin$patid)
}

r = results[[1]]

for(i in 2:length(results)){
  r = cbind(r, results[[i]])
}

## Sankey plots within certain number of clusters, all four methods compared

      number_of_clusters = as.character(2:4)
      methods = c("kmeans", "hclust", "mclust", "pam")
      plots = list()
      k = 1
      
      for(i in 1:length(number_of_clusters)){
        for(j in 1:length(methods)){
        data = r[,c(1,grep(methods[j], colnames(r)))]
        data = data[,c(1, grep(paste0("k=",number_of_clusters[i]), colnames(data)))]
        
        data = reshape2::melt(data)
        colnames(data) = c("patid", "method", "cluster")
        data = as.data.frame(lapply(data, as.factor))
      
      plots[[k]] = ggplot(data,
               aes(x = method, stratum = cluster, alluvium = patid,
                   fill = cluster, label = cluster)) +
          scale_fill_brewer(type = "qual", palette = "Set3") +
          geom_flow(stat = "alluvium", lode.guidance = "frontback",
                    color = "darkgray") +
          geom_stratum() +
          theme(legend.position = "bottom") +
          ggtitle(paste0("How patients are clustered in different clustering algorithms \nk=", number_of_clusters[i], " ", methods[j])) +
          theme_few()
      k = k+1
      }}
```

```
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
```

```r
      allplots <- ggarrange(plotlist=plots,
                            labels = 1:length(plots),
                            ncol = 4, nrow = 3)
      ggsave("plots/Sankey_plots_between_methods_seed_exploration.pdf", width = 11*2, height = 8*2, units = "in") 
```

### Leave one out cross validation sensitivity analysis

Here, we remove a different patient each clustering analysis, to see how much the clustering changes. Ideally, the clustering models are robust against the removal of one single patient.


```r
#remove one patient and watch how it changes the clustering
results = list()
for(i in 1:10){
  set.seed(i)
  # Randomly select a column index
  row_index <- sample(seq_len(nrow(assay)), size = 1)
  # Remove the selected column
  data <- assay[-row_index, ]
  clin_data = clin[-row_index,]
  results[[i]] <- perform_clustering(data, seed = 9, clin_labels = clin_data$patid)
}

r = results[[1]]

for(i in 2:length(results)){
  # Add "-2" to column names of mat2
  rr = results[[i]]
  colnames(rr)[2:length(colnames(rr))] <- paste(colnames(rr)[2:length(colnames(rr))], "-", i, sep = "")
  r = merge(r, rr, by = "patid", all = TRUE)
}


## Sankey plots within certain number of clusters, all four methods compared

      number_of_clusters = as.character(2:4)
      methods = c("kmeans", "hclust", "mclust", "pam")
      plots = list()
      k = 1
      
      for(i in 1:length(number_of_clusters)){
        for(j in 1:length(methods)){
        data = r[,c(1,grep(methods[j], colnames(r)))]
        data = data[,c(1, grep(paste0("k=",number_of_clusters[i]), colnames(data)))]
        
        data = reshape2::melt(data)
        colnames(data) = c("patid", "method", "cluster")
        data = as.data.frame(lapply(data, as.factor))
      
      plots[[k]] = ggplot(data,
               aes(x = method, stratum = cluster, alluvium = patid,
                   fill = cluster, label = cluster)) +
          scale_fill_brewer(type = "qual", palette = "Set3") +
          geom_flow(stat = "alluvium", lode.guidance = "frontback",
                    color = "darkgray") +
          geom_stratum() +
          theme(legend.position = "bottom") +
          ggtitle(paste0("How patients are clustered in different clustering algorithms \nk=", number_of_clusters[i], " ", methods[j]), "\nsensitivity analysis leave one out") +
          theme_few()
      k = k+1
      }}
```

```
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
```

```r
      allplots <- ggarrange(plotlist=plots,
                            labels = 1:length(plots),
                            ncol = 4, nrow = 3)
      ggsave("plots/Sankey_plots_between_methods_leave_one_out.pdf", width = 11*2, height = 8*2, units = "in") 
```



## Tam et al paper comparison

Tam et al have published a paper on clustering in ALS patients as well. However, they have used transcriptomics in brain tissue instead of proteomics in CSF. In their paper, they report on genes that are significantly associated with their found subgroups. I have put these genes in the 'Tam_et_al_genes.xlsx' file in the data folder. In the following chunks, we will investigate whether their reported genes are also associated with our clusters. However, due to that we are working with a completely different modality the translatability is quite poor. Only a very small proportion of their reported genes we can find in our gene list as well.

### Tam et al Violin plots


```r
library(readxl)
Tam_genes <- readxl::read_xlsx("data/Tam_et_al_genes.xlsx", sheet = "Table S2A")
```

```
## New names:
## • `` -> `...2`
## • `` -> `...3`
## • `` -> `...4`
```

```r
colnames(Tam_genes) = Tam_genes[1,]
Tam_genes = Tam_genes[-1,]
plots = list()

# Define expression matrix, cluster assignments, and protein groups
expression_matrix <- as.data.frame(scale(assay))
expression_matrix = expression_matrix[, colnames(expression_matrix) %in% Tam_genes$Gene]


Tam_genes = Tam_genes[Tam_genes$Gene %in% colnames(expression_matrix),]
ALS_TE = Tam_genes$Gene[Tam_genes$Subtype == "ALS-TE"]
ALS_Glia = Tam_genes$Gene[Tam_genes$Subtype == "ALS-Glia"]
ALS_Ox = Tam_genes$Gene[Tam_genes$Subtype == "ALS-Ox"]

em_ALS_TE = expression_matrix[,ALS_TE]
em_ALS_Glia = expression_matrix[,ALS_Glia]
em_ALS_Ox = expression_matrix[,ALS_Ox]


# Convert to long format using gather (tidyr function)
em_ALS_TE_long = gather(em_ALS_TE)
em_ALS_TE_long$patid = rep(rownames(em_ALS_TE), ncol(em_ALS_TE))
em_ALS_TE_long$value = as.numeric(em_ALS_TE_long$value)

em_ALS_Glia_long = gather(em_ALS_Glia)
em_ALS_Glia_long$patid = rep(rownames(em_ALS_Glia), ncol(em_ALS_Glia))
em_ALS_Glia_long$value = as.numeric(em_ALS_Glia_long$value)

em_ALS_Ox_long = gather(em_ALS_Ox)
em_ALS_Ox_long$patid = rep(rownames(em_ALS_Ox), ncol(em_ALS_Ox))
em_ALS_Ox_long$value = as.numeric(em_ALS_Ox_long$value)

comp <- list(c("1", "0"), c("0", "2"), c("1", "2"))

cluster_assignments = cluster_assignments[,1:9]

# a violin plot that plots the values of OUR expression matrix but only uses proteins that were also mentioned in the Tam et al paper
#this violin plot only plots the MEANS of all proteins that belong to the same subgroup according to the Tam et al paper
for(i in 2:ncol(cluster_assignments)){
    title = colnames(cluster_assignments)[i]
    if(grepl("k=2", title)){c = comp[1]}else{c = comp}
    cluster_assignment <- cluster_assignments[,i]
    em_ALS_TE_long$cluster = as.factor(rep(cluster_assignment, nrow(em_ALS_TE_long)/length(cluster_assignment)))
    em_ALS_Glia_long$cluster = as.factor(rep(cluster_assignment, nrow(em_ALS_Glia_long)/length(cluster_assignment)))
    em_ALS_Ox_long$cluster = as.factor(rep(cluster_assignment, nrow(em_ALS_Ox_long)/length(cluster_assignment)))
    
    data = list(ALS_TE = em_ALS_TE_long, 
                ALS_Glia  = em_ALS_Glia_long, 
                ALS_Ox = em_ALS_Ox_long)
    
    
    for(j in 1:length(data)){
      plots[[j]] = ggviolin(data[[j]], x = "cluster", y = "value", fill = "cluster", add = "boxplot", facet.by = "key") +
                    labs(title = paste0("Boxplot of Protein Expression by Cluster for ", title, " ", names(data)[j]),
                         x = "Protein",
                         y = "Expression") +
                    theme_few() +
                    theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
                    stat_compare_means(comparisons = c, method = "wilcox.test", 
                                       label = "p.signif", size = 3, color = "black", 
                                       label.y = 1.25, bracket.size = 0) 
       
    }
    ggarrange(plotlist = plots, ncol = 3, nrow = 1)
    ggsave(filename = paste0("plots/boxplots_Tam_paper_", title, ".pdf"), width = 12*2, height = 12, units = "in")
}

plots = list()
p = 1

# a violin plot that plots the values of OUR expression matrix but only uses proteins that were also mentioned in the Tam et al paper
#this violin plots each protein individually, again only proteins that were mentioned in the Tam et al paper
for(i in 2:ncol(cluster_assignments)){
    title = colnames(cluster_assignments)[i]
    if(grepl("k=2", title)){c = comp[1]}else{c = comp}
    cluster_assignment <- cluster_assignments[,i]
    em_ALS_TE_long$cluster = as.factor(rep(cluster_assignment, nrow(em_ALS_TE_long)/length(cluster_assignment)))
    em_ALS_Glia_long$cluster = as.factor(rep(cluster_assignment, nrow(em_ALS_Glia_long)/length(cluster_assignment)))
    em_ALS_Ox_long$cluster = as.factor(rep(cluster_assignment, nrow(em_ALS_Ox_long)/length(cluster_assignment)))
    
    data = list(ALS_TE = em_ALS_TE_long, 
                ALS_Glia  = em_ALS_Glia_long, 
                ALS_Ox = em_ALS_Ox_long)
    
    for(j in 1:length(data)){
      plots[[p]] = ggviolin(data[[j]], x = "cluster", y = "value", fill = "cluster", add = "boxplot") +
                    labs(title = paste0(title, " ", names(data)[j]),
                         x = "Cluster",
                         y = "Expression") +
                    theme_few() +
                    theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
                    stat_compare_means(comparisons = c, method = "wilcox.test", 
                                       label = "p.signif", size = 3, color = "black", 
                                       label.y = 1.25, bracket.size = 0) 
      p = p+1  
    }

}

    ggarrange(plotlist = plots, ncol = 3, nrow = 8)
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: Violin plots-1.png)<!-- -->

```r
    ggsave(filename = "plots/boxplots_Tam_paper_Lauras_approach.pdf", width = 12, height = 24, units = "in")
    
    library("xlsx")
    write.xlsx(cluster_assignments, file = "results/cluster_assignments.xlsx")
    write.xlsx(assay, file = "results/expression_matrix.xlsx")
```

### Tam et al Heatmap

With this section, we plot our expression matrix as a heatmap after filtering for proteins that were mentioned in the Tam et al paper.


```r
library(pheatmap)

save_pheatmap_pdf <- function(x, filename, width=7, height=7) {
   stopifnot(!missing(x))
   stopifnot(!missing(filename))
   pdf(filename, width=width, height=height)
   grid::grid.newpage()
   grid::grid.draw(x$gtable)
   dev.off()
}

Tam_genes2 = Tam_genes[,c(1,3)]
Tam_genes2$Subtype = factor(Tam_genes2$Subtype, ordered = T)
sorted_tam_genes <- Tam_genes2[order(Tam_genes2[,"Subtype"]), ]
```

```
## Warning in xtfrm.data.frame(x): cannot xtfrm data frames
```

```r
for(i in 2:ncol(cluster_assignments)){
      set.seed(9)
      title = colnames(cluster_assignments)[i]
      assign = cluster_assignments[,c(1,i)]
      sorted_assign <- assign[order(assign[,2]), ]
      assay2 = assay[sorted_assign$patid, sorted_tam_genes$Gene]
      assay2 = scale(assay2)
      assay2 = as.data.frame(t(assay2))
      
      
      # Create row and column annotations
      col_annotations <- data.frame(
        Cluster_assignment = as.character(sorted_assign[,2])
      )
      rownames(col_annotations) = sorted_assign$patid
      
      row_annotations <- data.frame(
        Tam_Subtypes = sorted_tam_genes$Subtype
      )
      rownames(row_annotations) = sorted_tam_genes$Gene
      
      # Create a not-clustered pheatmap with annotations
      p = pheatmap(
        assay2,
        annotation_row = row_annotations,
        annotation_col = col_annotations,
        cluster_rows = FALSE,
        cluster_cols = FALSE,
        fontsize = 8,  # Adjust the font size if needed
        border_color = NA, 
        main = title
      )
      save_pheatmap_pdf(p, paste0("plots/heatmap_Tam_", title ,"v2.pdf"))
      
      # Create a not-clustered pheatmap with annotations, adjusted colour scale
      set.seed(9)
      
      assay4 = assay2
      assay4[assay4>3] = 3
      assay4[assay4<(-3)] = -3

      
      p = pheatmap(
        assay4,
        annotation_row = row_annotations,
        annotation_col = col_annotations,
        cluster_rows = FALSE,
        cluster_cols = FALSE,
        fontsize = 8,  # Adjust the font size if needed
        border_color = NA, 
        main = title
      )
      save_pheatmap_pdf(p, paste0("plots/heatmap_Tam_", title ,"v2_colscale_cap3.pdf"))
      
      set.seed(9)
      
       # Create a clustered pheatmap with annotations,  adjusted colour scale (capped at 3)
      p = pheatmap(
        assay4,
        annotation_row = row_annotations,
        annotation_col = col_annotations,
        cluster_rows = FALSE,
        cluster_cols = TRUE,
        fontsize = 8,  # Adjust the font size if needed
        border_color = NA, 
        main = title
      )
      save_pheatmap_pdf(p, paste0("plots/heatmap_Tam_", title ,"_clustered_colscale_cap3.pdf"))
      
      # Create a clustered pheatmap with annotations,  adjusted colour scale (capped at 3)
      p = pheatmap(
        assay4,
        annotation_row = row_annotations,
        annotation_col = col_annotations,
        cluster_rows = TRUE,
        cluster_cols = TRUE,
        fontsize = 8,  # Adjust the font size if needed
        border_color = NA, 
        main = title
      )
      save_pheatmap_pdf(p, paste0("plots/heatmap_Tam_", title ,"_double_clustered_colscale_cap3.pdf"))
      
            # Create a clustered pheatmap with annotations,  adjusted colour scale (capped at 3)
      p = pheatmap(
        assay4,
        annotation_row = row_annotations,
        annotation_col = col_annotations,
        cluster_rows = TRUE,
        cluster_cols = FALSE,
        fontsize = 8,  # Adjust the font size if needed
        border_color = NA, 
        main = title
      )
      save_pheatmap_pdf(p, paste0("plots/heatmap_Tam_", title ,"_rowwise_clustered_colscale_cap3.pdf"))

}
```

![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-1.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-2.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-3.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-4.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-5.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-6.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-7.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-8.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-9.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-10.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-11.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-12.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-13.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-14.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-15.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-16.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-17.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-18.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-19.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-20.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-21.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-22.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-23.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-24.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-25.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-26.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-27.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-28.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-29.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-30.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-31.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-32.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-33.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-34.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-35.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-36.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-37.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-38.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-39.png)<!-- -->![](MOCK_Proteomics_MAXOMOD_clustering_files/figure-html/TAM PAPER CHECK GENES: heatmap Tam et al genes-40.png)<!-- -->


### Tam et al DEx analysis

With this section, we perform differential expression analysis using a subselection of the expression matrix, selecting only proteins that are mentioned by the Tam et al paper. We test our clusters against eachother. Ideally, if we find significant results, there is some overlap between our clusters and the ones reported by Tam et al.


```r
      set.seed(9)
      l = 1
      res = list()

      covariates = "age_cov"
      covariates_f = ~0 + condition + age_cat

for(k in 2:ncol(cluster_assignments)){
  #make summarized experiment, this time with cluster as condition
      assay2 = assay
      assay2 = assay2[,Tam_genes2$Gene]
      clin$cluster = as.factor(cluster_assignments[,k])
      levels(clin$cluster) = c("alpha", "beta", "theta")[1:length(levels(clin$cluster))]
      assay2 = as.data.frame(t(assay2))
      assay2$ID = assay2$name = rownames(assay2)
      abundance.columns <- grep("patient", colnames(assay2)) # get abundance column numbers
      experimental.design = clin[, c("patid","cluster", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")]
      colnames(experimental.design) = c("label","condition", "onset", "age", "sex", "neurofilaments", "genetics", "age_at_onset", "progression_rate", "age_cat")
      experimental.design$replicate = 1:nrow(experimental.design)
      se_abu_data_ALS <- make_se(assay2, abundance.columns, experimental.design)

      set.seed(9)
      title = paste0(colnames(cluster_assignments)[k],"_",covariates)
      d = se_abu_data_ALS
      control = "beta"
      print(title)
      print(dim(d))
      t = test_diff(d, type = "all", control = control,
            test = NULL, design_formula = formula(covariates_f))
      res[[l]] = as.data.frame(t@elementMetadata@listData)
      pval_index = grep("p.val",colnames(res[[l]]))
      if(length(pval_index)==1){
        res[[l]]$fdr = p.adjust(res[[l]][,pval_index], method="BH")
      }else{
        names = colnames(res[[l]])[pval_index]
        names = gsub("p.val", "fdr", names)
        res[[l]][,names[1]] = p.adjust(res[[l]][,pval_index[1]], method="BH")
        res[[l]][,names[2]] = p.adjust(res[[l]][,pval_index[2]], method="BH")
        res[[l]][,names[3]] = p.adjust(res[[l]][,pval_index[3]], method="BH")
      }
      names(res)[l] = title
      l = l+1
}
```

```
## [1] "hclust_k=2_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "mclust_k=2_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "kmeans_k=2_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "pam_k=2_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "hclust_k=3_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta, alpha_vs_theta, theta_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "mclust_k=3_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta, alpha_vs_theta, theta_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "kmeans_k=3_age_cov"
## [1] 52 50
```

```
## Tested contrasts: theta_vs_beta, theta_vs_alpha, alpha_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```
## [1] "pam_k=3_age_cov"
## [1] 52 50
```

```
## Tested contrasts: alpha_vs_beta, alpha_vs_theta, theta_vs_beta
```

```
## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!

## Warning in fdrtool::fdrtool(res$t, plot = FALSE, verbose = FALSE): There may be
## too few input test statistics for reliable FDR calculations!
```

```r
saveRDS(res, file = "results/TAM_GENES_Dex_results_all_in_list.rds")      
```

### Tam et al Volcano plot


```r
volcano_plot <- function(df, alpha_sig, name_title){
  df <- df %>%
    mutate(omic_type = case_when(x >= 0 & y >= (-log10(alpha_sig)) ~ "up",
                                 x <= (0) & y >= (-log10(alpha_sig)) ~ "down",
                                 TRUE ~ "ns")) 
  cols <- c("up" = "#d4552b", "down" = "#26b3ff", "ns" = "grey") 
  sizes <- c("up" = 2, "down" = 2, "ns" = 1) 
  alphas <- c("up" = 0.7, "down" = 0.7, "ns" = 0.5)
  ggplot(data = df, aes(x,y)) + 
    geom_point(aes(colour = omic_type), 
               alpha = 0.5, 
               shape = 16,
               size = 3) + 
    geom_hline(yintercept = -log10(alpha_sig),
               linetype = "dashed") + 
    geom_vline(xintercept = 0,linetype = "dashed") +
    geom_point(data = filter(df, y >= (-log10(alpha_sig))),
               aes(colour = omic_type), 
               alpha = 0.5, 
               shape = 16,
               size = 4) + 
    #annotate(geom="text", x=-1.9, y= (-log10(alpha_sig)) + 0.15, label="FDR = 10%",size = 5) +
    geom_text_repel(data = filter(df, y >= (-log10(alpha_sig)) & y > 0),
                     aes(label = name),
                     force = 1,
                    #hjust = 1,
                    #nudge_x = - 0.3,
                    #nudge_y = 0.1,
                    #direction = "x",
                     max.overlaps = 20,
                    segment.size = 0.2,
                     size = 4) +
    geom_text_repel(data = filter(df, y >= (-log10(alpha_sig)) & y < 0),
                    aes(label = name),
                    force = 1,
                    #hjust = 0,
                    #nudge_x = 0.3,
                    #nudge_y = 0.1,
                    #direction = "y",
                    max.overlaps = 20,
                    size = 4) +
    scale_colour_manual(values = cols) + 
    scale_fill_manual(values = cols) + 
    scale_x_continuous(expand = c(0, 0), 
                       limits = c(-0.25, 0.25)) + 
    scale_y_continuous(expand = c(0, 0), limits = c(-0.1, NA)) +
    labs(title = name_title,
         x = "log2(fold change)",
         y = expression(-log[10] ~ "(adjusted p-value)"),
         colour = "Differential \nExpression") +
    theme_classic() + # Select theme with a white background  
    theme(axis.title.y = element_text(size = 14),
          axis.title.x = element_text(size = 14),
          axis.text = element_text(size = 12),
          plot.title = element_text(size = 15, hjust = 0.5),
          text = element_text(size = 14)) +
    annotate("text", x = 0.1, y = 0.5, label = paste0(sum(df$omic_type=="up"), " more abundant \n", sum(df$omic_type=="down"), " less abundant"))
}

plots_FDR0.05 = plots_FDR0.1 = list()
l = 1

for(i in 1:length(res)){
    data_res = res[[i]]
    diff_index = grep("diff",colnames(data_res))
    fdr_index = grep("fdr",colnames(data_res))
    if(length(diff_index) == 1){
      title = paste0(names(res)[i], "_alpha_vs_beta")
      logFC = data_res[,diff_index]
      fdr = data_res[,fdr_index]
      df <- data.frame(x = logFC, 
                         y = -log10(fdr),
                         name = data_res$name)
      names(df) <- c("x","y","name")
      plots_FDR0.05[[l]] = volcano_plot(df, 0.05 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", title))
      plots_FDR0.1[[l]] = volcano_plot(df, 0.1 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", title))
      names(plots_FDR0.05)[l] = names(plots_FDR0.1)[l] = title
      l = l+1
    }else{
      names = colnames(data_res)[diff_index]
      names = gsub("_diff", "", names)
      for(j in 1:length(names)){
        title = paste0(names(res)[i], "_", names[j])
        logFC = data_res[,diff_index[j]]
        fdr = data_res[,fdr_index[j]]
        df <- data.frame(x = logFC, 
                           y = -log10(fdr),
                           name = data_res$name)
        names(df) <- c("x","y","name")
        plots_FDR0.05[[l]] = volcano_plot(df, 0.05 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.05\n", title))
        plots_FDR0.1[[l]] = volcano_plot(df, 0.1 , paste0("Volcano plot clustering proteomics \nalpha = FDR 0.1\n", title))
        names(plots_FDR0.05)[l] = names(plots_FDR0.1)[l] = title
        l = l+1
      }
    }
    
}

k2_plots = plots_FDR0.05[grep("k=2", names(plots_FDR0.05))]
k3_plots = plots_FDR0.05[grep("k=3", names(plots_FDR0.05))]

plot = ggarrange(plotlist=k2_plots, ncol = 1, nrow = 4)
```

```
## Warning: Removed 33 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 38 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 8 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 8 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 38 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 8 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 8 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 33 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_text_repel()`).
```

```r
ggsave("plots/TAM_GENES_K2_volcano_plots_FDR0.05.pdf", width = 7, height = 21, units = "in") 

plot = ggarrange(plotlist=k3_plots, ncol = 3, nrow = 4)
```

```
## Warning: Removed 34 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 42 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 38 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 34 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 42 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 38 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 38 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 42 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 34 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 34 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 4 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 42 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 10 rows containing missing values (`geom_text_repel()`).
```

```
## Warning: Removed 38 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_point()`).
```

```
## Warning: Removed 7 rows containing missing values (`geom_text_repel()`).
```

```r
ggsave("plots/TAM_GENES_K3_volcano_plots_FDR0.05.pdf", width = 7*3, height = 21, units = "in") 
```


## Age adjustment analyses

To rule out that we are not modelling age in our clustering models, we adjust the dataset for age and see if it changes our clustering model significantly. The following 5 chunks of code all do different analyses or visualizations of age-adjusted analyses. This approach is not very common and this was more of an experiment. 


```r
# Assume 'clinical_variable' is a vector of clinical variable values
# Assume 'protein_matrix' is a matrix where each column represents a protein and each row represents an observation

# Load necessary libraries
library(stats)

# Function to perform Fisher correlation and adjust p-values
perform_fisher_correlation <- function(clinical_variable, protein_matrix) {
  
  # Number of proteins
  num_proteins <- ncol(protein_matrix)
  
  # Initialize vectors to store correlation coefficients and p-values
  correlation_coefficients <- numeric(num_proteins)
  p_values <- numeric(num_proteins)
  
  # Perform Fisher correlation for each protein
  for (i in 1:num_proteins) {
    protein <- protein_matrix[, i]
    
    # Perform Fisher correlation test
    cor_test_result <- cor.test(clinical_variable, protein, method = "pearson")
    
    # Store correlation coefficient and raw p-value
    correlation_coefficients[i] <- cor_test_result$estimate
    p_values[i] <- cor_test_result$p.value
  }
  
  # Adjust p-values using Benjamini-Hochberg method
  adjusted_p_values <- p.adjust(p_values, method = "BH")
  
  # Create a data frame with results
  results_df <- data.frame(
    Protein = colnames(protein_matrix),
    CorrelationCoefficient = correlation_coefficients,
    RawPValue = p_values,
    AdjustedPValue = adjusted_p_values
  )
  
  return(results_df)
}

# Example usage:
# Replace 'clinical_variable' and 'protein_matrix' with your actual data
# Replace 'YourClinicalVariable' with the actual name of your clinical variable


# Perform Fisher correlation and adjust p-values
results <- perform_fisher_correlation(clin$age, assay)
```


```r
non_sig_age_proteins = results$Protein[results$AdjustedPValue>=0.05]
assay_age_adj = assay[,non_sig_age_proteins]

set.seed(9)

      library(cluster)
      
      cluster_assignments_age_adj = as.data.frame(clin$patid)
      colnames(cluster_assignments_age_adj) = "patid"
      silhouette_scores_age_adj = TWSS_scores_age_adj = AIC_scores_age_adj = BIC_scores_age_adj = as.data.frame(matrix())
      rownames(silhouette_scores_age_adj) = rownames(TWSS_scores_age_adj) = rownames(AIC_scores_age_adj) = rownames(BIC_scores_age_adj) = "hclust"

#PERFORM ALL THE CLUSTERING      
      
      #perform the clustering with trying cluster numbers 1-10
      for(i in 2:10){
    
# Hierarchical Clustering:
    
  #with hclust function
    # hclust: Performs hierarchical clustering.
    title = paste0("hclust_k=", i)
    #performing the clustering
    dist_mat <- dist(assay_age_adj, method = 'euclidean')
    cl <- hclust(dist_mat, method = 'ward.D')
    cluster_assignments_age_adj[,title] <- cutree(cl, k = i)
    
    
    
    #cluster fit measures
    ss = silhouette(cluster_assignments_age_adj[,title], dist(assay_age_adj))
    silhouette_scores_age_adj["hclust", i] = mean(ss[, 3])
    TWSS_scores_age_adj["hclust", i] = calc_TWSS(assay_age_adj, cluster_assignments_age_adj[,title])
    AIC_scores_age_adj["hclust", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,1]
    BIC_scores_age_adj["hclust", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,2]
    

# Model-Based Clustering:
# Mclust: Fits Gaussian finite mixture models for model-based clustering.
    
    title = paste0("mclust_k=", i)
    
    #performing the clustering
    library(mclust)
    cl <- Mclust(assay_age_adj, G = i)
    cluster_assignments_age_adj[,title] <- cl$classification

    #cluster fit measures
    ss = silhouette(cluster_assignments_age_adj[,title], dist(assay_age_adj))
    silhouette_scores_age_adj["mclust", i] = mean(ss[, 3])
    TWSS_scores_age_adj["mclust", i] = calc_TWSS(assay_age_adj, cluster_assignments_age_adj[,title])
    AIC_scores_age_adj["mclust", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,1]
    BIC_scores_age_adj["mclust", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,2]    
    
# K-Means Clustering:
      
    # kmeans: Performs k-means clustering.
    title = paste0("kmeans_k=", i)
    
    #performing the clustering
    cl <- kmeans(assay_age_adj, centers = i)
    cluster_assignments_age_adj[,title] = cl$cluster
    
    #cluster fit measures
    ss = silhouette(cluster_assignments_age_adj[,title], dist(assay_age_adj))
    silhouette_scores_age_adj["kmeans", i] = mean(ss[, 3])
    TWSS_scores_age_adj["kmeans", i] = calc_TWSS(assay_age_adj, cluster_assignments_age_adj[,title])
    AIC_scores_age_adj["kmeans", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,1]
    BIC_scores_age_adj["kmeans", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,2]

# Partitioning Around Medoids:
# pam: Performs Partitioning Around Medoids (PAM) clustering.
    title = paste0("pam_k=", i)
    
    #performing the clustering
    cl <- pam(assay_age_adj, k = i)
    cluster_assignments_age_adj[,title] <- cl$clustering

    #cluster fit measures
    ss = silhouette(cluster_assignments_age_adj[,title], dist(assay_age_adj))
    silhouette_scores_age_adj["pam", i] = mean(ss[, 3])
    TWSS_scores_age_adj["pam", i] = calc_TWSS(assay_age_adj, cluster_assignments_age_adj[,title])
    AIC_scores_age_adj["pam", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,1]
    BIC_scores_age_adj["pam", i] = BIC2(assay_age_adj, cluster_assignments_age_adj[,title])[1,2]
  
      }
      
cluster_assignments_age_adj[,2:ncol(cluster_assignments_age_adj)] = cluster_assignments_age_adj[,2:ncol(cluster_assignments_age_adj)]-1
```


```r
# set seed for reproducible results
set.seed(9)
library(RColorBrewer)
library(ggpubr)
colour_set = brewer.pal(10, "Set3")
colour_set[2] = "lightgoldenrod2"
      
      UMAP_plots = list()
      
      for(i in 2:ncol(cluster_assignments_age_adj)){
        title = colnames(cluster_assignments_age_adj)[i]
        labels = as.factor(cluster_assignments_age_adj[,i])
        names(labels) = cluster_assignments_age_adj[,1]
        colour_subset = colour_set[1:length(levels(labels))]
      
#perform plots with function      

      UMAP_plots[[i-1]] = UMAP_density_plot(data = assay_age_adj, 
                                ggtitle = paste0("UMAP with cluster labels\n", title, "\nage adjusted"), 
                                legend_name = "Cluster labels", 
                                labels = labels, 
                                colour_set = colour_subset)
      names(UMAP_plots)[i-1] = title
      }
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
## no non-missing arguments to max; returning -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
## no non-missing arguments to max; returning -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```
## Warning: Groups with fewer than two data points have been dropped.
## no non-missing arguments to max; returning -Inf
```

```
## Warning: Groups with fewer than two data points have been dropped.
```

```
## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning
## -Inf
```

```
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
## Scale for colour is already present.
## Adding another scale for colour, which will replace the existing scale.
```

```r
      UMAP_plots_subset = UMAP_plots[1:(6*4)] #only take until k=7
      
      allplots <- ggarrange(plotlist=UMAP_plots_subset,
                            labels = 1:length(UMAP_plots_subset),
                            ncol = 4, nrow = 6)
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

```
## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.

## Warning in as_grob.default(plot): Cannot convert object of class character into
## a grob.
```

```r
      ggsave("plots/UMAPs_age_adj.pdf", width = 11*2, height = 8*2, units = "in") 
```


```r
set.seed(9)

scores = list(AIC = AIC_scores_age_adj, BIC = BIC_scores_age_adj, silhouette = silhouette_scores_age_adj, TWSS = TWSS_scores_age_adj)
plots = list()

for(i in 1:length(scores)){
  colnames(scores[[i]]) = 1:10
  scores[[i]]$method = rownames(scores[[i]])
  melt = melt(scores[[i]])
  
  plots[[i]] = ggplot(melt, aes(x=variable, y=value, group = method, colour = method)) +
    geom_line() +
    geom_point() +
    ggtitle(paste0(names(scores)[i], " age adjusted"))
  
}
```

```
## Warning in melt(scores[[i]]): The melt generic in data.table has been passed a
## data.frame and will attempt to redirect to the relevant reshape2 method; please
## note that reshape2 is deprecated, and this redirection is now deprecated as
## well. To continue using melt methods from reshape2 while both libraries are
## attached, e.g. melt.list, you can prepend the namespace like
## reshape2::melt(scores[[i]]). In the next version, this warning will become an
## error.
```

```
## Using 1, method as id variables
```

```
## Warning in melt(scores[[i]]): The melt generic in data.table has been passed a
## data.frame and will attempt to redirect to the relevant reshape2 method; please
## note that reshape2 is deprecated, and this redirection is now deprecated as
## well. To continue using melt methods from reshape2 while both libraries are
## attached, e.g. melt.list, you can prepend the namespace like
## reshape2::melt(scores[[i]]). In the next version, this warning will become an
## error.
```

```
## Using 1, method as id variables
```

```
## Warning in melt(scores[[i]]): The melt generic in data.table has been passed a
## data.frame and will attempt to redirect to the relevant reshape2 method; please
## note that reshape2 is deprecated, and this redirection is now deprecated as
## well. To continue using melt methods from reshape2 while both libraries are
## attached, e.g. melt.list, you can prepend the namespace like
## reshape2::melt(scores[[i]]). In the next version, this warning will become an
## error.
```

```
## Using 1, method as id variables
```

```
## Warning in melt(scores[[i]]): The melt generic in data.table has been passed a
## data.frame and will attempt to redirect to the relevant reshape2 method; please
## note that reshape2 is deprecated, and this redirection is now deprecated as
## well. To continue using melt methods from reshape2 while both libraries are
## attached, e.g. melt.list, you can prepend the namespace like
## reshape2::melt(scores[[i]]). In the next version, this warning will become an
## error.
```

```
## Using 1, method as id variables
```

```r
names(plots) = names(scores)

allplots <- ggarrange(plotlist=plots,
                      labels = LETTERS[1:length(plots)],
                      ncol = 2, nrow = 2)
ggsave("plots/fit_scores_age_adj.pdf", width = 11, height = 8, units = "in") #CONCLUSION --> 3:5 Clusters is ideal
```


```r
set.seed(9)

library(ggalluvial)
library(tidyverse)
    cluster_assignments_both = cbind(cluster_assignments, cluster_assignments_age_adj)

## Sankey plots within certain number of clusters, all four methods compared

      number_of_clusters = as.character(2:4)
      methods = c("kmeans", "hclust", "mclust", "pam")
      plots = list()
      k = 1
      
      for(i in 1:length(number_of_clusters)){
        for(j in 1:length(methods)){
        data = cluster_assignments_both[,c(1, grep(number_of_clusters[i], colnames(cluster_assignments_both)))]
        data = data[,c(1, grep(methods[j], colnames(data)))]
        data = reshape2::melt(data)
        colnames(data) = c("patid", "method", "cluster")
        data = as.data.frame(lapply(data, as.factor))
      
      plots[[k]] = ggplot(data,
               aes(x = method, stratum = cluster, alluvium = patid,
                   fill = cluster, label = cluster)) +
          scale_fill_brewer(type = "qual", palette = "Set3") +
          geom_flow(stat = "alluvium", lode.guidance = "frontback",
                    color = "darkgray") +
          geom_stratum() +
          theme(legend.position = "bottom") +
          ggtitle(paste0("How patients are clustered in different clustering algorithms \nk=",methods[j], number_of_clusters[i], "\n age adjusted")) +
          theme_few()
      k = k+1
      }}
```

```
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
## Using patid as id variables
```

```r
      allplots <- ggarrange(plotlist=plots,
                            labels = 1:length(plots),
                            ncol = 4, nrow = 3)
      ggsave("plots/Sankey_plots_between_methods_age_adj.pdf", width = 11*2, height = 8*2, units = "in") 


# Sankey plots within a method, different number of clusters

    methods = c("kmeans", "hclust", "mclust", "pam")
    plots = list()

      
      for(i in 1:length(methods)){
        data = cluster_assignments_both[,c(1, grep(methods[i], colnames(cluster_assignments_both)))]
        data = reshape2::melt(data, id = "patid")
        colnames(data) = c("patid", "method", "cluster")
        data = as.data.frame(lapply(data, as.factor))
      
      plots[[i]] = ggplot(data,
               aes(x = method, stratum = cluster, alluvium = patid,
                   fill = cluster, label = cluster)) +
          scale_fill_brewer(type = "qual", palette = "Set3") +
          geom_flow(stat = "alluvium", lode.guidance = "frontback",
                    color = "darkgray") +
          geom_stratum() +
          theme(legend.position = "bottom") +
          ggtitle(paste0("How patients are clustered within one clustering algorithm \nmethod = ", methods[i], "\n age adjusted")) +
          theme_few() + 
          theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))
      }
      
      allplots <- ggarrange(plotlist=plots,
                            labels = 1:length(plots),
                            ncol = 2, nrow = 2)
      ggsave("plots/Sankey_plots_within_method_age_adj.pdf", width = 11*2, height = 8*2, units = "in") 
```



## Supervised learning to classify patients into clusters

We tried to build a supervised classifier to classify the patients in the correct cluster. However, we experienced big issues with overfitting, and did not follow up on the analysis. It makes sense that we experienced overfitting since we based the clusters on the same data as we use to train the model. Therefore, supervised learning in general is not very suitable to apply to the clustering analysis. Even with the validation cohort, using a supervised model would be problematic from a statistical point of view, since there is no such thing as a 'ground truth'. Instead, we should focus more on the stability of the clusters, and how much the clusters from the discovery cohort and the validation cohort overlap. However, I included the code below that it can be recycled for other purposes.


```r
# # Install and load necessary packages if not already installed
# # install.packages("caret")
# # install.packages("pROC")
# # install.packages("ggplot2")
# 
# library(caret)
# library(pROC)
# library(ggplot2)
# 
# run_bootstrap_lasso <- function(expression_matrix, disease_status, num_bootstrap, num_folds, method) {
#   results <- list(AUC = numeric(num_bootstrap), lasso_weights = list(), ROC = list(), models = list())
#   
#   for (b in 1:num_bootstrap) {
#     
#     # Create an index for stratified sampling
#     strat_index <- createDataPartition(disease_status, p = 0.8, list = FALSE, times = 1)
# 
#     # Split the data into training and testing sets using the stratified index
#     train_data <- expression_matrix[strat_index, ]
#     train_labels <- disease_status[strat_index]
#     
#     test_data <- expression_matrix[-strat_index, ]
#     test_labels <- disease_status[-strat_index]
#     
#     # Train the model on the training set
#     ctrl = trainControl(method="cv",   
#                               number = num_folds,        
#                               summaryFunction=twoClassSummary,   # Use AUC to pick the best model
#                               classProbs=TRUE,savePredictions = TRUE)
#     model <- train(
#       x = train_data, 
#       y = train_labels, 
#       method = method, #"glmnet" or #"multinom"
#       trControl = ctrl,
#       tuneGrid = expand.grid(alpha = 1, lambda = seq(0.001, 0.1, length = 10))
#     )
#     
#     # Predict on the test set
#     predictions <- predict(model, newdata = test_data, type = "prob")[,1]
#     
#     # Convert disease_status to a an ordered factor variable
#     test_labels <- factor(test_labels, ordered = T)
#     
#     
#     results$AUC[b] <- roc(predictor = predictions, response = test_labels)$auc
#     results$ROC[[b]] <- roc(predictor = predictions, response = test_labels)
#     results$lasso_weights[[b]] <- coef(model$finalModel, s = model$bestTune$lambda)
#     results$models[[b]] = model
#   }
#   return(results)
# }
# 
# run_bootstrap_svm <- function(expression_matrix, disease_status, num_bootstrap, num_folds, method) {
#     results <- list(AUC = numeric(num_bootstrap), svm_weights = list(), ROC = list(), models = list())
# 
#     for (b in 1:num_bootstrap) {
# 
#         # Create an index for stratified sampling
#         strat_index <- createDataPartition(disease_status, p = 0.8, list = FALSE, times = 1)
# 
#         # Split the data into training and testing sets using the stratified index
#         train_data <- expression_matrix[strat_index, ]
#         train_labels <- disease_status[strat_index]
# 
#         test_data <- expression_matrix[-strat_index, ]
#         test_labels <- disease_status[-strat_index]
# 
#         # Train the model on the training set
#         ctrl <- trainControl(method = "cv",
#                             number = num_folds,
#                             summaryFunction = twoClassSummary,
#                             classProbs = TRUE,
#                             savePredictions = TRUE)
# 
#         model <- train(
#             x = train_data,
#             y = train_labels,
#             method = method, #"svmLinear" or "svmPoly"
#             trControl = ctrl
#         )
# 
#         # Predict on the test set
#         predictions <- predict(model, newdata = test_data, type = "prob")[, 1]
# 
#         # Convert disease_status to an ordered factor variable
#         test_labels <- factor(test_labels, ordered = T)
#         
#         #model weights
#         coef = model$finalModel@coef[[1]]
#         matr = model$finalModel@xmatrix[[1]]
#         weig = as.data.frame(coef %*% matr)
# 
#         # Calculate AUC and store the results
#         results$AUC[b] <- roc(predictor = predictions, response = test_labels)$auc
#         results$ROC[[b]] <- roc(predictor = predictions, response = test_labels)
#         results$svm_weights[[b]] <- weig
#         results$models[[b]] = model
#     }
# 
#     return(results)
# }
# 
# 
# # Function to plot AUC curve
# plot_auc_curve <- function(results, title) {
#         # Calculate mean AUC and confidence interval
#       rocc = data.frame()
#       for(i in 1:length(results$ROC)){
#         roc_run = results$ROC[[i]]
#         rocc = rbind(rocc, data.frame(Sp = roc_run$specificities, Sn = roc_run$sensitivities, n = rep(1:length(roc_run$sensitivities))))
#       }
#       
#       # aggregate the results and create new data frame
#       Sp = aggregate(Sp ~ n, rocc, mean)$Sp
#       Sn = aggregate(Sn ~ n, rocc, mean)$Sn
#       errorSp = aggregate(Sp ~ n, rocc, sd)$Sp
#       errorSn = aggregate(Sn ~ n, rocc, sd)$Sn
#       plotci = data.frame(Sp,Sn,errorSp,errorSn)
#       auc = results$AUC
#       
#         
#       auc_plot = ggplot(plotci, aes(x=(1-Sp),y=Sn)) + 
#            geom_line(aes(color = "aquamarine4"), linewidth = 2) + 
#            theme_few() +
#            ggtitle(paste0(title, "\nmean ROC curve and 95 % CI")) +
#            geom_ribbon(aes(ymin = (Sn - 0.95*errorSn), 
#                            ymax = (Sn + 0.95*errorSn), 
#                            xmin = (1-Sp - 0.95*errorSp), 
#                            xmax = (1-Sp + 0.95*errorSp),
#                            fill = "#B2B2B2"), 
#                        alpha = 0.5) +
#           
#             scale_color_manual(name = NULL, label = "mean", values = c("aquamarine4")) +
#             scale_fill_manual(name = NULL, label = "95 % CI", values = c('#B2B2B2') ) +
#             annotate("text", x = 0.2, y = 0.8, label = paste("mean AUC: ", round(mean(auc),2), "\u00B1 ", round(sd(auc),2)) ) +
#             geom_abline(slope = 1, color="darkgrey", alpha = 0.3)
#         
#         return(auc_plot)
# }
# 
# plot_lasso_weights <- function(lasso_weights, title, n){
# 
#       all_weights = as.data.frame(results$lasso_weights[[1]][,1])
#       for(i in 2:length(results$lasso_weights)){
#         all_weights = cbind(all_weights, as.data.frame(results$lasso_weights[[i]][,1]))
#       }
#       all_weights = all_weights[-1,]
#       all_weights[all_weights == 0] <- NA
#       
#       times_included = apply(all_weights, function(x) sum(!is.na(x)) , MARGIN = 1)
#       mean_weight = apply(all_weights, function(x) mean(x, na.rm = T) , MARGIN = 1)
#       sd <- apply(all_weights, MARGIN = 1, function(x) sd(x, na.rm = TRUE))
#       summary_df = as.data.frame(cbind(times_included, mean_weight))
#       
#       summary_df[summary_df == "NaN"] = NA
#       
#       # Select the top highest counts and order them
#       top_count <- summary_df[order(-summary_df$times_included), ][1:n, ]
#       top_count$names = rownames(top_count)
#       
#       # Select the top 30 values with the largest absolute values and order them
#       top_weights <- summary_df[order(-abs(summary_df$mean_weight)), ][1:n, ]
#       top_weights$names = rownames(top_weights)
#       
#       # Create a horizontal barplot
#       bar_plot_weights <- ggplot(top_weights, aes(x = reorder(names, mean_weight), y = mean_weight)) +
#         geom_errorbar(aes(ymin = mean_weight - sd, ymax = mean_weight + sd), position = position_dodge(width = 0.8), width = 0.25) +
#         geom_bar(stat = "identity", fill = "pink2") +
#         labs(title = paste0(title, "\nBiggest Weight Data"), x = "Names", y = "Weights in the models") +
#         theme_few() +
#         coord_flip()
#       
#       return(bar_plot_weights)
# }
# 
# plot_svm_weights <- function(svm_weights, title, n){
# 
#     # Extract SVM weights from the results
#     all_weights <- as.data.frame(t(svm_weights[[1]][1,]))
#     for (i in 2:length(svm_weights)) {
#         all_weights <- cbind(all_weights, as.data.frame(t(svm_weights[[i]][1,])))
#     }
#     #all_weights <- all_weights[-1, ]
# 
#     # Remove zero values
#     all_weights[all_weights == 0] <- NA
# 
#     # Calculate mean weight 
#     mean_weight <- apply(all_weights, MARGIN = 1, function(x) mean(x, na.rm = TRUE))
#     sd <- apply(all_weights, MARGIN = 1, function(x) sd(x, na.rm = TRUE))
# 
#     # Select top n weights
#     top_weights <- as.data.frame(mean_weight[order(-abs(mean_weight))][1:n])
#     top_weights$names <- rownames(top_weights)
#     colnames(top_weights) = c("mean_weight", "names")
#     
#     # Create a horizontal barplot of weights
#     barplot <- ggplot(top_weights, aes(x = reorder(names, mean_weight), y = mean_weight)) +
#         geom_bar(stat = "identity", fill = "pink2") +
#         geom_errorbar(aes(ymin = mean_weight - sd, ymax = mean_weight + sd), position = position_dodge(width = 0.8), width = 0.25) +
#         labs(title = paste0(title, "\nTop Weighted Features"), x = "Names", y = "Weights") +
#         theme_few() +
#         coord_flip()
# 
#     return(barplot)
# }
```




```r
# 
# #.TO DO
# # change the function that they export more information about the models 
# #   --> export weight matrices
# #   --> export count matrices
# #   --> export number of features in each model
# #   --> export model details
# #   --> export prediction probabilities with patient labels
# # new weight plot where we put NA as 0
# # add title variable to plot functions
# # visualize the correlation clusters and which features are removed from the feature space
# # check if we get same results if we rerun the models
# 
# 
# library(caret)
# library(pROC)
# library(glmnet)
# library(ggpubr)
# set.seed(9)
# 
# #prepare data for analysis
# df_list = list()
# #k2
# df = assay
# df$status = cluster_assignments_2$`kmeans_k=2`
# df_list$k2 = df
# 
# #k3 - we want to differentiate one cluster from the rest
# df$status = cluster_assignments_2$`kmeans_k=3`
# df_list$k3 = df
# 
# #top protein combination list
# comb_list = list()
# comb_list$k2 = as.data.frame(expand.grid(
#   important_protein_names$k2_alpha[1:5,1], 
#   important_protein_names$k2_beta[1:5,1]))
# # Convert each row to a character vector
# comb_list$k2 <- apply(comb_list$k2, 1, as.character)
# 
# comb_list$k3 = expand.grid(
#   important_protein_names$k3_alpha[1:5,1], 
#   important_protein_names$k3_beta[1:5,1],
#   important_protein_names$k3_theta[1:5,1])
# # Convert each row to a character vector
# comb_list$k3 <- apply(comb_list$k3, 1, as.character)
# 
# models = list()
# auc_plots = list()
# weight_plots = list()
# m = 1
# 
# for(i in 1:length(df_list)){
#   for(k in 1:ncol(comb_list[[i]])){
#       set.seed(9)
#       print(names(df_list)[i])
#       print(k)
#       d = df_list[[i]]
#   
#   #filter proteins
#       prots = comb_list[[i]][,k]
#       d = d[, c(prots, "status")]
#       if(i == 1){
#         levels(d$status) = c("alpha", "beta", "beta", "beta")
#         method_lasso = "glmnet" 
#         method_svm = "svmLinear"
#       }
#       if(i == 2){
#         levels(d$status) =  c("alpha", "beta", "theta", "theta")
#         method_lasso = "multinom"
#         method_svm = "svmPoly"
#       }
#   
#       disease_status = d$status
#       expression_matrix = d[ ,!names(d) == 'status']
#       title = paste0(names(df_list)[i], "_", paste(prots, collapse = "_"))
#       print(title)
# 
#   # Run bootstrap LASSO
#   
#     if(!file.exists(paste0("results/supervisedlearning_model", title, "_lm.rds"))){
#       suppressMessages({
#         suppressWarnings({
#         results <- run_bootstrap_lasso(expression_matrix, disease_status, num_bootstrap = 500, num_folds = 5, method = method_lasso)
#         })})}else{
#           results = readRDS(paste0("results/supervisedlearning_model", title, "_lm.rds"))
#         }
#         
#         #save model in the model results
#         models[[m]] = results[1:3]
#         saveRDS(results, file = paste0("results/supervisedlearning_model_", title, "_lm.rds"))
#          
#         # Plot AUC curve
#         auc_plots[[m]] = plot_auc_curve(results = results, title = paste0(title, "\nLasso Regression"))
# 
#         # Plot lasso weights bar plot
#         weight_plots[[m]] <- plot_lasso_weights(results$lasso_weights, 
#                                     title = paste0(title, "\n lasso regression"), 
#                                     n = 2)
#         names(models)[m] = names(auc_plots)[m] = names(weight_plots)[m] = paste0(title, "_lasso")
#         
#         m = m+1
#         
#     #Run SUPPORT VECTOR MACHINE
#     if(!file.exists(paste0("results/supervisedlearning_model", title, "_svm.rds"))){
#           # Run bootstrap lasso regression
#         suppressMessages({
#         suppressWarnings({
#         results <- run_bootstrap_svm(expression_matrix, disease_status, num_bootstrap = 500, num_folds = 5, method = method_svm)
#         })})}else{
#           results = readRDS(paste0("results/supervisedlearning_model", title, "_svm.rds"))
#         }
#         
#         #save model in the model results
#         models[[m]] = results[1:3]
#         saveRDS(results, file = paste0("results/supervisedlearning_model_", title, "_svm.rds"))
#          
#         # Plot AUC curve
#         auc_plots[[m]] = plot_auc_curve(results = results, title = paste0(title, "\nSupport Vector Machine (linaer kernel)"))
#         
#         # Plot lasso weights bar plot
#         weight_plots[[m]] = plot_svm_weights(results$svm_weights, 
#                                     title = paste0(title, "\nSupport Vector Machine (linear kernel)"), 
#                                     n = 2)
#         weight_plots[[m]]
#         names(models)[m] = names(auc_plots)[m] = names(weight_plots)[m] = paste0(title, "_svm")
#         m = m+1
#     }
#   }
# 
#   
# saveRDS(models, file = "results/all_supervised_learning_models.rds")
# 
# #make weights plot
# ggarrange(plotlist = weight_plots, ncol = 5, nrow = 10)
# ggsave("plots/weight_plots.pdf", width = 20, height = 40, units = "in")
# 
# 
# #make auc plots
# ggarrange(plotlist = auc_plots, ncol = 5, nrow = 10)
# ggsave("plots/auc_plots.pdf", width = 20, height = 20, units = "in")
```


```r
# library(RColorBrewer)
# library(ggpubr)
# 
# 
# #models = readRDS(file = "results/all_supervised_learning_models.rds")
# 
# auc_matrix = as.data.frame(matrix(ncol = 4))
# colnames(auc_matrix) = c("mean", "sd", "model", "proteins")
# 
# 
# for(i in 1:length(models)){
#   name = names(models)[i]
#   name = strsplit(name, split = "_")[[1]]
#   
#   auc_matrix[i,"mean"] = mean(models[[i]]$AUC)
#   auc_matrix[i,"sd"] = sd(models[[i]]$AUC)
#   auc_matrix[i,"model"] = paste0(name[1], "_", name[4])
#   auc_matrix[i,"proteins"] = paste0(name[2], "_", name[3])
#   
# }
# 
# auc_matrix$proteins = as.factor(auc_matrix$proteins)
# auc_matrix$model = as.factor(auc_matrix$model)
# 
# 
# # Create a barplot with error bars
# ggplot(auc_matrix, aes(x = proteins, y = mean, fill = proteins)) +
#   geom_bar(stat = "identity", position = position_dodge(width = 0.8), width = 0.7) +
#   geom_errorbar(aes(ymin = mean - sd, ymax = mean + sd), position = position_dodge(width = 0.8), width = 0.25) +
#   labs(title = "Barplot with Error Bars for Model Performances (AUC)",
#        x = "Models",
#        y = "Mean AUC with Standard Deviation",
#        fill = "Models") +
#   theme_few() +
#   facet_wrap(facets = auc_matrix$model) + 
#   theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))
# 
# ggsave("plots/model_performances.pdf", width = 11*1.5, height = 8/2, units = "in")
```


```r
# #.TO DO
# # change the function that they export more information about the models 
# #   --> export weight matrices
# #   --> export count matrices
# #   --> export number of features in each model
# #   --> export model details
# #   --> export prediction probabilities with patient labels
# # new weight plot where we put NA as 0
# # add title variable to plot functions
# # visualize the correlation clusters and which features are removed from the feature space
# # check if we get same results if we rerun the models
# 
# set.seed(9)
# 
# models = list()
# auc_plots = list()
# weight_plots = list()
# m = 1
# i = 1
# 
# #for(i in 1:length(df_list)){
#   for(k in 1:ncol(comb_list[[i]])){
#       print(names(df_list)[i])
#       print(k)
#       d = df_list[[i]]
#   
#   #filter proteins
#       prots = sample(colnames(d)[colnames(d) != "status"], size = 2, replace = F)
#       d = d[, c(prots, "status")]
#       if(i == 1){
#         levels(d$status) = c("alpha", "beta", "beta", "beta")
#         method_lasso = "glmnet" 
#         method_svm = "svmLinear"
#       }
#       if(i == 2){
#         levels(d$status) =  c("alpha", "beta", "theta", "theta")
#         method_lasso = "multinom"
#         method_svm = "svmPoly"
#       }
#   
#       disease_status = d$status
#       expression_matrix = d[ ,!names(d) == 'status']
#       title = paste0(names(df_list)[i], "_", paste(prots, collapse = "_"), "_overfitting")
#       print(title)
# 
#   # Run bootstrap LASSO
#   
#       suppressMessages({
#         suppressWarnings({
#         results <- run_bootstrap_lasso(expression_matrix, disease_status, num_bootstrap = 500, num_folds = 5, method = method_lasso)
#         })})
#         
#         #save model in the model results
#         models[[m]] = results[1:3]
#         saveRDS(results, file = paste0("results/supervisedlearning_model_", title, "_lm.rds"))
#          
#         # Plot AUC curve
#         auc_plots[[m]] = plot_auc_curve(results = results, title = paste0(title, "\nLasso Regression"))
# 
#         # Plot lasso weights bar plot
#         weight_plots[[m]] <- plot_lasso_weights(results$lasso_weights, 
#                                     title = paste0(title, "\n lasso regression"), 
#                                     n = 2)
#         
#         names(models)[m] = names(auc_plots)[m] = names(weight_plots)[m] = paste0(title, "_lasso")
#         
#         m = m+1
#         
#     #Run SUPPORT VECTOR MACHINE
#           # Run bootstrap lasso regression
#         suppressMessages({
#         suppressWarnings({
#         results <- run_bootstrap_svm(expression_matrix, disease_status, num_bootstrap = 500, num_folds = 5, method = method_svm)
#         })})
#         
#         #save model in the model results
#         models[[m]] = results[1:3]
#         saveRDS(results, file = paste0("results/supervisedlearning_model_", title, "_svm.rds"))
#          
#         # Plot AUC curve
#         auc_plots[[m]] = plot_auc_curve(results = results, title = paste0(title, "\nSupport Vector Machine (linaer kernel)"))
#         
#         # Plot lasso weights bar plot
#         weight_plots[[m]] = plot_svm_weights(results$svm_weights, 
#                                     title = paste0(title, "\nSupport Vector Machine (linear kernel)"), 
#                                     n = 2)
#         names(models)[m] = names(auc_plots)[m] = names(weight_plots)[m] = paste0(title, "_svm")
#         m = m+1
#     }
#   #}
# 
#   
# saveRDS(models, file = "results/all_supervised_learning_models_overfitting.rds")
# 
# #make weights plot
# ggarrange(plotlist = weight_plots, ncol = 5, nrow = 10)
# ggsave("plots/weight_plots_overfitting.pdf", width = 20, height = 40, units = "in")
# 
# 
# #make auc plots
# ggarrange(plotlist = auc_plots, ncol = 5, nrow = 10)
# ggsave("plots/auc_plots_overfitting.pdf", width = 20, height = 20, units = "in")
```


```r
#models = readRDS(file = "results/all_supervised_learning_models.rds")
# 
# auc_matrix = as.data.frame(matrix(ncol = 4))
# colnames(auc_matrix) = c("mean", "sd", "model", "proteins")
# 
# 
# for(i in 1:length(models)){
#   name = names(models)[i]
#   name = strsplit(name, split = "_")[[1]]
#   
#   auc_matrix[i,"mean"] = mean(models[[i]]$AUC)
#   auc_matrix[i,"sd"] = sd(models[[i]]$AUC)
#   auc_matrix[i,"model"] = paste0(name[1], "_", name[5])
#   auc_matrix[i,"proteins"] = paste0(name[2], "_", name[3])
#   
# }
# 
# auc_matrix$proteins = as.factor(auc_matrix$proteins)
# auc_matrix$model = as.factor(auc_matrix$model)
# 
# 
# # Create a barplot with error bars
# ggplot(auc_matrix, aes(x = proteins, y = mean, fill = proteins)) +
#   geom_bar(stat = "identity", position = position_dodge(width = 0.8), width = 0.7) +
#   geom_errorbar(aes(ymin = mean - sd, ymax = mean + sd), position = position_dodge(width = 0.8), width = 0.25) +
#   labs(title = "Barplot with Error Bars for Model Performances (AUC)",
#        x = "Models",
#        y = "Mean AUC with Standard Deviation",
#        fill = "Models") +
#   theme_few() +
#   facet_wrap(facets = auc_matrix$model) + 
#   theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))
# 
# ggsave("plots/model_performances_overfitting.pdf", width = 11*1.5, height = 8/2, units = "in")
```


```r
# #.TO DO
# # change the function that they export more information about the models 
# #   --> export weight matrices
# #   --> export count matrices
# #   --> export number of features in each model
# #   --> export model details
# #   --> export prediction probabilities with patient labels
# # new weight plot where we put NA as 0
# # add title variable to plot functions
# # visualize the correlation clusters and which features are removed from the feature space
# # check if we get same results if we rerun the models
# 
# 
# library(caret)
# library(pROC)
# library(glmnet)
# library(ggpubr)
# set.seed(9)
# 
# #prepare data for analysis
# df_list = list()
# #k2
# df = assay
# df$status = cluster_assignments_2$`kmeans_k=2`
# df_list$k2 = df
# 
# #k3 - we want to differentiate one cluster from the rest
# df$status = cluster_assignments_2$`kmeans_k=3`
# df_list$k3 = df
# 
# #top protein combination list
# comb_list = list()
# comb_list$k2 = as.data.frame(expand.grid(
#   important_protein_names$k2_alpha[1:5,1], 
#   important_protein_names$k2_beta[1:5,1]))
# # Convert each row to a character vector
# comb_list$k2 <- apply(comb_list$k2, 1, as.character)
# 
# comb_list$k3 = expand.grid(
#   important_protein_names$k3_alpha[1:5,1], 
#   important_protein_names$k3_beta[1:5,1],
#   important_protein_names$k3_theta[1:5,1])
# # Convert each row to a character vector
# comb_list$k3 <- apply(comb_list$k3, 1, as.character)
# 
# models = list()
# auc_plots = list()
# weight_plots = list()
# m = 1
# 
# for(i in 1:length(df_list)){
#   for(k in 1:ncol(comb_list[[i]])){
#       set.seed(9)
#       print(names(df_list)[i])
#       print(k)
#       d = df_list[[i]]
#   
#   #filter proteins
#       prots = comb_list[[i]][,k]
#       d = d[, c(prots, "status")]
#       if(i == 1){
#         levels(d$status) = c("alpha", "beta", "beta", "beta")
#         method_lasso = "glmnet" 
#         method_svm = "svmLinear"
#       }
#       if(i == 2){
#         levels(d$status) =  c("alpha", "beta", "theta", "theta")
#         method_lasso = "multinom"
#         method_svm = "svmPoly"
#       }
#       
#       #RANDOMIZE STATUS LABELS:
#       random_labels = sample(c(0,1), size = nrow(d), replace = TRUE)
#       random_labels = as.factor(random_labels)
#       levels(random_labels) = c("alpha", "beta")
#       d$status = random_labels
#       
#       disease_status = d$status
#       expression_matrix = d[ ,!names(d) == 'status']
#       title = paste0(names(df_list)[i], "_", paste(prots, collapse = "_"))
#       print(title)
# 
#   # Run bootstrap LASSO
#       suppressMessages({
#         suppressWarnings({
#         results <- run_bootstrap_lasso(expression_matrix, disease_status, num_bootstrap = 500, num_folds = 5, method = method_lasso)
#         })})
#         
#         #save model in the model results
#         models[[m]] = results[1:3]
#         #saveRDS(results, file = paste0("results/supervisedlearning_model_", title, "_lm.rds"))
#          
#         # Plot AUC curve
#         auc_plots[[m]] = plot_auc_curve(results = results, title = paste0(title, "\nLasso Regression"))
# 
#         # Plot lasso weights bar plot
#         weight_plots[[m]] <- plot_lasso_weights(results$lasso_weights, 
#                                     title = paste0(title, "\n lasso regression"), 
#                                     n = 2)
#         names(models)[m] = names(auc_plots)[m] = names(weight_plots)[m] = paste0(title, "_lasso_overfitting")
#         
#         m = m+1
#         
#     #Run SUPPORT VECTOR MACHINE
#     if(!file.exists(paste0("results/supervisedlearning_model", title, "_svm.rds"))){
#           # Run bootstrap lasso regression
#         suppressMessages({
#         suppressWarnings({
#         results <- run_bootstrap_svm(expression_matrix, disease_status, num_bootstrap = 500, num_folds = 5, method = method_svm)
#         })})}else{
#           results = readRDS(paste0("results/supervisedlearning_model", title, "_svm.rds"))
#         }
#         
#         #save model in the model results
#         models[[m]] = results[1:3]
#         #saveRDS(results, file = paste0("results/supervisedlearning_model_", title, "_svm.rds"))
#          
#         # Plot AUC curve
#         auc_plots[[m]] = plot_auc_curve(results = results, title = paste0(title, "\nSupport Vector Machine (linaer kernel)"))
#         
#         # Plot lasso weights bar plot
#         weight_plots[[m]] = plot_svm_weights(results$svm_weights, 
#                                     title = paste0(title, "\nSupport Vector Machine (linear kernel)"), 
#                                     n = 2)
#         weight_plots[[m]]
#         names(models)[m] = names(auc_plots)[m] = names(weight_plots)[m] = paste0(title, "_svm_overfitting")
#         m = m+1
#     }
#   }
# 
#   
# saveRDS(models, file = "results/all_supervised_learning_models_overfitting2.rds")
# 
# #make weights plot
# ggarrange(plotlist = weight_plots, ncol = 5, nrow = 10)
# ggsave("plots/weight_plots_overfitting2.pdf", width = 20, height = 40, units = "in")
# 
# 
# #make auc plots
# ggarrange(plotlist = auc_plots, ncol = 5, nrow = 10)
# ggsave("plots/auc_plots_overfitting2.pdf", width = 20, height = 20, units = "in")
```


```r
# library(RColorBrewer)
# library(ggpubr)
# 
# 
# #models = readRDS(file = "results/all_supervised_learning_models.rds")
# 
# auc_matrix = as.data.frame(matrix(ncol = 4))
# colnames(auc_matrix) = c("mean", "sd", "model", "proteins")
# 
# 
# for(i in 1:length(models)){
#   name = names(models)[i]
#   name = strsplit(name, split = "_")[[1]]
#   
#   auc_matrix[i,"mean"] = mean(models[[i]]$AUC)
#   auc_matrix[i,"sd"] = sd(models[[i]]$AUC)
#   auc_matrix[i,"model"] = paste0(name[1], "_", name[4])
#   auc_matrix[i,"proteins"] = paste0(name[2], "_", name[3])
#   
# }
# 
# auc_matrix$proteins = as.factor(auc_matrix$proteins)
# auc_matrix$model = as.factor(auc_matrix$model)
# 
# 
# # Create a barplot with error bars
# ggplot(auc_matrix, aes(x = proteins, y = mean, fill = proteins)) +
#   geom_bar(stat = "identity", position = position_dodge(width = 0.8), width = 0.7) +
#   geom_errorbar(aes(ymin = mean - sd, ymax = mean + sd), position = position_dodge(width = 0.8), width = 0.25) +
#   labs(title = "Barplot with Error Bars for Model Performances (AUC)",
#        x = "Models",
#        y = "Mean AUC with Standard Deviation",
#        fill = "Models") +
#   theme_few() +
#   facet_wrap(facets = auc_matrix$model) + 
#   theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))
# 
# ggsave("plots/model_performances_overfitting2.pdf", width = 11*1.5, height = 8/2, units = "in")
```


## R and packages versions


```r
sessionInfo()
```

```
## R version 4.2.3 (2023-03-15)
## Platform: aarch64-apple-darwin20 (64-bit)
## Running under: macOS Ventura 13.3.1
## 
## Matrix products: default
## BLAS:   /Library/Frameworks/R.framework/Versions/4.2-arm64/Resources/lib/libRblas.0.dylib
## LAPACK: /Library/Frameworks/R.framework/Versions/4.2-arm64/Resources/lib/libRlapack.dylib
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] stats4    stats     graphics  grDevices utils     datasets  methods  
## [8] base     
## 
## other attached packages:
##  [1] xlsx_0.6.5                  readxl_1.4.3               
##  [3] gridExtra_2.3               dunn.test_1.3.5            
##  [5] nnet_7.3-19                 fgsea_1.24.0               
##  [7] httr_1.4.7                  lubridate_1.9.2            
##  [9] forcats_1.0.0               purrr_1.0.2                
## [11] tidyr_1.3.0                 tibble_3.2.1               
## [13] tidyverse_2.0.0             ggalluvial_0.12.5          
## [15] writexl_1.4.2               mclust_6.0.0               
## [17] cluster_2.1.4               RColorBrewer_1.1-3         
## [19] ggpubr_0.6.0                readr_2.1.4                
## [21] data.table_1.14.8           SummarizedExperiment_1.28.0
## [23] Biobase_2.58.0              GenomicRanges_1.50.2       
## [25] GenomeInfoDb_1.34.9         IRanges_2.32.0             
## [27] S4Vectors_0.36.2            BiocGenerics_0.44.0        
## [29] MatrixGenerics_1.10.0       naniar_1.0.0               
## [31] DEP_1.20.0                  cowplot_1.1.1              
## [33] umap_0.2.10.0               reshape2_1.4.4             
## [35] ggrepel_0.9.3               dplyr_1.1.2                
## [37] stringr_1.5.0               dichromat_2.0-0.1          
## [39] msigdbr_7.5.1               enrichplot_1.18.4          
## [41] clusterProfiler_4.6.2       wesanderson_0.3.6          
## [43] matrixStats_1.0.0           ggplot2_3.4.3              
## [45] pheatmap_1.0.12             ggthemes_4.2.4             
## 
## loaded via a namespace (and not attached):
##   [1] ragg_1.2.5             visdat_0.6.0           bit64_4.0.5           
##   [4] knitr_1.43             DelayedArray_0.24.0    KEGGREST_1.38.0       
##   [7] RCurl_1.98-1.12        doParallel_1.0.17      generics_0.1.3        
##  [10] preprocessCore_1.60.2  RSQLite_2.3.1          shadowtext_0.1.2      
##  [13] bit_4.0.5              tzdb_0.4.0             httpuv_1.6.11         
##  [16] assertthat_0.2.1       viridis_0.6.4          xfun_0.40             
##  [19] rJava_1.0-6            hms_1.1.3              jquerylib_0.1.4       
##  [22] babelgene_22.9         evaluate_0.21          promises_1.2.1        
##  [25] fansi_1.0.4            igraph_1.5.1           DBI_1.1.3             
##  [28] htmlwidgets_1.6.2      reshape_0.8.9          ellipsis_0.3.2        
##  [31] RSpectra_0.16-1        ggnewscale_0.4.9       backports_1.4.1       
##  [34] vctrs_0.6.3            imputeLCMD_2.1         abind_1.4-5           
##  [37] cachem_1.0.8           withr_2.5.0            ggforce_0.4.1         
##  [40] HDO.db_0.99.1          vroom_1.6.3            treeio_1.22.0         
##  [43] fdrtool_1.2.17         DOSE_3.24.2            ape_5.7-1             
##  [46] lazyeval_0.2.2         crayon_1.5.2           pkgconfig_2.0.3       
##  [49] labeling_0.4.3         tweenr_2.0.2           nlme_3.1-163          
##  [52] ProtGenerics_1.30.0    rlang_1.1.1            lifecycle_1.0.3       
##  [55] sandwich_3.0-2         downloader_0.4         affyio_1.68.0         
##  [58] cellranger_1.1.0       polyclip_1.10-4        Matrix_1.6-1          
##  [61] aplot_0.2.0            carData_3.0-5          zoo_1.8-12            
##  [64] GlobalOptions_0.1.2    png_0.1-8              viridisLite_0.4.2     
##  [67] rjson_0.2.21           mzR_2.32.0             bitops_1.0-7          
##  [70] shinydashboard_0.7.2   gson_0.1.0             Biostrings_2.66.0     
##  [73] blob_1.2.4             shape_1.4.6            qvalue_2.30.0         
##  [76] rstatix_0.7.2          gridGraphics_0.5-1     tmvtnorm_1.5          
##  [79] ggsignif_0.6.4         scales_1.2.1           memoise_2.0.1         
##  [82] magrittr_2.0.3         plyr_1.8.8             zlibbioc_1.44.0       
##  [85] compiler_4.2.3         scatterpie_0.2.1       pcaMethods_1.90.0     
##  [88] clue_0.3-64            cli_3.6.1              affy_1.76.0           
##  [91] XVector_0.38.0         patchwork_1.1.3        MASS_7.3-60           
##  [94] tidyselect_1.2.0       vsn_3.66.0             stringi_1.7.12        
##  [97] textshaping_0.3.6      highr_0.10             yaml_2.3.7            
## [100] GOSemSim_2.24.0        askpass_1.1            norm_1.0-11.1         
## [103] MALDIquant_1.22.1      grid_4.2.3             sass_0.4.7            
## [106] timechange_0.2.0       fastmatch_1.1-4        tools_4.2.3           
## [109] parallel_4.2.3         circlize_0.4.15        rstudioapi_0.15.0     
## [112] MsCoreUtils_1.10.0     foreach_1.5.2          farver_2.1.1          
## [115] mzID_1.36.0            ggraph_2.1.0           digest_0.6.33         
## [118] BiocManager_1.30.22    shiny_1.7.5            Rcpp_1.0.11           
## [121] car_3.1-2              broom_1.0.5            later_1.3.1           
## [124] ncdf4_1.21             MSnbase_2.24.2         AnnotationDbi_1.60.2  
## [127] ComplexHeatmap_2.14.0  colorspace_2.1-0       XML_3.99-0.14         
## [130] reticulate_1.31        splines_4.2.3          yulab.utils_0.0.8     
## [133] tidytree_0.4.5         graphlayouts_1.0.0     xlsxjars_0.6.1        
## [136] gmm_1.8                ggplotify_0.1.2        systemfonts_1.0.4     
## [139] xtable_1.8-4           jsonlite_1.8.7         ggtree_3.6.2          
## [142] tidygraph_1.2.3        ggfun_0.1.2            R6_2.5.1              
## [145] pillar_1.9.0           htmltools_0.5.6        mime_0.12             
## [148] glue_1.6.2             fastmap_1.1.1          DT_0.29               
## [151] BiocParallel_1.32.6    codetools_0.2-19       mvtnorm_1.2-3         
## [154] utf8_1.2.3             lattice_0.21-8         bslib_0.5.1           
## [157] curl_5.0.2             GO.db_3.16.0           openssl_2.1.0         
## [160] limma_3.54.2           rmarkdown_2.24         munsell_0.5.0         
## [163] GetoptLong_1.0.5       GenomeInfoDbData_1.2.9 iterators_1.0.14      
## [166] impute_1.72.3          gtable_0.3.4
```