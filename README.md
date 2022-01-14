# Selection_signature_mapping_scripts_with_replicated_selection
## Table of contents
#### 0 Introduction
#### 1 Pipeline for the analysis of GBS data adapted from [Wickland et al. 2013](https://github.com/dpwickland/GB-eaSy)
#### 2 Filtering
#### 3 <img src="https://render.githubusercontent.com/render/math?math=F_{ST}"> with replicated selection
#### 4 Allele frequency differences
#### 5 Significance threshold based on the FDR for selection
#### 6 Significance thresholds based on drift simulations 
#### 7 Significance thresholds based on the empiric distribution
#### 8 Simulation of Drift
#### 9 Sauron plot
#### 10 Other plotting scripts


## Introduction
This repository contains scripts for selection signature mapping with replicated selection.
Usually in experimental evolution studies a random-mating population is selected in divergent directions for one
trait of interest. The highly divergent subpopulations are then scanned for signatures of selection.
A previous problem in these studies was the determination of significance thresholds. 
One solution to this problem is the usage of **replicated selection**. 
This was introduced by Turner and Miller (2012), but only tested on *Drospohila melanogaster*. <br /> <br />
In this approach two subpopulations are selected in the same direction. 
By that we differentiate between changes caused by selection and changes caused by other factors like drift.
The **replicated selection** is used to calculate a *FDR for selection*. 
Turner and Miller (2012) applied this significance threshold only on the
statistic based on the allele frequency differences. 
Whereas we applied the this new significance threshold to the commonly used 
          <img src="https://render.githubusercontent.com/render/math?math=F_{ST}">
statistic.

## Pipeline for the analysis of GBS data adapted from [Wickland et al. 2013](https://github.com/dpwickland/GB-eaSy)
For the analysis of our raw reads from paired-end genotyping-by-sequencing (GBS) with ApeKI according to [Elshire et al. 2011](https://journals.plos.org/plosone/article/file?id=10.1371/journal.pone.0019379&type=printable), we used the GB-eaSy pipeline from [Wickland et al. 2013](https://github.com/dpwickland/GB-eaSy). <br />
The pipeline consists out of several steps, which comprises:
- [Demultiplexing and trimming of the adapter sequence](https://github.com/dpwickland/GB-eaSy#step-2-demultiplex-raw-reads)
- [Alignement to the reference genome](https://github.com/dpwickland/GB-eaSy#step-3-align-to-reference)
- [Create a list of sorted bam files](https://github.com/dpwickland/GB-eaSy#step-4-create-list-of-sorted-bam-files)
- [Generate pileup and SNP calling](https://github.com/dpwickland/GB-eaSy#step-5-and-6-generate-pileup-and-call-snps)
- Filtering for quality parameters <br /> <br />

This bash-script was run on every sequenced plate separately, since every sequenced plate had it's own adapters and the same set of barcodes was used for all sequenced plates in our case. The *Demultiplexing* and *Alignement to the reference genome* steps were conducted by the `Demultiplexing_and_alignement_to_the_reference_genome.bash` script. <br /> <br />
The *Generate Pileup*,*SNP calling* and *Filtering for quality parameters* were run for all sequencing runs together. The *Filtering for quality parameters* was changed a lot and differed from the GB-eaSy pipline introduced by [Wickland et al. 2013](https://github.com/dpwickland/GB-eaSy). The  *Generate Pileup*,*SNP calling* and *Filtering for quality parameters* steps were conducted by the `Pileup_SNP_calling_Filtering.bash` script.
          
## Filtering
After the VCF file was created with the GB-eaSy pipeline from [Wickland et al. 2013](https://github.com/dpwickland/GB-eaSy) and filtered for some quality parameters we did some additional filtering in R. 
### The filtering steps included:
- Filtering for individual samples with low coverage
- Filtering for read depth per sample
- Filtering for missingness
- Removal of non-diallelic and non-polymorphic markers 

### Loading of required packages and the VCF file:
Most of the function come from the `vcfR` package from [Knaus and Grünwald 2018](https://github.com/knausb/vcfR#:~:text=VcfR%20is%20an%20R%20package%20intended%20to%20allow,rapidly%20read%20from%20and%20write%20to%20VCF%20files.). The <vcfR> package from [Knaus and Grünwald 2018] works with S4 objects, therefore the syntax is quite different from "normal" R jargon. 
Most calculations and filtering steps over the entire set of markers were run by using the `mclapply()` function from the `parallel` package. We observed that the <mclapply()> function was extremly fast and efficient. 
```{r}
library(stringr)
library(data.table)
library(doMC)
library(vcfR)
library(parallel)
cores <- as.integer(Sys.getenv('SLURM_CPUS_PER_TASK'))
cores <- detectCores(all.tests = FALSE, logical = TRUE)
registerDoMC(cores)
setwd("/your/personal/working/directory/")
vcf_S4 <- read.vcfR("DATA.vcf", verbose = FALSE)
cat("VCF contains before anything was done:",nrow(vcf_S4@fix),"markers.","\n")
```
The last command also tells you, how many markers the VCF file comprises.

### Generate a plot to check the quality parameters and if the previous filtering worked:
```{r}
chrom <- create.chromR(name="Supercontig", vcf=vcf_S4, verbose=FALSE)
png(file='101_Quality_plot_distribution_before_filtering_vcf.png',width = 480, height = 480, units = "px", pointsize=12)
par(
  family='sans',
  cex.axis=1,
  bg='white')
chromoqc(chrom, dp.alpha = 66)
dev.off()
```
#### Filtering for individual samples with low coverage
In our case, we choosed a minimum coverage of 10% `(0.1)`> per individual sample.
```{r}
filter_for_ind_coverage <- function(data, threshold){
  gt <- extract.gt(data, element = "GT", as.numeric=TRUE)
  coverage_ind <- function(i){
    length(which(!is.na(gt[,i])))
  }
  coverage_ind_tot <- mapply(coverage_ind,1:ncol(gt))
  coverage_ind_rel <- coverage_ind_tot/NROW(gt)
  NAs_per_ind <- cbind(coverage_ind_tot,coverage_ind_rel)
  rownames(NAs_per_ind) <- colnames(gt)
  NAs_per_ind <- as.data.frame(NAs_per_ind)
  index_ind <- rownames(NAs_per_ind[which(NAs_per_ind$coverage_ind_rel < threshold),])
  look_up_index <- function(i){
    which(colnames(gt)==index_ind[i])
  }
  index_vcf_S4 <- unlist(mclapply(1:length(index_ind),look_up_index))
  data@gt <- data@gt[,-index_vcf_S4]
  return(data)
}
vcf_S4 <- filter_for_ind_coverage(data = vcf_S4, threshold = 0.1)
```
With the following function, the missing observation per individual sample are calculated. Additionally the relative coverage of an individual is calculated. This function needs only be applied, when you want to **save the information about individual coverage** e.g. for plotting. 
```{r}
coverage_per_individual <- function(data){
  gt <- extract.gt(data, element = "GT", as.numeric=TRUE)
  coverage_ind <- function(i){
    length(which(!is.na(gt[,i])))
  }
  coverage_ind_tot <- mapply(coverage_ind,1:ncol(gt))
  coverage_ind_rel <- coverage_ind_tot/NROW(gt)
  NAs_per_ind <- cbind(coverage_ind_tot,coverage_ind_rel)
  rownames(NAs_per_ind) <- colnames(gt)
  NAs_per_ind <- as.data.frame(NAs_per_ind)
  return(NAs_per_ind)
}
NAs_per_ind <- coverage_per_individual(data = vcf_S4)
```
### Filtering for read depth per sample
In our case, we choosed a minimum average read depth of 1 and a maximum average read depth of 10 per marker. We choosed these thresholds based on the distribution of average read depth across all markers. 
```{r}
filter_for_average_read_depth <- function(data, min_av_depth, max_av_depth){
  cat("Filtering for average read depth started.","\n")
  cat("VCF file before filtering contains:",nrow(data@fix),"markers.","\n")
  dp <- extract.info(data, element = "DP", as.numeric=TRUE)
  an <- extract.info(data, element = "AN", as.numeric=TRUE)
  obs <- (an/2)
  average_depth <- dp/obs
  average_depth <- as.data.frame(average_depth)
  rownames(average_depth) <- paste(data@fix[,1],data@fix[,2], sep = "_")
  average_depth$SNP_ID <- paste(data@fix[,1],data@fix[,2], sep = "_")
  average_depth$tot_DP <- dp
  average_depth$observations <- obs
  average_depth$observations <- as.numeric(average_depth$observations)
  average_depth$tot_DP <- as.numeric(average_depth$tot_DP)
  average_depth$average_depth <- as.numeric(average_depth$average_depth)
  index_depth <- which(average_depth$average_depth > min_av_depth & average_depth$average_depth < max_av_depth)
  data@gt <- data@gt[index_depth,]
  data@fix <- data@fix[index_depth,]
  cat(length(index_depth),"markers passed the threshold.","\n")
  cat("VCF file before filtering contains:",nrow(data@fix),"markers.","\n")
  return(data)
}
vcf_S4 <- filter_for_average_read_depth(data = vcf_S4, min_av_depth = 1, 
                                        max_av_depth = 10)   
``` 
**Distribution of average read depth across all markers**
![GB1002 1_Average_read_depth](https://user-images.githubusercontent.com/63467079/149306207-62129755-9db6-473d-a1a9-dc2795ec84c6.png)

In these steps the average read depth per sample is calculated. Based on this data, the previous plot was created:
```{r}
calulate_average_read_depth <- function(data){
  dp <- extract.info(data, element = "DP", as.numeric=TRUE)
  an <- extract.info(data, element = "AN", as.numeric=TRUE)
  obs <- (an/2)
  average_depth <- dp/obs
  average_depth <- as.data.frame(average_depth)
  rownames(average_depth) <- paste(data@fix[,1],data@fix[,2], sep = "_")
  average_depth$SNP_ID <- paste(data@fix[,1],data@fix[,2], sep = "_")
  average_depth$tot_DP <- dp
  average_depth$observations <- obs
  average_depth$observations <- as.numeric(average_depth$observations)
  average_depth$tot_DP <- as.numeric(average_depth$tot_DP)
  average_depth$average_depth <- as.numeric(average_depth$average_depth)
  return(average_depth)
}
average_depth <- calulate_average_read_depth(data = vcf_S4)
```
### Filtering for missingness
When we filter for missingness, we filter in every subpopulation for at least 40 observations. So that only markers pass the threshold, which have 40 observations in every subpopulation.          
```{r}
filter_missingness_per_pop <- function(data, 
                                       population_1,
                                       population_2,
                                       population_3,
                                       population_4,
                                       max_missing_obs){
  cat("Filtering for missing observations per markers started.","\n")
  cat("VCF file contains before filtering for missingness:",nrow(vcf_S4@fix),"markers.","\n")
  gt_pop <- extract.gt(vcf_S4, element = "GT", as.numeric=TRUE)
  dp <- extract.info(data, element = "DP", as.numeric=TRUE)
  pop_1 <- gt_pop[,which(str_detect(colnames(gt_pop),population_1))]
  pop_2 <- gt_pop[,which(str_detect(colnames(gt_pop),population_2))]
  pop_3 <- gt_pop[,which(str_detect(colnames(gt_pop),population_3))]
  pop_4 <- gt_pop[,which(str_detect(colnames(gt_pop),population_4))]
  get_missing_obs_per_pop <- function(pop){
    get_missing_obs <- function(i){
      length(which(is.na(pop[i,])))
    }
    mis_obs_pop <- unlist(mclapply(1:nrow(pop), get_missing_obs))
    return(mis_obs_pop)
  }
  mis_obs_pop1 <- get_missing_obs_per_pop(pop = pop_1)
  mis_obs_pop2 <- get_missing_obs_per_pop(pop = pop_2)
  mis_obs_pop3 <- get_missing_obs_per_pop(pop = pop_3)
  mis_obs_pop4 <- get_missing_obs_per_pop(pop = pop_4)
  NAs_per_marker_per_pop <- cbind(mis_obs_pop1,mis_obs_pop2,mis_obs_pop3,mis_obs_pop4)
  NAs_per_marker_per_pop <- as.data.frame(NAs_per_marker_per_pop)
  NAs_per_marker_per_pop <- cbind(rownames(gt_pop),NAs_per_marker_per_pop)
  colnames(NAs_per_marker_per_pop) <- c("SNP_ID","mis_obs_pop1","mis_obs_pop2","mis_obs_pop3","mis_obs_pop4")
  NAs_per_marker_per_pop <- as.data.frame(NAs_per_marker_per_pop)
  get_range_of_missingness <- function(i){
    abs(range(NAs_per_marker_per_pop[i,2:5])[1]-range(NAs_per_marker_per_pop[i,2:5])[2])
  }
  markers_diff <- unlist(mclapply(1:nrow(NAs_per_marker_per_pop),get_range_of_missingness))
  NAs_per_marker_per_pop$range_of_missingness <- markers_diff
  index_NA_markers_dp <- subset(rownames(dp),96-as.numeric(NAs_per_marker_per_pop[,2]) > max_missing_obs &
                                  96-as.numeric(NAs_per_marker_per_pop[,3]) > max_missing_obs &
                                  96-as.numeric(NAs_per_marker_per_pop[,4]) > max_missing_obs &
                                  96-as.numeric(NAs_per_marker_per_pop[,5]) > max_missing_obs)
  index_NA_markers_gt <- subset(rownames(gt_pop),96-as.numeric(NAs_per_marker_per_pop[,2]) > max_missing_obs &
                                  96-as.numeric(NAs_per_marker_per_pop[,3]) > max_missing_obs &
                                  96-as.numeric(NAs_per_marker_per_pop[,4]) > max_missing_obs &
                                  96-as.numeric(NAs_per_marker_per_pop[,5]) > max_missing_obs)
  cat(length(index_NA_markers_gt),"markers passed your threshold for missingness.","\n")
  data@gt <- data@gt[match(index_NA_markers_gt, rownames(gt_pop)),]
  data@fix <- data@fix[match(index_NA_markers_dp, rownames(dp)),]
  cat("VCF file contains after filtering for missingness:",nrow(data@gt),"markers.","\n")
  return(data)
}
vcf_S4_new <- filter_missingness_per_pop(data = vcf_S4, 
                           population_1 = "Shoepag_1",
                           population_2 = "Shoepag_2",
                           population_3 = "Shoepag_3",
                           population_4 = "Shoepag_4",
                           max_missing_obs = 40)
```
In these steps the average read depth per sample is calculated. This function needs only be applied, when you want to **save the information about individual coverage** e.g. for plotting. 
```{r}
calculate_missingness_per_pop <- function(data, 
                                          population_1,
                                          population_2,
                                          population_3,
                                          population_4){
  gt_pop <- extract.gt(vcf_S4, element = "GT", as.numeric=TRUE)
  dp <- extract.info(data, element = "DP", as.numeric=TRUE)
  pop_1 <- gt_pop[,which(str_detect(colnames(gt_pop),population_1))]
  pop_2 <- gt_pop[,which(str_detect(colnames(gt_pop),population_2))]
  pop_3 <- gt_pop[,which(str_detect(colnames(gt_pop),population_3))]
  pop_4 <- gt_pop[,which(str_detect(colnames(gt_pop),population_4))]
  get_missing_obs_per_pop <- function(pop){
    get_missing_obs <- function(i){
      length(which(is.na(pop[i,])))
    }
    mis_obs_pop <- unlist(mclapply(1:nrow(pop), get_missing_obs))
    return(mis_obs_pop)
  }
  mis_obs_pop1 <- get_missing_obs_per_pop(pop = pop_1)
  mis_obs_pop2 <- get_missing_obs_per_pop(pop = pop_2)
  mis_obs_pop3 <- get_missing_obs_per_pop(pop = pop_3)
  mis_obs_pop4 <- get_missing_obs_per_pop(pop = pop_4)
  NAs_per_marker_per_pop <- cbind(mis_obs_pop1,mis_obs_pop2,mis_obs_pop3,mis_obs_pop4)
  NAs_per_marker_per_pop <- as.data.frame(NAs_per_marker_per_pop)
  NAs_per_marker_per_pop <- cbind(rownames(gt_pop),NAs_per_marker_per_pop)
  colnames(NAs_per_marker_per_pop) <- c("SNP_ID","mis_obs_pop1","mis_obs_pop2","mis_obs_pop3","mis_obs_pop4")
  NAs_per_marker_per_pop <- as.data.frame(NAs_per_marker_per_pop)
  get_range_of_missingness <- function(i){
    abs(range(NAs_per_marker_per_pop[i,2:5])[1]-range(NAs_per_marker_per_pop[i,2:5])[2])
  }
  markers_diff <- unlist(mclapply(1:nrow(NAs_per_marker_per_pop),get_range_of_missingness))
  NAs_per_marker_per_pop$range_of_missingness <- markers_diff
  return(NAs_per_marker_per_pop)
}
NAs_per_marker_per_pop <- calculate_missingness_per_pop(data = vcf_S4,
                              population_1 = "Shoepag_1",
                              population_2 = "Shoepag_2",
                              population_3 = "Shoepag_3",
                              population_4 = "Shoepag_4")
```
With this command we also calculate the range of missingness between the populations. The range of missingness can be evaluated to look for markers which are more abundant in one population than in the others, which could skew the results.      
![GB1003_plot_missingness_02](https://user-images.githubusercontent.com/63467079/149474916-930cb3f0-bc73-4846-b334-f803ee950eb0.png)

### Removal of non-diallelic and non-polymorphic markers
```{r}
filter_for_non_diallelic_non_polymorphic <- function(data){
  cat("Filtering for diallelic markers started.","\n")
  dp <- extract.gt(data, element = "DP", as.numeric=TRUE)
  gt <- extract.gt(data, element = "GT", as.numeric=TRUE)
  index_not_diallelic <- which(str_count(data@fix[,5])!=1)
  cat(length(index_not_diallelic),"markers were not diallelic.","\n")
  data@gt <- data@gt[-index_not_diallelic,]
  data@fix <- data@fix[-index_not_diallelic,]
  index_polymorphic_markers <- which(is.polymorphic(data, na.omit = TRUE))
  data@gt <- data@gt[index_polymorphic_markers,]
  data@fix <- data@fix[index_polymorphic_markers,]
  cat(length(index_polymorphic_markers),"markers were polymorphic.","\n")
  cat("The VCF file still contains:",nrow(vcf_S4@gt),"markers.","\n")
  return(data)
}
vcf_S4_new <- filter_for_non_diallelic_non_polymorphic(data = vcf_S4)
```

## <img src="https://render.githubusercontent.com/render/math?math=F_{ST}"> with replicated selection


## Allele frequency differences


## Significance threshold based on the FDR for selection


## Significance thresholds based on drift simulations 


## Significance thresholds based on the empiric distribution


## Simulation of Drift


## Sauron plot

![GB1006_Sauron_plots_combined_values_2021_12_17](https://user-images.githubusercontent.com/63467079/149146525-ce94e222-dff8-4ad4-8dcb-ad14f7530032.png)

## Other plotting scripts

