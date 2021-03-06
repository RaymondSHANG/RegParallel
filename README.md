Standard regression functions in R enabled for parallel processing over large data-frames
================
Kevin Blighe
2019-01-22

-   [Introduction](#introduction)
-   [Installation](#installation)
    -   [1. Download the package from Bioconductor](#download-the-package-from-bioconductor)
    -   [2. Load the package into R session](#load-the-package-into-r-session)
-   [Quick start](#quick-start)
    -   [Perform the most basic logistic regression analysis](#perform-the-most-basic-logistic-regression-analysis)
    -   [Perform a basic linear regression](#perform-a-basic-linear-regression)
    -   [Perform the most basic negative binomial logistic regression analysis](#perform-the-most-basic-negative-binomial-logistic-regression-analysis)
    -   [Survival analysis via Cox Proportional Hazards regression](#survival-analysis-via-cox-proportional-hazards-regression)
    -   [Perform a conditional logistic regression](#perform-a-conditional-logistic-regression)
-   [Advanced features](#advanced-features)
    -   [Speed up processing](#speed-up-processing)
        -   [~2000 tests; blocksize, 500; cores, 2; nestedParallel, TRUE](#tests-blocksize-500-cores-2-nestedparallel-true)
        -   [~2000 tests; blocksize, 500; cores, 2; nestedParallel, FALSE](#tests-blocksize-500-cores-2-nestedparallel-false)
        -   [~40000 tests; blocksize, 2000; cores, 2; nestedParallel, TRUE](#tests-blocksize-2000-cores-2-nestedparallel-true)
        -   [~40000 tests; blocksize, 2000; cores, 2; nestedParallel, FALSE](#tests-blocksize-2000-cores-2-nestedparallel-false)
        -   [~40000 tests; blocksize, 5000; cores, 3; nestedParallel, TRUE](#tests-blocksize-5000-cores-3-nestedparallel-true)
    -   [Modify confidence intervals](#modify-confidence-intervals)
    -   [Remove some terms from output / include the intercept](#remove-some-terms-from-output-include-the-intercept)
-   [Acknowledgments](#acknowledgments)
-   [Session info](#session-info)
-   [References](#references)

Introduction
============

In many analyses, a large amount of variables have to be tested independently against the trait/endpoint of interest, and also adjusted for covariates and confounding factors at the same time. The major bottleneck in these is the amount of time that it takes to complete these analyses.

With <i>RegParallel</i>, a large number of tests can be performed simultaneously. On a 12-core system, 144 variables can be tested simultaneously, with 1000s of variables processed in a matter of seconds via 'nested' parallel processing.

Works for logistic regression, linear regression, conditional logistic regression, Cox proportional hazards and survival models, Bayesian logistic regression, and negative binomial regression.

Installation
============

1. Download the package from Bioconductor
-----------------------------------------

``` r
    if (!requireNamespace('BiocManager', quietly = TRUE))
        install.packages('BiocManager')
        BiocManager::install('RegParallel')
```

Note: to install development version:

``` r
    devtools::install_github('kevinblighe/RegParallel')
```

2. Load the package into R session
----------------------------------

``` r
    library(RegParallel)
```

Quick start
===========

For this quick start, we will follow the tutorial (from Section 3.1) of [RNA-seq workflow: gene-level exploratory analysis and differential expression](http://master.bioconductor.org/packages/release/workflows/vignettes/rnaseqGene/inst/doc/rnaseqGene.html). Specifically, we will load the 'airway' data, where different airway smooth muscle cells were treated with dexamethasone.

``` r
    library(airway)
    library(magrittr)

    data('airway')
    airway$dex %<>% relevel('untrt')
```

Normalise the raw counts in <i>DESeq2</i> and produce regularised log counts:

``` r
    library(DESeq2)

    dds <- DESeqDataSet(airway, design = ~ dex + cell)
    dds <- DESeq(dds, betaPrior = FALSE)
    rlogcounts <- assay(rlog(dds, blind = FALSE))
    rlogdata <- data.frame(colData(airway), t(rlogcounts))
```

Perform the most basic logistic regression analysis
---------------------------------------------------

Here, we fit a binomial logistic regression model to the data via <i>glmParallel</i>, with dexamethasone as the dependent variable.

``` r
    res1 <- RegParallel(
      data = rlogdata[ ,1:3000],
      formula = 'dex ~ [*]',
      FUN = function(formula, data)
        glm(formula = formula,
          data = data,
          family = binomial(link = 'logit')),
      FUNtype = 'glm',
      variables = colnames(rlogdata)[10:3000])

    res1[order(res1$P, decreasing=FALSE),]
```

    ##              Variable            Term       Beta StandardError
    ##    1: ENSG00000095464 ENSG00000095464   43.27934  2.593463e+01
    ##    2: ENSG00000071859 ENSG00000071859   12.96251  7.890287e+00
    ##    3: ENSG00000069812 ENSG00000069812  -44.37139  2.704021e+01
    ##    4: ENSG00000072415 ENSG00000072415  -19.90841  1.227527e+01
    ##    5: ENSG00000073921 ENSG00000073921   14.59470  8.999831e+00
    ##   ---                                                         
    ## 2817: ENSG00000068831 ENSG00000068831  110.84893  2.729072e+05
    ## 2818: ENSG00000069020 ENSG00000069020 -186.45744  4.603615e+05
    ## 2819: ENSG00000083642 ENSG00000083642 -789.55666  1.951104e+06
    ## 2820: ENSG00000104331 ENSG00000104331  394.14700  9.749138e+05
    ## 2821: ENSG00000083097 ENSG00000083097 -217.48873  5.398191e+05
    ##                   Z          P            OR      ORlower      ORupper
    ##    1:  1.6687854476 0.09515991  6.251402e+18 5.252646e-04 7.440065e+40
    ##    2:  1.6428433092 0.10041536  4.261323e+05 8.190681e-02 2.217017e+12
    ##    3: -1.6409412536 0.10080961  5.367228e-20 5.165170e-43 5.577191e+03
    ##    4: -1.6218306224 0.10483962  2.258841e-09 8.038113e-20 6.347711e+01
    ##    5:  1.6216635641 0.10487540  2.179701e+06 4.761313e-02 9.978541e+13
    ##   ---                                                                 
    ## 2817:  0.0004061781 0.99967592  1.383811e+48 0.000000e+00           NA
    ## 2818: -0.0004050239 0.99967684  1.053326e-81 0.000000e+00           NA
    ## 2819: -0.0004046717 0.99967712  0.000000e+00 0.000000e+00           NA
    ## 2820:  0.0004042891 0.99967742 1.499222e+171 0.000000e+00           NA
    ## 2821: -0.0004028919 0.99967854  3.514359e-95 0.000000e+00           NA

Perform a basic linear regression
---------------------------------

Here, we will perform the linear regression using both <i>glmParallel</i> and <i>lmParallel</i>. We will appreciate that a linear regression is the same using either function with the default settings.

Regularised log counts from our <i>DESeq2</i> data will be used.

``` r
  rlogdata <- rlogdata[ ,1:2000]

  res2 <- RegParallel(
    data = rlogdata,
    formula = '[*] ~ cell',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = colnames(rlogdata)[10:ncol(rlogdata)],
    p.adjust = "none")

  res3 <- RegParallel(
    data = rlogdata,
    formula = '[*] ~ cell',
    FUN = function(formula, data)
      lm(formula = formula,
        data = data),
    FUNtype = 'lm',
    variables = colnames(rlogdata)[10:ncol(rlogdata)],
    p.adjust = "none")

  subset(res2, P<0.05)
```

    ##             Variable        Term        Beta StandardError          t
    ##   1: ENSG00000001461 cellN061011 -0.46859875    0.10526111  -4.451775
    ##   2: ENSG00000001461 cellN080611 -0.84020922    0.10526111  -7.982143
    ##   3: ENSG00000001461  cellN61311 -0.87778101    0.10526111  -8.339082
    ##   4: ENSG00000001561 cellN080611 -1.71802758    0.13649920 -12.586357
    ##   5: ENSG00000001561  cellN61311 -1.05328889    0.13649920  -7.716448
    ##  ---                                                                 
    ## 519: ENSG00000092108 cellN061011 -0.12721659    0.01564082  -8.133625
    ## 520: ENSG00000092108  cellN61311 -0.12451203    0.01564082  -7.960708
    ## 521: ENSG00000092148 cellN080611 -0.34988071    0.10313461  -3.392467
    ## 522: ENSG00000092200 cellN080611  0.05906656    0.01521063   3.883241
    ## 523: ENSG00000092208 cellN080611 -0.28587683    0.08506716  -3.360602
    ##                 P        OR   ORlower   ORupper
    ##   1: 0.0112313246 0.6258787 0.5092039 0.7692873
    ##   2: 0.0013351958 0.4316202 0.3511586 0.5305181
    ##   3: 0.0011301853 0.4157043 0.3382098 0.5109554
    ##   4: 0.0002293465 0.1794197 0.1373036 0.2344544
    ##   5: 0.0015182960 0.3487887 0.2669157 0.4557753
    ##  ---                                           
    ## 519: 0.0012429963 0.8805429 0.8539591 0.9079544
    ## 520: 0.0013489163 0.8829276 0.8562718 0.9104133
    ## 521: 0.0274674209 0.7047722 0.5757851 0.8626549
    ## 522: 0.0177922771 1.0608458 1.0296864 1.0929482
    ## 523: 0.0282890537 0.7513552 0.6359690 0.8876762

``` r
  subset(res3, P<0.05)
```

    ##             Variable        Term        Beta StandardError          t
    ##   1: ENSG00000001461 cellN061011 -0.46859875    0.10526111  -4.451775
    ##   2: ENSG00000001461 cellN080611 -0.84020922    0.10526111  -7.982143
    ##   3: ENSG00000001461  cellN61311 -0.87778101    0.10526111  -8.339082
    ##   4: ENSG00000001561 cellN080611 -1.71802758    0.13649920 -12.586357
    ##   5: ENSG00000001561  cellN61311 -1.05328889    0.13649920  -7.716448
    ##  ---                                                                 
    ## 519: ENSG00000092108 cellN061011 -0.12721659    0.01564082  -8.133625
    ## 520: ENSG00000092108  cellN61311 -0.12451203    0.01564082  -7.960708
    ## 521: ENSG00000092148 cellN080611 -0.34988071    0.10313461  -3.392467
    ## 522: ENSG00000092200 cellN080611  0.05906656    0.01521063   3.883241
    ## 523: ENSG00000092208 cellN080611 -0.28587683    0.08506716  -3.360602
    ##                 P        OR   ORlower   ORupper
    ##   1: 0.0112313246 0.6258787 0.5092039 0.7692873
    ##   2: 0.0013351958 0.4316202 0.3511586 0.5305181
    ##   3: 0.0011301853 0.4157043 0.3382098 0.5109554
    ##   4: 0.0002293465 0.1794197 0.1373036 0.2344544
    ##   5: 0.0015182960 0.3487887 0.2669157 0.4557753
    ##  ---                                           
    ## 519: 0.0012429963 0.8805429 0.8539591 0.9079544
    ## 520: 0.0013489163 0.8829276 0.8562718 0.9104133
    ## 521: 0.0274674209 0.7047722 0.5757851 0.8626549
    ## 522: 0.0177922771 1.0608458 1.0296864 1.0929482
    ## 523: 0.0282890537 0.7513552 0.6359690 0.8876762

Perform the most basic negative binomial logistic regression analysis
---------------------------------------------------------------------

Here, we will utilise normalised, unlogged counts from <i>DESeq2</i>. Unlogged counts in RNA-seq naturally follow a negative binomial / Poisson-like distribution. <i>glm.nbParallel</i> will be used.

``` r
    nbcounts <- round(counts(dds, normalized = TRUE), 0)
    nbdata <- data.frame(colData(airway), t(nbcounts))

    res4 <- RegParallel(
      data = nbdata[ ,1:3000],
      formula = '[*] ~ dex',
      FUN = function(formula, data)
        glm.nb(formula = formula,
          data = data),
      FUNtype = 'glm.nb',
      variables = colnames(nbdata)[10:3000],
    p.adjust = "fdr")

    res4[order(res4$Theta, decreasing = TRUE),]
```

    ##              Variable   Term         Beta StandardError          Z
    ##    1: ENSG00000102226 dextrt -0.139286172    0.01862107 -7.4800288
    ##    2: ENSG00000102910 dextrt -0.030278805    0.01582026 -1.9139254
    ##    3: ENSG00000063601 dextrt -0.002822867    0.02656550 -0.1062607
    ##    4: ENSG00000083642 dextrt -0.128217080    0.02412001 -5.3157976
    ##    5: ENSG00000023041 dextrt  0.134325359    0.02728332  4.9233512
    ##   ---                                                             
    ## 2817: ENSG00000029559 dextrt -1.386294361    1.73450917 -0.7992430
    ## 2818: ENSG00000006128 dextrt  1.386294361    1.73450917  0.7992430
    ## 2819: ENSG00000101197 dextrt -1.386294361    1.73450917 -0.7992430
    ## 2820: ENSG00000102109 dextrt -0.318453731    1.39587712 -0.2281388
    ## 2821: ENSG00000069122 dextrt  0.552068582    1.61318296  0.3422232
    ##                  P        Theta      SEtheta  2xLogLik Dispersion
    ##    1: 7.430632e-14 1.987085e+08 1.508873e+10 -73.91315          1
    ##    2: 5.562968e-02 1.124953e+08 6.192688e+09 -78.06984          1
    ##    3: 9.153755e-01 6.328822e+07 4.095363e+09 -68.80847          1
    ##    4: 1.061911e-07 3.111265e+07 1.578546e+09 -72.68055          1
    ##    5: 8.507456e-07 3.003015e+07 1.594172e+09 -70.09368          1
    ##   ---                                                            
    ## 2817: 4.241495e-01 2.843297e-01 3.514737e-01 -15.24150          1
    ## 2818: 4.241495e-01 2.843297e-01 3.514737e-01 -15.24150          1
    ## 2819: 4.241495e-01 2.843297e-01 3.514737e-01 -15.24150          1
    ## 2820: 8.195383e-01 2.664530e-01 1.500629e-01 -41.63690          1
    ## 2821: 7.321830e-01 1.984580e-01 1.233914e-01 -37.96063          1
    ##              OR    ORlower     ORupper     P.adjust
    ##    1: 0.8699790 0.83880014   0.9023169 2.624401e-12
    ##    2: 0.9701750 0.94055425   1.0007286 1.401173e-01
    ##    3: 0.9971811 0.94658900   1.0504772 9.941050e-01
    ##    4: 0.8796624 0.83904459   0.9222465 1.361660e-06
    ##    5: 1.1437649 1.08420938   1.2065918 8.333172e-06
    ##   ---                                              
    ## 2817: 0.2500000 0.00834686   7.4878457 6.161820e-01
    ## 2818: 4.0000000 0.13354976 119.8055313 6.161820e-01
    ## 2819: 0.2500000 0.00834686   7.4878457 6.161820e-01
    ## 2820: 0.7272727 0.04715465  11.2168280 9.303492e-01
    ## 2821: 1.7368421 0.07355573  41.0113595 8.703426e-01

Survival analysis via Cox Proportional Hazards regression
---------------------------------------------------------

For this example, we will load breast cancer gene expression data with recurrence free survival (RFS) from [Gene Expression Profiling in Breast Cancer: Understanding the Molecular Basis of Histologic Grade To Improve Prognosis](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE2990). Specifically, we will encode each gene's expression into Low|Mid|High based on Z-scores and compare these against RFS while adjusting for tumour grade in a Cox Proportional Hazards model.

First, let's read in and prepare the data

``` r
  library(Biobase)

  library(GEOquery)

  # load series and platform data from GEO
  gset <- getGEO('GSE2990', GSEMatrix =TRUE, getGPL=FALSE)

  x <- exprs(gset[[1]])

  # remove Affymetrix control probes
  x <- x[-grep('^AFFX', rownames(x)),]

  # transform the expression data to Z scores
  x <- t(scale(t(x)))

  # extract information of interest from the phenotype data (pdata)
  idx <- which(colnames(pData(gset[[1]])) %in%
    c('age:ch1', 'distant rfs:ch1', 'er:ch1',
      'ggi:ch1', 'grade:ch1', 'node:ch1',
      'size:ch1', 'time rfs:ch1'))

  metadata <- data.frame(pData(gset[[1]])[,idx],
    row.names = rownames(pData(gset[[1]])))

  # remove samples from the pdata that have any NA value
  discard <- apply(metadata, 1, function(x) any(is.na(x)))

  metadata <- metadata[!discard,]

  # filter the Z-scores expression data to match the samples in our pdata
  x <- x[,which(colnames(x) %in% rownames(metadata))]

  # check that sample names match exactly between pdata and Z-scores 
  all((colnames(x) == rownames(metadata)) == TRUE)
```

    ## [1] TRUE

``` r
  # create a merged pdata and Z-scores object
  coxdata <- data.frame(metadata, t(x))

  # tidy column names
  colnames(coxdata)[1:8] <- c('Age', 'Distant.RFS', 'ER',
    'GGI', 'Grade', 'Node',
    'Size', 'Time.RFS')

  # prepare certain phenotypes
  coxdata$Age <- as.numeric(gsub('^KJ', '', coxdata$Age))
  coxdata$Distant.RFS <- as.numeric(coxdata$Distant.RFS)
  coxdata$ER <- factor(coxdata$ER, levels = c(0, 1))
  coxdata$Grade <- factor(coxdata$Grade, levels = c(1, 2, 3))
  coxdata$Time.RFS <- as.numeric(gsub('^KJX|^KJ', '', coxdata$Time.RFS))
```

With the data prepared, we can now apply a Cox Proportional Hazards model independently for each probe in the dataset against RFS.

In this we also increase the default blocksize to 2000 in order to speed up the analysis.

``` r
  library(survival)

  res5 <- RegParallel(
    data = coxdata,
    formula = 'Surv(Time.RFS, Distant.RFS) ~ [*]',
    FUN = function(formula, data)
      coxph(formula = formula,
        data = data,
        ties = 'breslow',
        singular.ok = TRUE),
    FUNtype = 'coxph',
    variables = colnames(coxdata)[9:ncol(coxdata)],
    blocksize = 2000,
    p.adjust = "BH")

  res5 <- res5[!is.na(res5$P),]
  res5
```

    ##           Variable        Term          Beta StandardError             Z
    ##     1:  X1007_s_at  X1007_s_at  0.3780639987     0.3535022  1.0694811914
    ##     2:    X1053_at    X1053_at  0.1177398813     0.2275041  0.5175285346
    ##     3:     X117_at     X117_at  0.6265036787     0.6763106  0.9263549892
    ##     4:     X121_at     X121_at -0.6138126274     0.6166626 -0.9953783151
    ##     5:  X1255_g_at  X1255_g_at -0.2043297829     0.3983930 -0.5128849375
    ##    ---                                                                  
    ## 22211:   X91703_at   X91703_at -0.4124539527     0.4883759 -0.8445419981
    ## 22212: X91816_f_at X91816_f_at  0.0482030943     0.3899180  0.1236236554
    ## 22213:   X91826_at   X91826_at  0.0546751431     0.3319572  0.1647053850
    ## 22214:   X91920_at   X91920_at -0.6452125945     0.8534623 -0.7559942684
    ## 22215:   X91952_at   X91952_at -0.0001396044     0.7377681 -0.0001892254
    ##                P       LRT      Wald   LogRank        HR    HRlower
    ##     1: 0.2848529 0.2826716 0.2848529 0.2848400 1.4594563 0.72994385
    ##     2: 0.6047873 0.6085603 0.6047873 0.6046839 1.1249515 0.72024775
    ##     3: 0.3542615 0.3652989 0.3542615 0.3541855 1.8710573 0.49706191
    ##     4: 0.3195523 0.3188303 0.3195523 0.3186921 0.5412832 0.16162940
    ##     5: 0.6080318 0.6084157 0.6080318 0.6077573 0.8151935 0.37337733
    ##    ---                                                             
    ## 22211: 0.3983666 0.3949865 0.3983666 0.3981244 0.6620237 0.25419512
    ## 22212: 0.9016133 0.9015048 0.9016133 0.9016144 1.0493838 0.48869230
    ## 22213: 0.8691759 0.8691994 0.8691759 0.8691733 1.0561974 0.55103934
    ## 22214: 0.4496526 0.4478541 0.4496526 0.4498007 0.5245510 0.09847349
    ## 22215: 0.9998490 0.9998490 0.9998490 0.9998490 0.9998604 0.23547784
    ##         HRupper  P.adjust LRT.adjust Wald.adjust LogRank.adjust
    ##     1: 2.918050 0.9999969  0.9999969   0.9999969      0.9999969
    ##     2: 1.757056 0.9999969  0.9999969   0.9999969      0.9999969
    ##     3: 7.043097 0.9999969  0.9999969   0.9999969      0.9999969
    ##     4: 1.812712 0.9999969  0.9999969   0.9999969      0.9999969
    ##     5: 1.779809 0.9999969  0.9999969   0.9999969      0.9999969
    ##    ---                                                         
    ## 22211: 1.724169 0.9999969  0.9999969   0.9999969      0.9999969
    ## 22212: 2.253373 0.9999969  0.9999969   0.9999969      0.9999969
    ## 22213: 2.024453 0.9999969  0.9999969   0.9999969      0.9999969
    ## 22214: 2.794191 0.9999969  0.9999969   0.9999969      0.9999969
    ## 22215: 4.245498 0.9999969  0.9999969   0.9999969      0.9999969

We now take the top probes from the model by Log Rank p-value and use <i>biomaRt</i> to look up the corresponding gene symbols.

``` r
  res5 <- res5[order(res5$LogRank, decreasing = FALSE),]
  final <- subset(res5, LogRank < 0.01)
  probes <- gsub('^X', '', final$Variable)

  library(biomaRt)
  mart <- useMart('ENSEMBL_MART_ENSEMBL')
  mart <- useDataset("hsapiens_gene_ensembl", mart)
  annotLookup <- getBM(mart = mart,
    attributes = c('affy_hg_u133a',
      'ensembl_gene_id',
      'gene_biotype',
      'external_gene_name'),
    filter = 'affy_hg_u133a',
    values = probes,
    uniqueRows = TRUE)
```

Two of the top hits include <i>CXCL12</i> and <i>MMP10</i>. High expression of <i>CXCL12</i> was previously associated with good progression free and overall survival in breast cancer in (doi: 10.1016/j.cca.2018.05.041.)\[<https://www.ncbi.nlm.nih.gov/pubmed/29800557>\] , whilst high expression of <i>MMP10</i> was associated with poor prognosis in colon cancer in (doi: 10.1186/s12885-016-2515-7)\[<https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4950722/>\].

We can further explore the role of these genes to RFS by dividing their gene expression Z-scores into tertiles for low, mid, and high expression:

``` r
  # extract RFS and probe data for downstream analysis
  survplotdata <- coxdata[,c('Time.RFS', 'Distant.RFS',
    'X203666_at', 'X205680_at')]
  colnames(survplotdata) <- c('Time.RFS', 'Distant.RFS',
    'CXCL12', 'MMP10')

  # set Z-scale cut-offs for high and low expression
  highExpr <- 1.0
  lowExpr <- 1.0

  # encode the expression for CXCL12 and MMP10 as low, mid, and high
  survplotdata$CXCL12 <- ifelse(survplotdata$CXCL12 >= highExpr, 'High',
    ifelse(x <= lowExpr, 'Low', 'Mid'))
  survplotdata$MMP10 <- ifelse(survplotdata$MMP10 >= highExpr, 'High',
    ifelse(x <= lowExpr, 'Low', 'Mid'))

  # relevel the factors to have mid as the reference level
  survplotdata$CXCL12 <- factor(survplotdata$CXCL12,
    levels = c('Mid', 'Low', 'High'))
  survplotdata$MMP10 <- factor(survplotdata$MMP10,
    levels = c('Mid', 'Low', 'High'))
```

Plot the survival curves and place Log Rank p-value in the plots:

``` r
  library(survminer)

  ggsurvplot(survfit(Surv(Time.RFS, Distant.RFS) ~ CXCL12,
    data = survplotdata),
    data = survplotdata,
    risk.table = TRUE,
    pval = TRUE,
    break.time.by = 500,
    ggtheme = theme_minimal(),
    risk.table.y.text.col = TRUE,
    risk.table.y.text = FALSE)
```

![Survival analysis via Cox Proportional Hazards regression.](README_files/figure-markdown_github/coxphParallel5-1.png)

``` r
  ggsurvplot(survfit(Surv(Time.RFS, Distant.RFS) ~ MMP10,
    data = survplotdata),
    data = survplotdata,
    risk.table = TRUE,
    pval = TRUE,
    break.time.by = 500,
    ggtheme = theme_minimal(),
    risk.table.y.text.col = TRUE,
    risk.table.y.text = FALSE)
```

![Survival analysis via Cox Proportional Hazards regression.](README_files/figure-markdown_github/coxphParallel5-2.png)

Perform a conditional logistic regression
-----------------------------------------

In this example, we will re-use the Cox data for the purpose of performing conditional logistic regression with tumour grade as our grouping / matching factor. For this example, we will use ER status as the dependent variable and also adjust for age.

``` r
  x <- exprs(gset[[1]])
  x <- x[-grep('^AFFX', rownames(x)),]
  x <- scale(x)
  x <- x[,which(colnames(x) %in% rownames(metadata))]

  coxdata <- data.frame(metadata, t(x))

  colnames(coxdata)[1:8] <- c('Age', 'Distant.RFS', 'ER',
    'GGI', 'Grade', 'Node',
    'Size', 'Time.RFS')

  coxdata$Age <- as.numeric(gsub('^KJ', '', coxdata$Age))
  coxdata$Grade <- factor(coxdata$Grade, levels = c(1, 2, 3))
  coxdata$ER <- as.numeric(coxdata$ER)
  coxdata <- coxdata[!is.na(coxdata$ER),]

  res6 <- RegParallel(
    data = coxdata,
    formula = 'ER ~ [*] + Age + strata(Grade)',
    FUN = function(formula, data)
      clogit(formula = formula,
        data = data,
        method = 'breslow'),
    FUNtype = 'clogit',
    variables = colnames(coxdata)[9:ncol(coxdata)],
    blocksize = 2000)

  subset(res6, P < 0.01)
```

    ##        Variable         Term       Beta StandardError         Z
    ## 1:   X204667_at   X204667_at  0.9940504     0.3628087  2.739875
    ## 2:   X205225_at   X205225_at  0.4444556     0.1633857  2.720285
    ## 3: X207813_s_at X207813_s_at  0.8218501     0.3050777  2.693904
    ## 4:   X212108_at   X212108_at  1.9610211     0.7607284  2.577820
    ## 5: X219497_s_at X219497_s_at -1.0249671     0.3541401 -2.894242
    ##              P         LRT       Wald    LogRank        HR   HRlower
    ## 1: 0.006146252 0.006808415 0.02212540 0.02104525 2.7021573 1.3270501
    ## 2: 0.006522559 0.010783544 0.01941078 0.01701248 1.5596409 1.1322713
    ## 3: 0.007062046 0.037459927 0.02449358 0.02424809 2.2747043 1.2509569
    ## 4: 0.009942574 0.033447973 0.03356050 0.03384960 7.1065797 1.6000274
    ## 5: 0.003800756 0.005153233 0.01387183 0.01183245 0.3588083 0.1792329
    ##      HRupper
    ## 1:  5.502169
    ## 2:  2.148319
    ## 3:  4.136257
    ## 4: 31.564132
    ## 5:  0.718302

``` r
  getBM(mart = mart,
    attributes = c('affy_hg_u133a',
      'ensembl_gene_id',
      'gene_biotype',
      'external_gene_name'),
    filter = 'affy_hg_u133a',
    values = c('204667_at',
      '205225_at',
      '207813_s_at',
      '212108_at',
      '219497_s_at'),
    uniqueRows=TRUE)
```

    ##   affy_hg_u133a ensembl_gene_id   gene_biotype external_gene_name
    ## 1   207813_s_at ENSG00000161513 protein_coding               FDXR
    ## 2     205225_at ENSG00000091831 protein_coding               ESR1
    ## 3   219497_s_at ENSG00000119866 protein_coding             BCL11A
    ## 4     212108_at ENSG00000113194 protein_coding               FAF2

Oestrogen receptor (<i>ESR1</i>) comes out top - makes sense! Also, although 204667\_at is not listed in <i>biomaRt</i>, it overlaps an exon of <i>FOXA1</i>, which also makes sense in relation to oestrogen signalling.

Advanced features
=================

Advanced features include the ability to modify block size, choose different numbers of cores, enable 'nested' parallel processing, modify limits for confidence intervals, and exclude certain model terms from output.

Speed up processing
-------------------

First create some test data for the purpose of benchmarking:

``` r
  options(scipen=10)
  options(digits=6)

  # create a data-matrix of 20 x 60000 (rows x cols) random numbers
  col <- 60000
  row <- 20
  mat <- matrix(
    rexp(col*row, rate = .1),
    ncol = col)

  # add fake gene and sample names
  colnames(mat) <- paste0('gene', 1:ncol(mat))

  rownames(mat) <- paste0('sample', 1:nrow(mat))

  # add some fake metadata
  modelling <- data.frame(
    cell = rep(c('B', 'T'), nrow(mat) / 2),
    group = c(rep(c('treatment'), nrow(mat) / 2), rep(c('control'), nrow(mat) / 2)),
    dosage = t(data.frame(matrix(rexp(row, rate = 1), ncol = row))),
    mat,
    row.names = rownames(mat))
```

### ~2000 tests; blocksize, 500; cores, 2; nestedParallel, TRUE

With 2 cores instead of the default of 3, coupled with nestedParallel being enabled, a total of 2 x 2 = 4 threads will be used.

``` r
  df <- modelling[ ,1:2000]
  variables <- colnames(df)[4:ncol(df)]

  ptm <- proc.time()

  res <- RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 500,
    cores = 2,
    nestedParallel = TRUE,
    p.adjust = "BY")

  proc.time() - ptm
```

    ##    user  system elapsed 
    ##  14.932   5.004  15.338

### ~2000 tests; blocksize, 500; cores, 2; nestedParallel, FALSE

``` r
  df <- modelling[ ,1:2000]
  variables <- colnames(df)[4:ncol(df)]

  ptm <- proc.time()

  res <- RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 500,
    cores = 2,
    nestedParallel = FALSE,
    p.adjust = "BY")

  proc.time() - ptm
```

    ##    user  system elapsed 
    ##  13.088   1.068  14.148

Focusing on the elapsed time (as system time only reports time from the last core that finished), we can see that nested processing has negligible improvement or may actually be slower under certain conditions when tested over a small number of variables. This is likely due to the system being slowed by simply managing the larger number of threads. Nested processing's benefits can only be gained when processing a large number of variables:

### ~40000 tests; blocksize, 2000; cores, 2; nestedParallel, TRUE

``` r
  df <- modelling[ ,1:40000]
  variables <- colnames(df)[4:ncol(df)]

  system.time(RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 2000,
    cores = 2,
    nestedParallel = TRUE))
```

    ##    user  system elapsed 
    ## 770.252  86.556 275.310

### ~40000 tests; blocksize, 2000; cores, 2; nestedParallel, FALSE

``` r
  df <- modelling[,1:40000]
  variables <- colnames(df)[4:ncol(df)]

  system.time(RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 2000,
    cores = 2,
    nestedParallel = FALSE))
```

    ##    user  system elapsed 
    ## 330.520   1.564 336.270

Performance is system-dependent and even increasing cores may not result in huge gains in time. Performance is a trade-off between cores, forked threads, blocksize, and the number of terms in each model.

### ~40000 tests; blocksize, 5000; cores, 3; nestedParallel, TRUE

In this example, we choose a large blocksize and 3 cores. With nestedParallel enabled, this translates to 9 simultaneous threads.

``` r
  df <- modelling[,1:40000]
  variables <- colnames(df)[4:ncol(df)]

  system.time(RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 5000,
    cores = 3,
    nestedParallel = TRUE))
```

    ##     user   system  elapsed 
    ## 1256.924   66.492  451.724

Modify confidence intervals
---------------------------

``` r
  df <- modelling[ ,1:500]
  variables <- colnames(df)[4:ncol(df)]

  # 99% confidfence intervals
  RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 150,
    cores = 3,
    nestedParallel = TRUE,
    conflevel = 99)
```

    ##       Variable         Term       Beta StandardError         Z        P
    ##    1:    gene1        gene1 -0.0145456     0.0354115 -0.410761 0.681248
    ##    2:    gene1 cellB:dosage -0.1506926     0.2663616 -0.565745 0.571567
    ##    3:    gene1 cellT:dosage -0.1885051     0.4355192 -0.432828 0.665139
    ##    4:    gene2        gene2 -0.0286017     0.0597072 -0.479033 0.631915
    ##    5:    gene2 cellB:dosage -0.1675974     0.2684081 -0.624413 0.532357
    ##   ---                                                                  
    ## 1487:  gene496 cellB:dosage  0.0549183     0.2807548  0.195610 0.844916
    ## 1488:  gene496 cellT:dosage  0.2471517     0.4966379  0.497650 0.618731
    ## 1489:  gene497      gene497 -0.0148307     0.0620004 -0.239203 0.810948
    ## 1490:  gene497 cellB:dosage -0.1598489     0.2690730 -0.594073 0.552464
    ## 1491:  gene497 cellT:dosage -0.1259319     0.5192998 -0.242503 0.808390
    ##             OR  ORlower ORupper
    ##    1: 0.985560 0.899641 1.07968
    ##    2: 0.860112 0.433097 1.70815
    ##    3: 0.828196 0.269731 2.54293
    ##    4: 0.971803 0.833270 1.13337
    ##    5: 0.845694 0.423598 1.68839
    ##   ---                          
    ## 1487: 1.056454 0.512601 2.17732
    ## 1488: 1.280373 0.356257 4.60161
    ## 1489: 0.985279 0.839849 1.15589
    ## 1490: 0.852273 0.426162 1.70444
    ## 1491: 0.881675 0.231411 3.35918

``` r
  # 95% confidfence intervals (default)
  RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 150,
    cores = 3,
    nestedParallel = TRUE,
    conflevel = 95)
```

    ##       Variable         Term       Beta StandardError         Z        P
    ##    1:    gene1        gene1 -0.0145456     0.0354115 -0.410761 0.681248
    ##    2:    gene1 cellB:dosage -0.1506926     0.2663616 -0.565745 0.571567
    ##    3:    gene1 cellT:dosage -0.1885051     0.4355192 -0.432828 0.665139
    ##    4:    gene2        gene2 -0.0286017     0.0597072 -0.479033 0.631915
    ##    5:    gene2 cellB:dosage -0.1675974     0.2684081 -0.624413 0.532357
    ##   ---                                                                  
    ## 1487:  gene496 cellB:dosage  0.0549183     0.2807548  0.195610 0.844916
    ## 1488:  gene496 cellT:dosage  0.2471517     0.4966379  0.497650 0.618731
    ## 1489:  gene497      gene497 -0.0148307     0.0620004 -0.239203 0.810948
    ## 1490:  gene497 cellB:dosage -0.1598489     0.2690730 -0.594073 0.552464
    ## 1491:  gene497 cellT:dosage -0.1259319     0.5192998 -0.242503 0.808390
    ##             OR  ORlower ORupper
    ##    1: 0.985560 0.919476 1.05639
    ##    2: 0.860112 0.510302 1.44971
    ##    3: 0.828196 0.352711 1.94468
    ##    4: 0.971803 0.864481 1.09245
    ##    5: 0.845694 0.499740 1.43114
    ##   ---                          
    ## 1487: 1.056454 0.609357 1.83160
    ## 1488: 1.280373 0.483724 3.38903
    ## 1489: 0.985279 0.872538 1.11259
    ## 1490: 0.852273 0.502971 1.44416
    ## 1491: 0.881675 0.318625 2.43970

Remove some terms from output / include the intercept
-----------------------------------------------------

``` r
  # remove terms but keep Intercept
  RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 150,
    cores = 3,
    nestedParallel = TRUE,
    conflevel = 95,
    excludeTerms = c('cell', 'dosage'),
    excludeIntercept = FALSE)
```

    ##      Variable        Term       Beta StandardError          Z        P
    ##   1:    gene1 (Intercept)  0.4654455     0.7801895  0.5965801 0.550788
    ##   2:    gene1       gene1 -0.0145456     0.0354115 -0.4107608 0.681248
    ##   3:    gene2 (Intercept)  0.5430816     0.8422252  0.6448175 0.519045
    ##   4:    gene2       gene2 -0.0286017     0.0597072 -0.4790331 0.631915
    ##   5:    gene3 (Intercept)  0.0118877     0.8409042  0.0141368 0.988721
    ##  ---                                                                  
    ## 990:  gene495     gene495  0.0260860     0.0397872  0.6556379 0.512057
    ## 991:  gene496 (Intercept) -1.5019811     1.0789375 -1.3920928 0.163894
    ## 992:  gene496     gene496  0.2101329     0.1334633  1.5744619 0.115381
    ## 993:  gene497 (Intercept)  0.3926244     0.7860373  0.4994984 0.617428
    ## 994:  gene497     gene497 -0.0148307     0.0620004 -0.2392028 0.810948
    ##            OR   ORlower ORupper
    ##   1: 1.592724 0.3451771 7.34918
    ##   2: 0.985560 0.9194765 1.05639
    ##   3: 1.721303 0.3303345 8.96935
    ##   4: 0.971803 0.8644812 1.09245
    ##   5: 1.011959 0.1947080 5.25947
    ##  ---                           
    ## 990: 1.026429 0.9494281 1.10968
    ## 991: 0.222689 0.0268723 1.84540
    ## 992: 1.233842 0.9498508 1.60274
    ## 993: 1.480862 0.3172769 6.91179
    ## 994: 0.985279 0.8725379 1.11259

``` r
  # remove everything but the variable being tested
  RegParallel(
    data = df,
    formula = 'factor(group) ~ [*] + (cell:dosage) ^ 2',
    FUN = function(formula, data)
      glm(formula = formula,
        data = data,
        family = binomial(link = 'logit'),
        method = 'glm.fit'),
    FUNtype = 'glm',
    variables = variables,
    blocksize = 150,
    cores = 3,
    nestedParallel = TRUE,
    conflevel = 95,
    excludeTerms = c('cell', 'dosage'),
    excludeIntercept = TRUE)
```

    ##      Variable    Term       Beta StandardError         Z         P
    ##   1:    gene1   gene1 -0.0145456     0.0354115 -0.410761 0.6812479
    ##   2:    gene2   gene2 -0.0286017     0.0597072 -0.479033 0.6319151
    ##   3:    gene3   gene3  0.0310578     0.0632369  0.491134 0.6233318
    ##   4:    gene4   gene4  0.0433150     0.0522265  0.829369 0.4068958
    ##   5:    gene5   gene5 -0.0350645     0.0740899 -0.473269 0.6360210
    ##  ---                                                              
    ## 493:  gene493 gene493  0.0750731     0.0622396  1.206195 0.2277425
    ## 494:  gene494 gene494 -0.1210313     0.0706871 -1.712212 0.0868577
    ## 495:  gene495 gene495  0.0260860     0.0397872  0.655638 0.5120571
    ## 496:  gene496 gene496  0.2101329     0.1334633  1.574462 0.1153807
    ## 497:  gene497 gene497 -0.0148307     0.0620004 -0.239203 0.8109483
    ##            OR  ORlower ORupper
    ##   1: 0.985560 0.919476 1.05639
    ##   2: 0.971803 0.864481 1.09245
    ##   3: 1.031545 0.911299 1.16766
    ##   4: 1.044267 0.942662 1.15682
    ##   5: 0.965543 0.835038 1.11644
    ##  ---                          
    ## 493: 1.077963 0.954169 1.21782
    ## 494: 0.886006 0.771379 1.01767
    ## 495: 1.026429 0.949428 1.10968
    ## 496: 1.233842 0.949851 1.60274
    ## 497: 0.985279 0.872538 1.11259

Acknowledgments
===============

*RegParallel* would not exist were it not for initial contributions from:

[Jessica Lasky-Su](https://connects.catalyst.harvard.edu/Profiles/display/Person/79780), [Myles Lewis](https://www.qmul.ac.uk/whri/people/academic-staff/items/lewismyles.html), [Michael Barnes](https://www.qmul.ac.uk/whri/people/academic-staff/items/barnesmichael.html)

Thanks also to Horacio Montenegro and GenoMax for testing.

Session info
============

``` r
sessionInfo()
```

    ## R version 3.5.2 (2018-12-20)
    ## Platform: x86_64-pc-linux-gnu (64-bit)
    ## Running under: Ubuntu 16.04.5 LTS
    ## 
    ## Matrix products: default
    ## BLAS: /usr/lib/atlas-base/atlas/libblas.so.3.0
    ## LAPACK: /usr/lib/atlas-base/atlas/liblapack.so.3.0
    ## 
    ## locale:
    ##  [1] LC_CTYPE=pt_BR.UTF-8       LC_NUMERIC=C              
    ##  [3] LC_TIME=en_GB.UTF-8        LC_COLLATE=pt_BR.UTF-8    
    ##  [5] LC_MONETARY=en_GB.UTF-8    LC_MESSAGES=pt_BR.UTF-8   
    ##  [7] LC_PAPER=en_GB.UTF-8       LC_NAME=C                 
    ##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
    ## [11] LC_MEASUREMENT=en_GB.UTF-8 LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] stats4    parallel  stats     graphics  grDevices utils     datasets 
    ## [8] methods   base     
    ## 
    ## other attached packages:
    ##  [1] survminer_0.4.3             ggpubr_0.2                 
    ##  [3] ggplot2_3.1.0               biomaRt_2.38.0             
    ##  [5] bindrcpp_0.2.2              GEOquery_2.50.5            
    ##  [7] DESeq2_1.22.2               magrittr_1.5               
    ##  [9] airway_1.2.0                SummarizedExperiment_1.12.0
    ## [11] DelayedArray_0.8.0          BiocParallel_1.16.5        
    ## [13] matrixStats_0.54.0          Biobase_2.42.0             
    ## [15] GenomicRanges_1.34.0        GenomeInfoDb_1.18.1        
    ## [17] IRanges_2.16.0              S4Vectors_0.20.1           
    ## [19] BiocGenerics_0.28.0         RegParallel_1.3.0          
    ## [21] arm_1.10-1                  lme4_1.1-19                
    ## [23] Matrix_1.2-15               MASS_7.3-51.1              
    ## [25] survival_2.43-3             stringr_1.3.1              
    ## [27] data.table_1.12.0           doParallel_1.0.14          
    ## [29] iterators_1.0.10            foreach_1.4.4              
    ## [31] knitr_1.21                 
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] minqa_1.2.4            colorspace_1.4-0       htmlTable_1.13.1      
    ##  [4] XVector_0.22.0         base64enc_0.1-3        rstudioapi_0.9.0      
    ##  [7] bit64_0.9-7            AnnotationDbi_1.44.0   xml2_1.2.0            
    ## [10] codetools_0.2-16       splines_3.5.2          geneplotter_1.60.0    
    ## [13] Formula_1.2-3          nloptr_1.2.1           km.ci_0.5-2           
    ## [16] broom_0.5.1            annotate_1.60.0        cluster_2.0.7-1       
    ## [19] readr_1.3.1            compiler_3.5.2         httr_1.4.0            
    ## [22] backports_1.1.3        assertthat_0.2.0       lazyeval_0.2.1        
    ## [25] limma_3.38.3           acepack_1.4.1          htmltools_0.3.6       
    ## [28] prettyunits_1.0.2      tools_3.5.2            coda_0.19-2           
    ## [31] gtable_0.2.0           glue_1.3.0             GenomeInfoDbData_1.2.0
    ## [34] dplyr_0.7.8            Rcpp_1.0.0             nlme_3.1-137          
    ## [37] xfun_0.4               XML_3.98-1.16          zoo_1.8-4             
    ## [40] zlibbioc_1.28.0        scales_1.0.0           hms_0.4.2             
    ## [43] RColorBrewer_1.1-2     yaml_2.2.0             curl_3.3              
    ## [46] memoise_1.1.0          gridExtra_2.3          KMsurv_0.1-5          
    ## [49] rpart_4.1-13           latticeExtra_0.6-28    stringi_1.2.4         
    ## [52] RSQLite_2.1.1          highr_0.7              genefilter_1.64.0     
    ## [55] checkmate_1.9.1        rlang_0.3.1            pkgconfig_2.0.2       
    ## [58] bitops_1.0-6           evaluate_0.12          lattice_0.20-38       
    ## [61] purrr_0.2.5            bindr_0.1.1            labeling_0.3          
    ## [64] htmlwidgets_1.3        cmprsk_2.2-7           bit_1.1-14            
    ## [67] tidyselect_0.2.5       plyr_1.8.4             R6_2.3.0              
    ## [70] generics_0.0.2         Hmisc_4.1-1            DBI_1.0.0             
    ## [73] pillar_1.3.1           foreign_0.8-71         withr_2.1.2           
    ## [76] abind_1.4-5            RCurl_1.95-4.11        nnet_7.3-12           
    ## [79] tibble_2.0.1           crayon_1.3.4           survMisc_0.5.5        
    ## [82] rmarkdown_1.11.3       progress_1.2.0         locfit_1.5-9.1        
    ## [85] grid_3.5.2             blob_1.1.1             digest_0.6.18         
    ## [88] xtable_1.8-3           tidyr_0.8.2            munsell_0.5.0

References
==========

Blighe (2018)

Blighe, Kevin. 2018. “RegParallel: Standard regression functions in R enabled for parallel processing over large data-frames.” <https://github.com/kevinblighe>.
