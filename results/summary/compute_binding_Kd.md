Compute per-barcode binding functional score
================
Tyler Starr
5/1/2020

This notebook reads in per-barcode counts from `count_variants.ipynb`
for ACE2-binding Tite-seq experiments, computes functional scores for
RBD ACE2-binding affiniity, and does some basic QC on variant binding
functional scores.

``` r
require("knitr")
knitr::opts_chunk$set(echo = T)
knitr::opts_chunk$set(dev.args = list(png = list(type = "cairo")))

#list of packages to install/load
packages = c("yaml","data.table","tidyverse","Hmisc","gridExtra")
#install any packages not already installed
installed_packages <- packages %in% rownames(installed.packages())
if(any(installed_packages == F)){
  install.packages(packages[!installed_packages])
}
#load packages
invisible(lapply(packages, library, character.only=T))

#read in config file
config <- read_yaml("config.yaml")

#make output directory
if(!file.exists(config$Titeseq_Kds_dir)){
  dir.create(file.path(config$Titeseq_Kds_dir))
}
```

Session info for reproducing environment:

``` r
sessionInfo()
```

    ## R version 3.6.1 (2019-07-05)
    ## Platform: x86_64-pc-linux-gnu (64-bit)
    ## Running under: Ubuntu 14.04.6 LTS
    ## 
    ## Matrix products: default
    ## BLAS/LAPACK: /app/easybuild/software/OpenBLAS/0.2.18-GCC-5.4.0-2.26-LAPACK-3.6.1/lib/libopenblas_prescottp-r0.2.18.so
    ## 
    ## locale:
    ##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
    ##  [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
    ##  [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
    ##  [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
    ##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
    ## [11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ##  [1] gridExtra_2.3     Hmisc_4.2-0       Formula_1.2-3    
    ##  [4] survival_2.44-1.1 lattice_0.20-38   forcats_0.4.0    
    ##  [7] stringr_1.4.0     dplyr_0.8.3       purrr_0.3.2      
    ## [10] readr_1.3.1       tidyr_0.8.3       tibble_2.1.3     
    ## [13] ggplot2_3.2.0     tidyverse_1.2.1   data.table_1.12.2
    ## [16] yaml_2.2.0        knitr_1.23       
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] tidyselect_0.2.5    xfun_0.7            splines_3.6.1      
    ##  [4] haven_2.1.1         colorspace_1.4-1    generics_0.0.2     
    ##  [7] htmltools_0.3.6     base64enc_0.1-3     rlang_0.4.0        
    ## [10] pillar_1.4.2        foreign_0.8-71      glue_1.3.1         
    ## [13] withr_2.1.2         RColorBrewer_1.1-2  modelr_0.1.4       
    ## [16] readxl_1.3.1        munsell_0.5.0       gtable_0.3.0       
    ## [19] cellranger_1.1.0    rvest_0.3.4         htmlwidgets_1.3    
    ## [22] evaluate_0.14       latticeExtra_0.6-28 htmlTable_1.13.1   
    ## [25] broom_0.5.2         Rcpp_1.0.1          acepack_1.4.1      
    ## [28] checkmate_1.9.4     scales_1.0.0        backports_1.1.4    
    ## [31] jsonlite_1.6        hms_0.4.2           digest_0.6.20      
    ## [34] stringi_1.4.3       grid_3.6.1          cli_1.1.0          
    ## [37] tools_3.6.1         magrittr_1.5        lazyeval_0.2.2     
    ## [40] cluster_2.1.0       crayon_1.3.4        pkgconfig_2.0.2    
    ## [43] Matrix_1.2-17       xml2_1.2.0          lubridate_1.7.4    
    ## [46] assertthat_0.2.1    rmarkdown_1.13      httr_1.4.0         
    ## [49] rstudioapi_0.10     rpart_4.1-15        R6_2.4.0           
    ## [52] nnet_7.3-12         nlme_3.1-140        compiler_3.6.1

## Setup

First, we will read in metadata on our sort samples and the table giving
number of reads of each barcode in each of the sort bins, convert from
Illumina read counts to estimates of the number of cells that were
sorted into a bin, and add some other useful information to our counts
data.

Note: we are loading our per-barcode counts in as a `data.table` object,
which improves speed by vectorizing many typical dataframe operations
and automatically parallelizing when possible, and has more streamlined
slicing and dicing operators without requiring all of the same `$` and
`""` syntax typically needed in a `data.frame`. Many ways of slicing a
typical `data.frame` will not work with the `data.table`, so check out
guides on `data.table` if you’re going to dig in yourself\!

``` r
#read dataframe with list of barcode runs
barcode_runs <- read.csv(file=config$barcode_runs,stringsAsFactors=F); barcode_runs <- subset(barcode_runs, select=-c(R1))

#eliminate rows from barcode_runs that are not from a binding Tite-seq experiment
barcode_runs <- barcode_runs[barcode_runs$sample_type == "TiteSeq",]

#read file giving count of each barcode in each sort partition
counts <- data.table(read.csv(file=config$variant_counts_file,stringsAsFactors=F)); counts <- counts[order(counts$library,counts$target,counts$barcode),]

#eliminate rows from counts that are not part of an expression sort-seq bin
counts <- subset(counts, sample %in% barcode_runs[barcode_runs$sample_type=="TiteSeq","sample"])

#for each bin, normalize the read counts to the observed ratio of cell recovery among bins
for(i in 1:nrow(barcode_runs)){
  lib <- as.character(barcode_runs$library[i])
  bin <- as.character(barcode_runs$sample[i])
  if(sum(counts[library==lib & sample==bin,"count"]) < barcode_runs$number_cells[i]){ #if there are fewer reads from a sortseq bin than cells sorted
    counts[library==lib & sample==bin, count.norm := as.numeric(count)] #don't normalize cell counts, make count.norm the same as count
    print(paste("reads < cells for",lib,bin,", un-normalized")) #print to console to inform of undersampled bins
  }else{
    ratio <- sum(counts[library==lib & sample==bin,"count"])/barcode_runs$number_cells[i]
    counts[library==lib & sample==bin, count.norm := as.numeric(count/ratio)] #normalize read counts by the average read:cell ratio, report in new "count.norm" column
  }
}

#annotate each barcode as to whether it's a homolog variant, SARS-CoV-2 wildtype, synonymous muts only, stop, nonsynonymous, >1 nonsynonymous mutations
counts[target != "SARS-CoV-2", variant_class := target]
counts[target == "SARS-CoV-2" & n_codon_substitutions==0, variant_class := "wildtype"]
counts[target == "SARS-CoV-2" & n_codon_substitutions > 0 & n_aa_substitutions==0, variant_class := "synonymous"]
counts[target == "SARS-CoV-2" & n_aa_substitutions>0 & grepl("*",aa_substitutions,fixed=T), variant_class := "stop"]
counts[target == "SARS-CoV-2" & n_aa_substitutions == 1 & !grepl("*",aa_substitutions,fixed=T), variant_class := "1 nonsynonymous"]
counts[target == "SARS-CoV-2" & n_aa_substitutions > 1 & !grepl("*",aa_substitutions,fixed=T), variant_class := ">1 nonsynonymous"]

#cast the counts data frame into wide format for lib1 and lib2 replicates
counts_lib1 <- dcast(counts[library=="lib1",], barcode + variant_call_support + target + variant_class + aa_substitutions + n_aa_substitutions + codon_substitutions + n_codon_substitutions ~ sample, value.var="count.norm")
counts_lib2 <- dcast(counts[library=="lib2",], barcode + variant_call_support + target + variant_class + aa_substitutions + n_aa_substitutions + codon_substitutions + n_codon_substitutions ~ sample, value.var="count.norm")

#make tables giving names of Titeseq samples and the corresponding ACE2 incubation concentrations
samples_lib1 <- data.frame(sample=unique(paste(barcode_runs$sample_type,formatC(barcode_runs$concentration, width=2,flag="0"),sep="_")),conc=c(10^-6, 10^-6.5, 10^-7, 10^-7.5, 10^-8, 10^-8.5, 10^-9, 10^-9.5, 10^-10, 10^-10.5, 10^-11, 10^-11.5, 10^-12, 10^-12.5, 10^-13,0))

samples_lib2 <- data.frame(sample=unique(paste(barcode_runs$sample_type,formatC(barcode_runs$concentration, width=2,flag="0"),sep="_")),conc=c(10^-6, 10^-6.5, 10^-7, 10^-7.5, 10^-8, 10^-8.5, 10^-9, 10^-9.5, 10^-10, 10^-10.5, 10^-11, 10^-11.5, 10^-12, 10^-12.5, 10^-13,0))
```

## Calculating mean bin for each barcode at each sample concentration

Next, for each barcode at each of the 16 ACE2 concentrations, calculate
the “mean bin” response variable. This is calculated as a simple mean,
where the value of each bin is the integer value of the bin
(bin1=unbound, bin4=highly bound) – because of how bins are defined, the
mean fluorescence of cells in each bin are equally spaced on a
log-normal scale, so mean bin correlates with simple mean fluorescence.

``` r
#function that returns mean bin and sum of counts for four bins cell counts
calc.meanbin <- function(vec){return( list((vec[1]*1+vec[2]*2+vec[3]*3+vec[4]*4)/(vec[1]+vec[2]+vec[3]+vec[4]),
                                           (vec[1]+vec[2]+vec[3]+vec[4])) )}

#iterate through Titeseq samples, compute mean_bin and total_count for each barcode variant
for(i in 1:nrow(samples_lib1)){ #iterate through titeseq sample (concentration)
  meanbin_out <- paste(samples_lib1[i,"sample"],"_meanbin",sep="") #define the header name for the meanbin output for the given concentration sample
  totalcount_out <- paste(samples_lib1[i,"sample"],"_totalcount",sep="") #define the header name for the total cell count output for the given concentration sample
  bin1_in <- paste(samples_lib1[i,"sample"],"_bin1",sep="") #define the header names for the input cell counts for bins1-4 of the given concnetration sample
  bin2_in <- paste(samples_lib1[i,"sample"],"_bin2",sep="")
  bin3_in <- paste(samples_lib1[i,"sample"],"_bin3",sep="")
  bin4_in <- paste(samples_lib1[i,"sample"],"_bin4",sep="")
  counts_lib1[,c(meanbin_out,totalcount_out) := calc.meanbin(c(get(bin1_in),get(bin2_in),get(bin3_in),get(bin4_in))),by=barcode]
}

for(i in 1:nrow(samples_lib2)){ #iterate through titeseq sample (concentration)
  meanbin_out <- paste(samples_lib2[i,"sample"],"_meanbin",sep="") #define the header name for the meanbin output for the given concentration sample
  totalcount_out <- paste(samples_lib2[i,"sample"],"_totalcount",sep="") #define the header name for the total cell count output for the given concentration sample
  bin1_in <- paste(samples_lib2[i,"sample"],"_bin1",sep="") #define the header names for the input cell counts for bins1-4 of the given concnetration sample
  bin2_in <- paste(samples_lib2[i,"sample"],"_bin2",sep="")
  bin3_in <- paste(samples_lib2[i,"sample"],"_bin3",sep="")
  bin4_in <- paste(samples_lib2[i,"sample"],"_bin4",sep="")
  counts_lib2[,c(meanbin_out,totalcount_out) := calc.meanbin(c(get(bin1_in),get(bin2_in),get(bin3_in),get(bin4_in))),by=barcode]
}
```

## Estimating variance on mean bin measurements for weighted least squares regression

We want to fit titration curves to meanbin \~ concentration using least
squares regression – but importantly, within a single barcode there is
variance in how accurately each mean bin value is determined. We
therefore want to use an estimate of variance in each meanbin
measurement to perform *weighted* least squares regression, with weights
proportional to the inverse of variance of a meanbin estimate. The major
contributor to this variance in meanbin measurements is likely the cell
count with which a mean bin value was estimated for a particular barcode
at a concentration – for instance, a mean bin value calculated from 100
cell observations of a barcode across the four bins is going to be a
more precise estimate than a mean bin value calculated from 10 cell
observations.

We will use an empirical approach to assign variance estimates as a
function of cell count at each concentration. We have replicate
measurements of WT/synonymous variants in the library across a range of
`totalcount` values. We bin WT/synonymous barcodes within groupings of
similar `totalcount` values, giving us a sampling distribution of mean
bin across different levels of coverage, from which we can compute
variances. We will pick a TiteSeq sample concentration where the
WT/synonymous barcodes average mean bin is closest to 2.5 – this is
where the wildtype titration curve is near its *K*<sub>D,app</sub> /
inflection point, which should give the most conservative (highest)
estimate of empirical variabilities as a function of cell count.

Then, for each library meanbin measurement, we will assign it a variance
based on the empirical variance estimate for the given `totalcount` of
cells sampled for that barcode at that concentration.

``` r
#want to fit titration curve as weighted least squares -- need an estimate of variance. Will use repeated observations of wildtype across different count depths at the s10 concentration (~Kd) to generate these estimated variances?
wt_lib1 <- counts_lib1[variant_class %in% c("synonymous","wildtype") & !is.na(TiteSeq_10_meanbin) ,]
#mean(wt_lib1[TiteSeq_10_totalcount>10,TiteSeq_10_meanbin],na.rm=T) #for chocie of sample 10
#make bins of wildtype observations
n.breaks_lib1 <- 30
wt.bins_lib1 <- data.frame(bin=1:n.breaks_lib1)
breaks_lib1 <- cut2(wt_lib1$TiteSeq_10_totalcount,m=250,g=n.breaks_lib1,onlycuts=T)
#pool wt observations that fall within the bin breaks, compute sd
for(i in 1:nrow(wt.bins_lib1)){
  wt.bins_lib1$range.cells[i] <- list(c(breaks_lib1[i],breaks_lib1[i+1]))
  data <- wt_lib1[TiteSeq_10_totalcount >= wt.bins_lib1$range.cells[i][[1]][[1]] & TiteSeq_10_totalcount < wt.bins_lib1$range.cells[i][[1]][[2]],]
  wt.bins_lib1$mean.meanbin[i] <- mean(data$TiteSeq_10_meanbin,na.rm=T)
  wt.bins_lib1$sd.meanbin[i] <- sd(data$TiteSeq_10_meanbin,na.rm=T)
  wt.bins_lib1$median.cells[i] <- median(data$TiteSeq_10_totalcount,na.rm=T)
}
#look at relationship between variance and cell counts; fit curve to estimate variance from cell count
par(mfrow=c(1,2))
y_lib1 <- (wt.bins_lib1$sd.meanbin)^2
x_lib1 <- wt.bins_lib1$median.cells
plot(x_lib1,y_lib1,xlab="number cells",ylab="variance in meanbin measurement",main="lib1",pch=19,col="#92278F")
plot(log(x_lib1),log(y_lib1),xlab="log(number cells)",ylab="log(variance in meanbin measurement)",main="lib1",pch=19,col="#92278F")
wt.fit_lib1 <- lm(log(y_lib1) ~ log(x_lib1));summary(wt.fit_lib1);abline(wt.fit_lib1)
```

    ## 
    ## Call:
    ## lm(formula = log(y_lib1) ~ log(x_lib1))
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.34579 -0.21136 -0.04264  0.15584  0.69825 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) -0.34107    0.19992  -1.706   0.0991 .  
    ## log(x_lib1) -0.87115    0.05194 -16.772 3.88e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.2973 on 28 degrees of freedom
    ## Multiple R-squared:  0.9095, Adjusted R-squared:  0.9062 
    ## F-statistic: 281.3 on 1 and 28 DF,  p-value: 3.877e-16

<img src="compute_binding_Kd_files/figure-gfm/estimate_variance_lib1-1.png" style="display: block; margin: auto;" />

``` r
#repeat for lib2
par(mfrow=c(1,2))
wt_lib2 <- counts_lib2[variant_class %in% c("synonymous","wildtype") & !is.na(TiteSeq_11_meanbin) ,]
mean(wt_lib2[TiteSeq_11_totalcount>10,TiteSeq_11_meanbin],na.rm=T)
```

    ## [1] 2.193846

``` r
#make bins of wildtype observations
n.breaks_lib2 <- 30
wt.bins_lib2 <- data.frame(bin=1:n.breaks_lib2)
breaks_lib2 <- cut2(wt_lib2$TiteSeq_10_totalcount,m=250,g=n.breaks_lib2,onlycuts=T)
#pool wt observations that fall within the bin breaks, compute sd
for(i in 1:nrow(wt.bins_lib2)){
  wt.bins_lib2$range.cells[i] <- list(c(breaks_lib2[i],breaks_lib2[i+1]))
  data <- wt_lib2[TiteSeq_10_totalcount >= wt.bins_lib2$range.cells[i][[1]][[1]] & TiteSeq_10_totalcount < wt.bins_lib2$range.cells[i][[1]][[2]],]
  wt.bins_lib2$mean.meanbin[i] <- mean(data$TiteSeq_10_meanbin,na.rm=T)
  wt.bins_lib2$sd.meanbin[i] <- sd(data$TiteSeq_10_meanbin,na.rm=T)
  wt.bins_lib2$median.cells[i] <- median(data$TiteSeq_10_totalcount,na.rm=T)
}
#look at relationship between variance and cell counts; fit curve to estimate variance from cell count
y_lib2 <- (wt.bins_lib2$sd.meanbin)^2
x_lib2 <- wt.bins_lib2$median.cells
plot(x_lib2,y_lib2,xlab="number cells",ylab="variance in meanbin measurement",main="lib2",pch=19,col="#92278F")
plot(log(x_lib2),log(y_lib2),xlab="log(number cells)",ylab="log(variance in meanbin measurement)",main="lib2",pch=19,col="#92278F")
wt.fit_lib2 <- lm(log(y_lib2) ~ log(x_lib2));summary(wt.fit_lib2);abline(wt.fit_lib2)
```

    ## 
    ## Call:
    ## lm(formula = log(y_lib2) ~ log(x_lib2))
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.62935 -0.36628 -0.06639  0.24569  0.99458 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) -0.90190    0.26535  -3.399  0.00205 ** 
    ## log(x_lib2) -0.86238    0.06624 -13.019 2.13e-13 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.4223 on 28 degrees of freedom
    ## Multiple R-squared:  0.8582, Adjusted R-squared:  0.8532 
    ## F-statistic: 169.5 on 1 and 28 DF,  p-value: 2.129e-13

<img src="compute_binding_Kd_files/figure-gfm/estimate_variance_lib2-1.png" style="display: block; margin: auto;" />

``` r
#give function that returns variance from cell count (but doesn't give any variance lower than the lowest observed empirical variance)
est.var.lib1 <- function(count,fit=wt.fit_lib1){
  var <- exp(as.numeric(fit$coefficients[1]) + as.numeric(fit$coefficients[2]) * log(count))
  if(var > min(wt.bins_lib1$sd.meanbin)^2){return(var)}else{return(min(wt.bins_lib1$sd.meanbin)^2)}
}
est.var.lib2 <- function(count,fit=wt.fit_lib2){
  var <- exp(as.numeric(fit$coefficients[1]) + as.numeric(fit$coefficients[2]) * log(count))
  if(var > min(wt.bins_lib2$sd.meanbin)^2){return(var)}else{return(min(wt.bins_lib2$sd.meanbin)^2)}
} #easier code-wise later on to just duplicate the function and ascribe the fit that is used as a default parameter, even though it could be kept more flexible with one function
```

## Fit titration curves

We will use nonlinear least squares regression to fit curves to each
barcode’s titration series. We will do weighted nls, using the empirical
variance estimates from above to weight each observation. We will also
include a minimum cell count that is required for a meanbin estimate to
be used in the titration fit, and a minimum number of concentrations
with determined meanbin that is required for a titration to be reported.

``` r
#For QC and filtering, output columns giving the average number of cells that were sampled for a barcode across the 16 sample concnetrations, and a value for the number of meanbin estimates that were removed for being below the # of cells cutoff
cutoff <- 2
counts_lib1[,avgcount := mean(c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,TiteSeq_04_totalcount,
                                TiteSeq_05_totalcount,TiteSeq_06_totalcount,TiteSeq_07_totalcount,TiteSeq_08_totalcount,
                                TiteSeq_09_totalcount,TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,TiteSeq_16_totalcount)),by=barcode]
counts_lib2[,avgcount := mean(c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,TiteSeq_04_totalcount,
                                TiteSeq_05_totalcount,TiteSeq_06_totalcount,TiteSeq_07_totalcount,TiteSeq_08_totalcount,
                                TiteSeq_09_totalcount,TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,TiteSeq_16_totalcount)),by=barcode]
counts_lib1[,min_cell_filtered := sum(c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,TiteSeq_04_totalcount,
                                        TiteSeq_05_totalcount,TiteSeq_06_totalcount,TiteSeq_07_totalcount,TiteSeq_08_totalcount,
                                        TiteSeq_09_totalcount,TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                        TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,
                                        TiteSeq_16_totalcount)<cutoff),by=barcode]
counts_lib2[,min_cell_filtered := sum(c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,TiteSeq_04_totalcount,
                                        TiteSeq_05_totalcount,TiteSeq_06_totalcount,TiteSeq_07_totalcount,TiteSeq_08_totalcount,
                                        TiteSeq_09_totalcount,TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                        TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,
                                        TiteSeq_16_totalcount)<cutoff),by=barcode]

#function that fits a nls regression to the titration series, including weights from the counts and an option to filter below certain thresholds for average cells across all samples, and number of samples below a cutoff of cells
fit.titration.lib1 <- function(y.vals,x.vals,count.vals,min.cfu=cutoff,min.means=0.6,min.average=5,Kd.start=2e-11,a.start=3,a.lower=1.5,a.upper=3,b.start=1,b.lower=1,b.upper=1.5){
  indices <- count.vals>min.cfu
  y <- y.vals[indices]
  x <- x.vals[indices]
  w <- 1/sapply(count.vals[indices], est.var.lib1)
  if((length(y) < min.means*length(y.vals)) | (mean(count.vals,na.rm=T) < min.average)){ #return NAs if < min.means fraction of concentrations have above min.cfu counts or if the average count across all concentrations is below min.average
    return(list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA)))
  }else{
    fit <- nls(y ~ a*(x/(x+Kd))+b,
               start=list(a=a.start,b=b.start,Kd=Kd.start),
               lower=list(a=a.lower,b=b.lower,Kd=min(x.vals[x.vals>0])/100), #constrain Kd to be no lower than 100x the lowest concentration value
               upper=list(a=a.upper,b=b.upper,Kd=max(x.vals[x.vals>0])*100), #constrain Kd to be no higher than 100x the highest concentration value
               weights=w,algorithm="port")  
    return(list(as.numeric(summary(fit)$coefficients["Kd","Estimate"]),
                as.numeric(summary(fit)$coefficients["Kd","Std. Error"]),
                as.numeric(summary(fit)$coefficients["a","Estimate"]),
                as.numeric(summary(fit)$coefficients["b","Estimate"]),
                as.numeric(summary(fit)$sigma),
                list(fit)))
  }
}

#fit titration to lib1 Titeseq data for each barcode
counts_lib1[,c("Kd","Kd_SE","response","baseline","RSE","fit") := tryCatch(fit.titration.lib1(y.vals=c(TiteSeq_01_meanbin,TiteSeq_02_meanbin,TiteSeq_03_meanbin,TiteSeq_04_meanbin,
                                                                                                       TiteSeq_05_meanbin,TiteSeq_06_meanbin,TiteSeq_07_meanbin,TiteSeq_08_meanbin,
                                                                                                       TiteSeq_09_meanbin,TiteSeq_10_meanbin,TiteSeq_11_meanbin,TiteSeq_12_meanbin,
                                                                                                       TiteSeq_13_meanbin,TiteSeq_14_meanbin,TiteSeq_15_meanbin,TiteSeq_16_meanbin),
                                                                                              x.vals=samples_lib1$conc,
                                                                                              count.vals=c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,
                                                                                                           TiteSeq_04_totalcount,TiteSeq_05_totalcount,TiteSeq_06_totalcount,
                                                                                                           TiteSeq_07_totalcount,TiteSeq_08_totalcount,TiteSeq_09_totalcount,
                                                                                                           TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                                                                                           TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,
                                                                                                           TiteSeq_16_totalcount)),
                                                                           error=function(e){list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))}),by=barcode]

counts_lib1_unfiltered <- copy(counts_lib1) #copy unfiltered data frame, since we'll be filtering some observations downstream

#repeat for lib2
fit.titration.lib2 <- function(y.vals,x.vals,count.vals,min.cfu=cutoff,min.means=0.6,min.average=5,Kd.start=2e-11,a.start=3,a.lower=1.5,a.upper=3,b.start=1,b.lower=1,b.upper=1.5){
  indices <- count.vals>min.cfu
  y <- y.vals[indices]
  x <- x.vals[indices]
  w <- 1/sapply(count.vals[indices], est.var.lib2)
  if((length(y) < min.means*length(y.vals)) | (mean(count.vals,na.rm=T) < min.average)){ #return NAs if < min.means fraction of concentrations have above min.cfu counts or if the average count across all concentrations is below min.average
    return(list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA)))
  }else{
    fit <- nls(y ~ a*(x/(x+Kd))+b,
               start=list(a=a.start,b=b.start,Kd=Kd.start),
               lower=list(a=a.lower,b=b.lower,Kd=min(x.vals[x.vals>0])/100), #constrain Kd to be no lower than 100x the lowest concentration value
               upper=list(a=a.upper,b=b.upper,Kd=max(x.vals[x.vals>0])*100), #constrain Kd to be no higher than 100x the highest concentration value
               weights=w,algorithm="port")  
    return(list(as.numeric(summary(fit)$coefficients["Kd","Estimate"]),
                as.numeric(summary(fit)$coefficients["Kd","Std. Error"]),
                as.numeric(summary(fit)$coefficients["a","Estimate"]),
                as.numeric(summary(fit)$coefficients["b","Estimate"]),
                as.numeric(summary(fit)$sigma),list(fit)))
  }
}

#fit titration to lib2 Titeseq data for each barcode
counts_lib2[,c("Kd","Kd_SE","response","baseline","RSE","fit") := tryCatch(fit.titration.lib2(y.vals=c(TiteSeq_01_meanbin,TiteSeq_02_meanbin,TiteSeq_03_meanbin,TiteSeq_04_meanbin,
                                                                                                       TiteSeq_05_meanbin,TiteSeq_06_meanbin,TiteSeq_07_meanbin,TiteSeq_08_meanbin,
                                                                                                       TiteSeq_09_meanbin,TiteSeq_10_meanbin,TiteSeq_11_meanbin,TiteSeq_12_meanbin,
                                                                                                       TiteSeq_13_meanbin,TiteSeq_14_meanbin,TiteSeq_15_meanbin,TiteSeq_16_meanbin),
                                                                                              x.vals=samples_lib2$conc,
                                                                                              count.vals=c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,
                                                                                                           TiteSeq_04_totalcount,TiteSeq_05_totalcount,TiteSeq_06_totalcount, 
                                                                                                           TiteSeq_07_totalcount,TiteSeq_08_totalcount,TiteSeq_09_totalcount,
                                                                                                           TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                                                                                           TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,
                                                                                                           TiteSeq_16_totalcount)),
                                                                           error=function(e){list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))}),by=barcode]

counts_lib2_unfiltered <- copy(counts_lib2) #copy unfiltered data frame, since we'll be filtering some observations downstream
```

## QC and sanity checks

We will do some QC to make sure we got good titration curves for most of
our library barcodes. We will also spot check titration curves from
across our measurement range, and spot check curves whose fit parameters
hit the different boundary conditions of the fit variables.

We successfully generated *K*<sub>D,app</sub> estimates for 79583 of our
lib1 barcodes (79.86%) and 78651 of our lib2 barcodes (80.51%).

Why were estimates not returned for some barcodes? The histograms below
show that many barcodes with unsuccessful titration fits have lower
average cell counts and more concentrations with fewer than the minimum
cutoff number of cells (cutoff=2) than those that were fit. Therefore,
we can see the the majority of unfit barcodes come from our minimum read
cutoffs, meaning there weren’t too many curves that failed to be fit for
issues such as nls convergence.

``` r
par(mfrow=c(2,2))
hist(log10(counts_lib1[!is.na(Kd),avgcount]+0.5),breaks=20,xlim=c(0,5),main="lib1",col="gray50",xlab="average cell count across concentration samples")
hist(log10(counts_lib1[is.na(Kd),avgcount]+0.5),breaks=20,add=T,col="red")

hist(log10(counts_lib2[!is.na(Kd),avgcount]+0.5),breaks=20,xlim=c(0,5),main="lib2",col="gray50",xlab="average cell count across concentration samples")
hist(log10(counts_lib2[is.na(Kd),avgcount]+0.5),breaks=20,add=T,col="red")

hist(counts_lib1[!is.na(Kd),min_cell_filtered],breaks=5,main="lib1",col="gray50",xlab="number of sample concentrations below cutoff cell number",xlim=c(0,16))
hist(counts_lib1[is.na(Kd),min_cell_filtered],breaks=16,add=T,col="red")

hist(counts_lib2[!is.na(Kd),min_cell_filtered],breaks=5,main="lib2",col="gray50",xlab="number of sample concentrations below cutoff cell number",xlim=c(0,16))
hist(counts_lib2[is.na(Kd),min_cell_filtered],breaks=16,add=T,col="red")
```

<img src="compute_binding_Kd_files/figure-gfm/avgcount-1.png" style="display: block; margin: auto;" />

Let’s checkout what the data looks like for some curves that didn’t
converge on a titration fit, particularly when having high cell counts.
I define functions that take a row from one of the `counts_libX` data
tables and plot the meanbin estimates and the fit titration curve (if
converged). This allows for quick and easy troubleshooting and
spot-checking of curves if the `counts_lib1` and `_lib2` data tables are
loaded up in an interactive session.

In the plots below for non-converging fits, we can see that the data
seem to have very low plateaus/signal over the concentration range and
perhaps some noise. I understand why they are difficult to fit, and I am
not worried by their exclusion, as I can’t by eye tell what their fit
should be hitting. My best guess is they would have a “response”
parameter lower than the minimum allowable, but that is also a hard Kd
then to estimate reliably so I’m ok not fitting these relatively small
number of curves.

``` r
#make functions that allows me to plot a titration for any given row from the counts data frames, for spot checking curves
plot.titration.lib1 <- function(row,output.text=F){
  y.vals <- c();for(sample in samples_lib1$sample){y.vals <- c(y.vals,paste(sample,"_meanbin",sep=""))};y.vals <- unlist(counts_lib1[row,y.vals,with=F])
  x.vals <- samples_lib1$conc
  count.vals <- c();for(sample in samples_lib1$sample){count.vals <- c(count.vals,paste(sample,"_totalcount",sep=""))};count.vals <- unlist(counts_lib1[row,count.vals,with=F])
  plot(x.vals[count.vals>cutoff],y.vals[count.vals>cutoff],xlab="[ACE2] (M)",
       ylab="mean bin",log="x",ylim=c(1,4),xlim=c(1e-13,1e-6),pch=19,main=counts_lib1[row,aa_substitutions])
  fit <- counts_lib1[row,fit[[1]]]
  if(!is.na(fit)[1]){
    lines(x.vals,predict(fit,newdata=list(x=x.vals)))
    legend("topleft",bty="n",cex=1,legend=paste("Kd",format(counts_lib1[row,Kd],digits=3),"M"))
  }
  if(output.text==T){ #for troubleshooting and interactive work, output some info from the counts table for the given row
    counts_lib1[row,.(barcode,variant_class,aa_substitutions,avgcount,min_cell_filtered,Kd,Kd_SE,baseline,response,RSE)]
  }
}

plot.titration.lib2 <- function(row, output.text=F){
  y.vals <- c();for(sample in samples_lib2$sample){y.vals <- c(y.vals,paste(sample,"_meanbin",sep=""))};y.vals <- unlist(counts_lib2[row,y.vals,with=F])
  x.vals <- samples_lib2$conc
  count.vals <- c();for(sample in samples_lib2$sample){count.vals <- c(count.vals,paste(sample,"_totalcount",sep=""))};count.vals <- unlist(counts_lib2[row,count.vals,with=F])
  plot(x.vals[count.vals>cutoff],y.vals[count.vals>cutoff],xlab="[ACE2] (M)",
       ylab="mean bin",log="x",ylim=c(1,4),xlim=c(1e-13,1e-6),pch=19,main=counts_lib2[row,aa_substitutions])
  fit <- counts_lib2[row,fit[[1]]]
  if(!is.na(fit)[1]){
    lines(x.vals,predict(fit,newdata=list(x=x.vals)))
    legend("topleft",bty="n",cex=1,legend=paste("Kd",format(counts_lib2[row,Kd],digits=3),"M"))  
  }
  if(output.text==T){ #for troubleshooting and interactive work, output some info from the counts table for the given row
    counts_lib2[row,.(barcode,variant_class,aa_substitutions,avgcount,min_cell_filtered,Kd,Kd_SE,baseline,response,RSE)]
  }
}

par(mfrow=c(2,2))
#checkout the points for some barcodes that had high coverage but no fit
plot.titration.lib1(which(counts_lib1$avgcount > 50 & is.na(counts_lib1$Kd))[1])
plot.titration.lib1(which(counts_lib1$avgcount > 50 & is.na(counts_lib1$Kd))[2])
plot.titration.lib2(which(counts_lib2$avgcount > 50 & is.na(counts_lib2$Kd))[1])
plot.titration.lib2(which(counts_lib2$avgcount > 50 & is.na(counts_lib2$Kd))[2])
```

<img src="compute_binding_Kd_files/figure-gfm/check_failed_titrations-1.png" style="display: block; margin: auto;" />

Some stop variants eked through our RBD+ selection, either perhaps
because of stop codon readthrough, improper PacBio sequence annotation,
or other weirdness. Either way, the vast majority of nonsense mutants
were purged before this step, and the remaining ones are unreliable and
muddy the waters\!

``` r
#remove stop variants, which even if they eke through, either a) still have low counts and give poor fits as a result, or b) seem to be either dubious PacBio calls (lower variant_call_support) or have late stop codons which perhaps don't totally ablate funciton. Either way, the vast majority were purged before this step and we don't want to deal with the remaining ones!
counts_lib1[variant_class == "stop",c("Kd","Kd_SE","response","baseline","RSE","fit") := list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))]
counts_lib2[variant_class == "stop",c("Kd","Kd_SE","response","baseline","RSE","fit") := list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))]
```

Also, downstream vetting showed that there is one curve in lib2 that is
fit to the minimum boundary of 10<sup>-15</sup>. It’s curve is
visualizied below. It does have a mean bin of 4 at the lowest sample
concentration, but it also has many missing observations and is just
above the average of 5 cell count per mean bin estimate. I am therefore
going to veto this curve and just manually censor it given its extremity
on Kd, cell count, and min\_cell\_filtered boundary conditions.

``` r
plot.titration.lib2(which(counts_lib2$Kd == 1e-15)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-15_Kd-1.png" style="display: block; margin: auto;" />

``` r
counts_lib2[counts_lib2$Kd==1e-15,c("Kd","Kd_SE","response","baseline","RSE","fit") := list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))]
```

Next, let’s look at our distribution of *K*<sub>D,app</sub> estimates.
We can see below that the distribution of wildtype barcodes (purple) is
very tight. The homologs in the library (lumped together in blue) show
an expected variation: some of the homologs have similar or very
slightly reduced or enhanced affinity relative to WT SARS-CoV-2, one
shows an intermediate level of binding (RaTG13), and the clade2
genotypes show no binding. The mutant genotypes of SARS-CoV-2 span the
spectrum here, with the large bar at 10<sup>-4</sup> reflecting barocdes
that had no response even at the highest concentration and thus were
assigned the highest allowable value per the fitting constraints.

``` r
par(mfrow=c(1,2))
hist(log10(counts_lib1[variant_class %in% (c("1 nonsynonymous",">1 nonsynonymous")),Kd]),col="gray40",breaks=60,xlab="log10(K_D,app) (M)",main="lib1")
hist(log10(counts_lib1[variant_class %in% (c("synonymous","wildtype")),Kd]),col="#92278F",add=T,breaks=20)
hist(log10(counts_lib1[variant_class == target,Kd]),col="#2E3192",add=T,breaks=60)
#hist(log10(counts_lib1[variant_class %in% (c("stop")),Kd]),col="#BE1E2D",add=T,breaks=50)

hist(log10(counts_lib2[variant_class %in% (c("1 nonsynonymous",">1 nonsynonymous")),Kd]),col="gray40",breaks=60,xlab="log10(K_D,app) (M)",main="lib2")
hist(log10(counts_lib2[variant_class %in% (c("synonymous","wildtype")),Kd]),col="#92278F",add=T,breaks=30)
hist(log10(counts_lib2[variant_class == target,Kd]),col="#2E3192",add=T,breaks=60)
```

<img src="compute_binding_Kd_files/figure-gfm/Kd_distribution-1.png" style="display: block; margin: auto;" />

``` r
#save pdf
invisible(dev.print(pdf, paste(config$Titeseq_Kds_dir,"/hist_Kd-per-barcode.pdf",sep="")))
```

Let’s take a look at some of the curves with *K*<sub>D,app</sub> values
across this distribution to get a broad sense of how things look.

First, curves with *K*<sub>D,app</sub> fixed at the 10<sup>-4</sup>
maximum. We can see these are all flat-lined curves with no response.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd==max(counts_lib1$Kd,na.rm=T))[1])
plot.titration.lib2(which(counts_lib2$Kd==max(counts_lib2$Kd,na.rm=T))[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-4_Kd-1.png" style="display: block; margin: auto;" />

Next, with *K*<sub>D,app</sub> around 10<sup>-5</sup>. These
10<sup>-5</sup> curves are honestly probably not any different than
those censored to 10<sup>-4</sup> – the curve just decides to begin to
turn right at the end, perhaps sometimes with genuine signal but perhaps
often because of noise.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-5 & counts_lib1$Kd < 1.2e-5)[1])
plot.titration.lib2(which(counts_lib2$Kd > 1e-5 & counts_lib2$Kd < 1.2e-5)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-5_Kd-1.png" style="display: block; margin: auto;" />

Next, with *K*<sub>D,app</sub> around 10<sup>-6</sup>. These
10<sup>-6</sup> curves are similar to the 10<sup>-5</sup>, though
perhaps a bit more belieivable that the curve is bending at the highest
concentration. Overall, I think that differences in *K*<sub>D,app</sub>
within this large mode in the distribution of all variant
*K*<sub>D,app</sub>s probably isn’t reporting on much meaningful
variation.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-6 & counts_lib1$Kd < 1.2e-6)[1])
plot.titration.lib2(which(counts_lib2$Kd > 1e-6 & counts_lib2$Kd < 1.2e-6)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-6_Kd-1.png" style="display: block; margin: auto;" />

With *K*<sub>D,app</sub> around 10<sup>-7</sup>, we seem to be picking
up more consistent signals. Many of these curves still are showing
noise, but not as consistently as the prior curves. (Cherry picked an
example below which is not the \[1\] index example – this is just to
show that some of the curves I spot-checked aren’t half bad. But this is
not representative of all spot-checks.)

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-7 & counts_lib1$Kd < 1.2e-7)[2])
plot.titration.lib2(which(counts_lib2$Kd > 1e-7 & counts_lib2$Kd < 1.2e-7)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-7_Kd-1.png" style="display: block; margin: auto;" />

At *K*<sub>D,app</sub> of 10<sup>-8</sup>, we are likewise picking up
some signal but also many curves with lots of noise.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-8 & counts_lib1$Kd < 1.2e-8)[3])
plot.titration.lib2(which(counts_lib2$Kd > 1e-8 & counts_lib2$Kd < 1.2e-8)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-8_Kd-1.png" style="display: block; margin: auto;" />

At *K*<sub>D,app</sub> of 10<sup>-9</sup>, now we are starting to get
more consistent, genuine signal, with the curves looking *much* better.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-9 & counts_lib1$Kd < 1.2e-9)[1])
plot.titration.lib2(which(counts_lib2$Kd > 1e-9 & counts_lib2$Kd < 1.2e-9)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-9_Kd-1.png" style="display: block; margin: auto;" />

At *K*<sub>D,app</sub> of 10<sup>-10</sup>, curves are looking
beautiful.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-10 & counts_lib1$Kd < 1.2e-10)[1])
plot.titration.lib2(which(counts_lib2$Kd > 1e-10 & counts_lib2$Kd < 1.2e-10)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-10_Kd-1.png" style="display: block; margin: auto;" />

As do curves with *K*<sub>D,app</sub> \~ 10<sup>-11</sup>.

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd > 1e-11 & counts_lib1$Kd < 1.2e-11)[1])
plot.titration.lib2(which(counts_lib2$Kd > 1e-11 & counts_lib2$Kd < 1.2e-11)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-11_Kd-1.png" style="display: block; margin: auto;" />

Now, as we get to the other side of the bulk of the distribution, for
curves showing improved affinities around *K*<sub>D,app</sub> of
10<sup>-12</sup>, we are once again seeing mainly noise, and in
particular, curves with low cell counts, large number of missing meanbin
samples, and correspondingly high residuals (so will likely be
eliminated with our filtering step later on).

``` r
par(mfrow=c(1,2))
plot.titration.lib1(which(counts_lib1$Kd < 5e-12)[1])
plot.titration.lib2(which(counts_lib2$Kd < 5e-12)[1])
```

<img src="compute_binding_Kd_files/figure-gfm/1e-12_Kd-1.png" style="display: block; margin: auto;" />

Next, let’s spot check curves at the boundary conditions for the
baseline and response fit values, which are constrained by the model fit
to fall within certain ranges.

The plots below show the histograms of the titration fit parameter
corresponding to the titration baseline, followed by two example
titrations that are fixed to the minimal, and then the maximal allowed
value for this parameter. These curves aren’t always mind-blowingly
good, but at least the baseline fits don’t seem egregious/divergent from
the actual data due to the constrained parameter ranges.

``` r
par(mfrow=c(3,2))
#first, histograms for the baseline fit parameter
hist(counts_lib1$baseline,col="gray50",main="lib1",xlab="baseline fit parameter")
hist(counts_lib2$baseline,col="gray50",main="lib2",xlab="baseline fit parameter")

#check out fits with the minimum baseline=1
#variability with what's going on in the curves, but the baselines seem fine
plot.titration.lib1(which(counts_lib1$baseline==1.0)[2])
plot.titration.lib2(which(counts_lib2$baseline==1.0)[3])

#check out fits with the maximum baseline=1.5
#seem fine with the higher baseline, one commonality is perhaps some missing concentrations due to min cells filter
plot.titration.lib1(which(counts_lib1$baseline==1.5)[2])
plot.titration.lib2(which(counts_lib2$baseline==1.5)[3])
```

<img src="compute_binding_Kd_files/figure-gfm/spot_check_baseline-1.png" style="display: block; margin: auto;" />

Similarly, the plots below show the histograms of the titration fit
parameter corresponding to the titration response (the difference
between the plateau and baseline), followed by two example titrations
that are fixed to the minimal, and then the maximal allowed value for
this parameter. Once again, these response calls don’t seem
egregious/divergent from the actual data.

``` r
par(mfrow=c(3,2))
#next, histograms for the response fit parameter
hist(counts_lib1$response,col="gray50",main="lib1",xlab="response fit parameter")
hist(counts_lib2$response,col="gray50",main="lib2",xlab="response fit parameter")

#check out fits with the minimum response=1.5
#some seem to be the curves that begin to blip right at the end, others like shown perhaps do have lower response?
plot.titration.lib1(which(counts_lib1$response==1.5)[3])
plot.titration.lib2(which(counts_lib2$response==1.5)[3])

#check out fits with the maximum response=3.0
#many are just extrapolating the maximum response, though some are real curves (like one example below)
plot.titration.lib1(which(counts_lib1$response==3)[3])
plot.titration.lib2(which(counts_lib2$response==3)[2])
```

<img src="compute_binding_Kd_files/figure-gfm/spot_check_response-1.png" style="display: block; margin: auto;" />

## Data filtering

Next, let’s compute a quality metric for each curve fit, and filter out
poor fits from our final dataset for downstream analyses. For each curve
fit, we will compute a *normalized* mean square residual (nMSR). This
metric computes the residual between the observed response variable and
that predicted from the titration fit, normalizes this residual by the
response range of the titration fit (which is allowed to vary between
1.5 and 3 per the titration fits above), and computes the mean-square of
these normalized residuals.

``` r
#function to calculate mean squared residual normalized to response range
calc.nMSR <- function(y.obs,x.vals,count.vals,response,fit,cfu.cutoff=cutoff){
  indices <- count.vals>cfu.cutoff
  y.obs <- y.obs[indices]
  x.vals <- x.vals[indices]
  y.pred <- predict(fit,newdata=list(x=x.vals))
  resid <- y.obs - y.pred
  resid.norm <- resid/response
  return(mean((resid.norm)^2))
}

#calculate normalized MSR for each fit
#lib1
counts_lib1[!is.na(Kd), nMSR := calc.nMSR(y.obs=c(TiteSeq_01_meanbin,TiteSeq_02_meanbin,TiteSeq_03_meanbin,TiteSeq_04_meanbin,
                                                  TiteSeq_05_meanbin,TiteSeq_06_meanbin,TiteSeq_07_meanbin,TiteSeq_08_meanbin,
                                                  TiteSeq_09_meanbin,TiteSeq_10_meanbin,TiteSeq_11_meanbin,TiteSeq_12_meanbin,
                                                  TiteSeq_13_meanbin,TiteSeq_14_meanbin,TiteSeq_15_meanbin,TiteSeq_16_meanbin),
                                        x.vals=samples_lib1$conc,
                                        count.vals=c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,TiteSeq_04_totalcount,
                                                     TiteSeq_05_totalcount,TiteSeq_06_totalcount,TiteSeq_07_totalcount,TiteSeq_08_totalcount,
                                                     TiteSeq_09_totalcount,TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                                     TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,TiteSeq_16_totalcount),
                                        response=response,
                                        fit=fit[[1]]),by=barcode]
#lib2
counts_lib2[!is.na(Kd), nMSR := calc.nMSR(y.obs=c(TiteSeq_01_meanbin,TiteSeq_02_meanbin,TiteSeq_03_meanbin,TiteSeq_04_meanbin,
                                                  TiteSeq_05_meanbin,TiteSeq_06_meanbin,TiteSeq_07_meanbin,TiteSeq_08_meanbin,
                                                  TiteSeq_09_meanbin,TiteSeq_10_meanbin,TiteSeq_11_meanbin,TiteSeq_12_meanbin,
                                                  TiteSeq_13_meanbin,TiteSeq_14_meanbin,TiteSeq_15_meanbin,TiteSeq_16_meanbin),
                                        x.vals=samples_lib2$conc,
                                        count.vals=c(TiteSeq_01_totalcount,TiteSeq_02_totalcount,TiteSeq_03_totalcount,TiteSeq_04_totalcount,
                                                     TiteSeq_05_totalcount,TiteSeq_06_totalcount,TiteSeq_07_totalcount,TiteSeq_08_totalcount,
                                                     TiteSeq_09_totalcount,TiteSeq_10_totalcount,TiteSeq_11_totalcount,TiteSeq_12_totalcount,
                                                     TiteSeq_13_totalcount,TiteSeq_14_totalcount,TiteSeq_15_totalcount,TiteSeq_16_totalcount),
                                        response=response,
                                        fit=fit[[1]]),by=barcode]
```

Next, let’s look at the normalized MSR metric.

``` r
par(mfrow=c(3,2))
hist(counts_lib1$nMSR,main="lib1",xlab="Response-normalized mean squared residual",col="gray50",breaks=40)
hist(counts_lib2$nMSR,main="lib2",xlab="Response-normalized mean squared residual",col="gray50",breaks=30)

# #how does the normalized MSR compare to the MSR computed? Plot as scatter, color points by response fit
# ggplot(counts_lib1,aes(x=MSR,y=nMSR,color=response))+geom_point()+scale_color_gradient(low="#FFFF0030",high="#0000FF")+ggtitle("lib1")
# ggplot(counts_lib2,aes(x=MSR,y=nMSR,color=response))+geom_point()+scale_color_gradient(low="#FFFF0030",high="#0000FF")+ggtitle("lib2")

#As we would expect, the MSR stat decreases with cell count, indicating that higher cell counts leads to better curve fits
plot(log10(counts_lib1$avgcount),counts_lib1$nMSR,pch=19,col="#00000010",main="lib1",xlab="average cell count (log10)",ylab="nMSR",xlim=c(0,5))
plot(log10(counts_lib2$avgcount),counts_lib2$nMSR,pch=19,col="#00000010",main="lib2",xlab="average cell count (log10)",ylab="nMSR",xlim=c(0,5))

#MSR is still higher for curves with lower Kd -- makes sense, becuase opportunity for residuals is higher when there is actual response, and therefore variation to have residuals on
plot(log10(counts_lib1$Kd),counts_lib1$nMSR,pch=19,col="#00000010",main="lib1",xlab="Kd",ylab="nMSR")
plot(log10(counts_lib2$Kd),counts_lib2$nMSR,pch=19,col="#00000010",main="lib2",xlab="Kd",ylab="nMSR")
```

<img src="compute_binding_Kd_files/figure-gfm/nMSR_distribution-1.png" style="display: block; margin: auto;" />
Let’s see what titration curves look like in different nMSR regimes.
First, let’s see two titrations from each library at the median nMSR.
Suggests that the ‘typical’ curve looks quite nice.

``` r
par(mfrow=c(2,2))
plot.titration.lib1(which(counts_lib1$nMSR > quantile(counts_lib1$nMSR,0.49,na.rm=T) & counts_lib1$nMSR < quantile(counts_lib1$nMSR,0.51,na.rm=T))[1])
plot.titration.lib1(which(counts_lib1$nMSR > quantile(counts_lib1$nMSR,0.49,na.rm=T) & counts_lib1$nMSR < quantile(counts_lib1$nMSR,0.51,na.rm=T))[2])

plot.titration.lib2(which(counts_lib2$nMSR > quantile(counts_lib2$nMSR,0.49,na.rm=T) & counts_lib2$nMSR < quantile(counts_lib2$nMSR,0.51,na.rm=T))[1])
plot.titration.lib2(which(counts_lib2$nMSR > quantile(counts_lib2$nMSR,0.49,na.rm=T) & counts_lib2$nMSR < quantile(counts_lib2$nMSR,0.51,na.rm=T))[2])
```

<img src="compute_binding_Kd_files/figure-gfm/example_curves_median_nMSR-1.png" style="display: block; margin: auto;" />

If we set a cutoff of filtering out the worst 5% of curves based on
nMSR, what would the borderline cases look like that we would be
retaining?

These curves do indeed look quite noisy\! However, these represent the
*worst* curves that we would be keeping, and I’m ok with that.
Especially compared to the types of curves we would be eliminating,
shown in the next section.

``` r
par(mfrow=c(2,2))
plot.titration.lib1(which(counts_lib1$nMSR > quantile(counts_lib1$nMSR,0.945,na.rm=T) & counts_lib1$nMSR < quantile(counts_lib1$nMSR,0.955,na.rm=T))[1])
plot.titration.lib1(which(counts_lib1$nMSR > quantile(counts_lib1$nMSR,0.945,na.rm=T) & counts_lib1$nMSR < quantile(counts_lib1$nMSR,0.955,na.rm=T))[2])

plot.titration.lib2(which(counts_lib2$nMSR > quantile(counts_lib2$nMSR,0.945,na.rm=T) & counts_lib2$nMSR < quantile(counts_lib2$nMSR,0.955,na.rm=T))[1])
plot.titration.lib2(which(counts_lib2$nMSR > quantile(counts_lib2$nMSR,0.945,na.rm=T) & counts_lib2$nMSR < quantile(counts_lib2$nMSR,0.955,na.rm=T))[2])
```

<img src="compute_binding_Kd_files/figure-gfm/example_curves_cutoff_nMSR-1.png" style="display: block; margin: auto;" />

Now, let’s do a tour-de-crap, and see some example curves that will be
eliminated by this 5% nMSR cutoff. For each library, we show one example
curve from the *K*<sub>D,app</sub> \<10<sup>-8</sup> range, and one from
the \>10<sup>-6</sup> range. Very happy to be removing these\!

``` r
par(mfrow=c(2,2))
plot.titration.lib1(which(counts_lib1$nMSR > 0.2 & counts_lib1$Kd < 10^-8)[3])
plot.titration.lib1(which(counts_lib1$nMSR > 0.2 & counts_lib1$Kd > 10^-6)[1])

plot.titration.lib2(which(counts_lib2$nMSR > 0.2 & counts_lib2$Kd < 10^-8)[1])
plot.titration.lib2(which(counts_lib2$nMSR > 0.2 & counts_lib2$Kd > 10^-6)[2])
```

<img src="compute_binding_Kd_files/figure-gfm/example_curves_eliminated_nMSR-1.png" style="display: block; margin: auto;" />

Next, we will apply this filtering step on normalized MSR, removing the
worst 5% of curves on this metric.

``` r
counts_lib1[nMSR > quantile(counts_lib1$nMSR,0.95,na.rm=T),c("Kd","Kd_SE","response","baseline","RSE","fit") := list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))]
counts_lib2[nMSR > quantile(counts_lib2$nMSR,0.95,na.rm=T),c("Kd","Kd_SE","response","baseline","RSE","fit") := list(as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.numeric(NA),as.list(NA))]
```

This leaves us with filtered *K*<sub>D,app</sub> estimates for 74917 of
our lib1 barcodes (75.18%) and 73691 of our lib2 barcodes (75.43%).

As a final filtering step, we noticed above that curves with
*K*<sub>D,app</sub> between 10<sup>-6</sup> and 10<sup>-4</sup> were
more or less indistinguishable. This is consistent with our top labeling
concentration being 10<sup>-6</sup>. Therefore, we will squash all
*K*<sub>D,app</sub> estimates in this range to a censored at
10<sup>-6</sup>.

``` r
counts_lib1[Kd > 10^-6,Kd := 10^-6]
counts_lib2[Kd > 10^-6,Kd := 10^-6]
```

Last, let’s convert our *K*<sub>D,app</sub> to 1) a
log<sub>10</sub>-scale, where mutations are expected to combine
additively (instead of multiplicatively), and 2) *K*<sub>A,app</sub>,
the inverse of *K*<sub>D,app</sub>, such that higher values are
associated with tighter binding, as is more intuitive. (If we want to
continue to discuss in terms of *K*<sub>D,app</sub>, since people are
often more familiar with *K*<sub>D</sub>, we can refer to the
log<sub>10</sub>(*K*<sub>A,app</sub>) as
-log<sub>10</sub>(*K*<sub>D,app</sub>), which are identical.

``` r
counts_lib1[,log10Kd := log10(Kd),by=barcode]
counts_lib2[,log10Kd := log10(Kd),by=barcode]

counts_lib1[,log10Ka := -log10Kd,by=barcode]
counts_lib2[,log10Ka := -log10Kd,by=barcode]

#error propagation of the SE of the Kd fit onto the log10 scale
counts_lib1[,log10SE := 0.434*Kd_SE/Kd,by=barcode]
counts_lib2[,log10SE := 0.434*Kd_SE/Kd,by=barcode]
```

Some curve fits derived a nonsensically high standard error on the
*K*<sub>D,app</sub> estimate, with it being orders and orders of
magnitude larger than the *K*<sub>D,app</sub> estimate itself (see
histogram below, which is already bounded – without bounds the largest
errors extend out to 10<sup>9</sup>, and this is already on the log10
scale\!). This appears to happen in fits where *K*<sub>D,app</sub> is at
the maximum of the range – which makes sense then that there would be
large variance in this estimate, since it’s just extrapolating out that
*K*<sub>D,app</sub> is *somewhere* higher than 10<sup>-6</sup>, without
information as to *where* in this range the curve will actually respond.

In our case, we may be usinig these SE estimaes in global epistasis
model fits. (We don’t use them for any other purpose). In that case, we
don’t want the global epistasis model to be able to use these large
variances to “put” these measurements wherever it needs to to minimize
fitting errors – we’d rather that it take the 10<sup>-6</sup>
*K*<sub>D,app</sub> values as provided without crazy high variance, and
use the nonlinear curve fit portion of the model to “learn” that these
10<sup>-6</sup> values are really a highly censored indicator of values
somewhere at 10<sup>-6</sup> or higher. Therefore, we are going to
squash these standard error measures to a maximum value, as well.

``` r
par(mfrow=c(3,2))
hist(counts_lib1$log10SE[counts_lib1$log10SE < 5],breaks=20,main="lib1",xlab="Standard Error of log10(K_A)")
hist(counts_lib2$log10SE[counts_lib2$log10SE < 5],breaks=20,main="lib2",xlab="Standard Error of log10(K_A)")

plot(counts_lib1$log10Ka,counts_lib1$log10SE,pch=19,col="#00000050",ylim=c(0,5),main="lib1",xlab="log10(Ka)",ylab="SE on log10(Ka)")
plot(counts_lib2$log10Ka,counts_lib2$log10SE,pch=19,col="#00000050",ylim=c(0,5),main="lib2",xlab="log10(Ka)",ylab="SE on log10(Ka)")

#for now, fix log10SE values >2 to be max 2 
counts_lib1[log10SE>2, log10SE:=2]
counts_lib2[log10SE>2, log10SE:=2]

hist(counts_lib1$log10SE,breaks=20,main="lib1",xlab="Standard Error of log10(K_A)")
hist(counts_lib2$log10SE,breaks=20,main="lib2",xlab="Standard Error of log10(K_A)")
```

<img src="compute_binding_Kd_files/figure-gfm/censor_log10SE_max-1.png" style="display: block; margin: auto;" />

Let’s visualize the final binding measurements as violin plots, faceted
by variant class. Repeat separately for variant classes of SARS-CoV-2,
and one for the different RBD homologs.

``` r
p1 <- ggplot(counts_lib1[target=="SARS-CoV-2" & !is.na(log10Ka),],aes(x=variant_class,y=log10Ka))+
  geom_violin(scale="width")+stat_summary(fun.y=median,geom="point",size=1)+
  ggtitle("lib1")+xlab("variant class")+theme(axis.text.x=element_text(angle=-45,hjust=0))+
  scale_y_continuous(limits=c(6,12))

p2 <- ggplot(counts_lib2[target=="SARS-CoV-2" & !is.na(log10Ka),],aes(x=variant_class,y=log10Ka))+
  geom_violin(scale="width")+stat_summary(fun.y=median,geom="point",size=1)+
  ggtitle("lib2")+xlab("variant class")+theme(axis.text.x=element_text(angle=-45,hjust=0))+
  scale_y_continuous(limits=c(6,12))

p3 <- ggplot(counts_lib1[!is.na(log10Ka) & !(variant_class %in% c("synonymous","1 nonsynonymous",">1 nonsynonymous")),],aes(x=target,y=log10Ka))+
  geom_violin(scale="width")+stat_summary(fun.y=median,geom="point",size=0.5)+
  ggtitle("lib1")+xlab("variant class")+theme(axis.text.x=element_text(angle=-45,hjust=0))+
  scale_y_continuous(limits=c(6,12))

p4 <- ggplot(counts_lib2[!is.na(log10Ka) & !(variant_class %in% c("synonymous","1 nonsynonymous",">1 nonsynonymous")),],aes(x=target,y=log10Ka))+
  geom_violin(scale="width")+stat_summary(fun.y=median,geom="point",size=0.5)+
  ggtitle("lib2")+xlab("variant class")+theme(axis.text.x=element_text(angle=-45,hjust=0))+
  scale_y_continuous(limits=c(6,12))

grid.arrange(p1,p2,p3,p4,ncol=2)
```

<img src="compute_binding_Kd_files/figure-gfm/final_pheno_DFE-1.png" style="display: block; margin: auto;" />

``` r
#save pdf
invisible(dev.print(pdf, paste(config$Titeseq_Kds_dir,"/violin-plot_log10Ka-by-target.pdf",sep="")))
```

## Data Output

Finally, let’s output our measurements for downstream analyses. Since
only the SARS-CoV-2 data is going into the global epistasis models, we
will output separate files, for all barcodes, and for barcodes for
SARS-CoV-2 targets only

``` r
counts_lib1[,library:="lib1"]
counts_lib2[,library:="lib2"]

write.csv(rbind(counts_lib1[,.(library, target, barcode, variant_call_support, avgcount, log10Ka, log10SE, Kd, Kd_SE, response,
                               baseline, nMSR, variant_class, aa_substitutions, n_aa_substitutions)],
                counts_lib2[,.(library, target, barcode, variant_call_support, avgcount, log10Ka, log10SE, Kd, Kd_SE, response,
                               baseline, nMSR, variant_class, aa_substitutions, n_aa_substitutions)]),
          file=config$Titeseq_Kds_all_targets_file)

write.csv(rbind(counts_lib1[target=="SARS-CoV-2",.(library, target, barcode, variant_call_support, avgcount, log10Ka, log10SE, Kd, Kd_SE, response,
                                                   baseline, nMSR, variant_class, aa_substitutions, n_aa_substitutions)],
                counts_lib2[target=="SARS-CoV-2",.(library, target, barcode, variant_call_support, avgcount, log10Ka, log10SE, Kd, Kd_SE, response,
                                                   baseline, nMSR, variant_class, aa_substitutions, n_aa_substitutions)]),
          file=config$Titeseq_Kds_file)
```