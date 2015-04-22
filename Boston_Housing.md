# UCI Machine Learning Repository: Boston Housing: MEDV regression
bdanalytics  

**  **    
**Date: (Wed) Apr 22, 2015**    

# Introduction:  

Data: 
Source: 
    Training:   https://courses.edx.org/c4x/MITx/15.071x_2/asset/boston.csv  
    New:        <newdt_url>  
Time period: 



# Synopsis:

Based on analysis utilizing <> techniques, <conclusion heading>:  

### ![](<filename>.png)

## Potential next steps include:
- Organization:
    - Categorize by chunk
    - Priority criteria:
        0. Ease of change
        1. Impacts report
        2. Cleans innards
        3. Bug report
        
- manage.missing.data chunk:
    - cleaner way to manage re-splitting of training vs. new entity

- fit.models chunk:
    - Prediction accuracy scatter graph:
    -   Add tiles (raw vs. PCA)
    -   Use shiny for drop-down of "important" features
    -   Use plot.ly for interactive plots ?
    
    - Change .fit suffix of model metrics to .mdl if it's data independent (e.g. AIC, Adj.R.Squared - is it truly data independent ?, etc.)
    - move model_type parameter to myfit_mdl before indep_vars_vctr (keep all model_* together)
    - create a custom model for rpart that has minbucket as a tuning parameter
    - varImp for randomForest crashes in caret version:6.0.41 -> submit bug report

- Probability handling for multinomials vs. desired binomial outcome
-   ROCR currently supports only evaluation of binary classification tasks (version 1.0.7)
-   extensions toward multiclass classification are scheduled for the next release

- Skip trControl.method="cv" for dummy classifier ?
- Add custom model to caret for a dummy (baseline) classifier (binomial & multinomial) that generates proba/outcomes which mimics the freq distribution of glb_rsp_var values; Right now glb_dmy_glm_mdl always generates most frequent outcome in training data
- glm_dmy_mdl should use the same method as glm_sel_mdl until custom dummy classifer is implemented

- Compare glb_sel_mdl vs. glb_fin_mdl:
    - varImp
    - Prediction differences (shd be minimal ?)

- Move glb_analytics_diag_plots to mydsutils.R: (+) Easier to debug (-) Too many glb vars used
- Add print(ggplot.petrinet(glb_analytics_pn) + coord_flip()) at the end of every major chunk
- Parameterize glb_analytics_pn
- Move glb_impute_missing_data to mydsutils.R: (-) Too many glb vars used; glb_<>_df reassigned
- Replicate myfit_mdl_classification features in myfit_mdl_regression
- Do non-glm methods handle interaction terms ?
- f-score computation for classifiers should be summation across outcomes (not just the desired one ?)
- Add accuracy computation to glb_dmy_mdl in predict.data.new chunk
- Why does splitting fit.data.training.all chunk into separate chunks add an overhead of ~30 secs ? It's not rbind b/c other chunks have lower elapsed time. Is it the number of plots ?
- Incorporate code chunks in print_sessionInfo
- Test against 
    - projects in github.com/bdanalytics
    - lectures in jhu-datascience track

# Analysis: 

```r
rm(list=ls())
set.seed(12345)
options(stringsAsFactors=FALSE)
source("~/Dropbox/datascience/R/mydsutils.R")
source("~/Dropbox/datascience/R/myplot.R")
source("~/Dropbox/datascience/R/mypetrinet.R")
# Gather all package requirements here
#suppressPackageStartupMessages(require())
#packageVersion("snow")

#require(sos); findFn("pinv", maxPages=2, sortby="MaxScore")

# Analysis control global variables
glb_trnng_url <- "https://courses.edx.org/c4x/MITx/15.071x_2/asset/boston.csv"
glb_newdt_url <- "<newdt_url>"
glb_is_separate_newent_dataset <- FALSE    # or TRUE
glb_split_entity_newent_datasets <- TRUE   # or FALSE
glb_split_newdata_method <- "sample"          # "condition" or "sample"
glb_split_newdata_condition <- "<col_name> <condition_operator> <value>"    # or NULL
glb_split_newdata_size_ratio <- 0.3               # > 0 & < 1
glb_split_sample.seed <- 123               # or any integer
glb_max_obs <- NULL # or any integer

glb_is_regression <- TRUE; glb_is_classification <- FALSE

glb_rsp_var_raw <- "MEDV"

# for classification, the response variable has to be a factor
glb_rsp_var <- glb_rsp_var_raw # or "MEDV.fctr"

# if the response factor is based on numbers e.g (0/1 vs. "A"/"B"), 
#   caret predict(..., type="prob") crashes
glb_map_rsp_raw_to_var <- NULL # or function(raw) {
    #relevel(factor(ifelse(raw == 1, "R", "A")), as.factor(c("R", "A")), ref="A")
    #as.factor(paste0("B", raw))
#}
#glb_map_rsp_raw_to_var(c(1, 2, 3, 4, 5))

glb_map_rsp_var_to_raw <- NULL # or function(var) {
    #as.numeric(var) - 1
    #as.numeric(var)
#}
#glb_map_rsp_var_to_raw(glb_map_rsp_raw_to_var(c(1, 2, 3, 4, 5)))

if ((glb_rsp_var != glb_rsp_var_raw) & is.null(glb_map_rsp_raw_to_var))
    stop("glb_map_rsp_raw_to_var function expected")

glb_rsp_var_out <- paste0(glb_rsp_var, ".predict.") # model_id is appended later
glb_id_vars <- c("TRACT")  # or NULL

# List transformed vars  
#   trying TOWN.fctr here to see if varImp(rf) still crashes
glb_exclude_vars_as_features <- c("TOWN.fctr") # or c(NULL)
# List feats that shd be excluded due to known causation by prediction variable
if (glb_rsp_var_raw != glb_rsp_var)
    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                            glb_rsp_var_raw)
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c(NULL)) # or c("<col_name>")
# List output vars (useful during testing in console)          
# glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
#                         grep(glb_rsp_var_out, names(glb_trnent_df), value=TRUE)) 

glb_impute_na_data <- FALSE            # or TRUE
glb_mice_complete.seed <- 144               # or any integer

# Regression
#   rpart:  .rnorm messes with the models badly
#           caret creates dummy vars for factor feats which messes up the tuning
#               - better to feed as.numeric(<feat>.fctr) to caret 

glb_models_method_vctr <- c("lm", "glm", "rpart", "rf")

# Classification
#   rpart:  .rnorm messes with the models badly
#           caret creates dummy vars for factor feats which messes up the tuning
#               - better to feed as.numeric(<feat>.fctr) to caret 

#glb_models_method_vctr <- c("glm", "rpart", "rf")   # Binomials
#glb_models_method_vctr <- c("rpart", "rf")          # Multinomials

glb_models_lst <- list(); glb_models_df <- data.frame()
# Baseline prediction model feature(s)
glb_Baseline_mdl_var <- NULL # or c("<col_name>")

glb_model_metric_terms <- NULL # or matrix(c(
#                               0,1,2,3,4,
#                               2,0,1,2,3,
#                               4,2,0,1,2,
#                               6,4,2,0,1,
#                               8,6,4,2,0
#                           ), byrow=TRUE, nrow=5)
glb_model_metric <- NULL # or "<metric_name>"
glb_model_metric_maximize <- NULL # or FALSE (TRUE is not the default for both classification & regression) 
glb_model_metric_smmry <- NULL # or function(data, lev=NULL, model=NULL) {
#     confusion_mtrx <- t(as.matrix(confusionMatrix(data$pred, data$obs)))
#     #print(confusion_mtrx)
#     #print(confusion_mtrx * glb_model_metric_terms)
#     metric <- sum(confusion_mtrx * glb_model_metric_terms) / nrow(data)
#     names(metric) <- glb_model_metric
#     return(metric)
# }

glb_tune_models_df <- 
   rbind(
    data.frame(parameter="cp", min=0.001, max=0.010, by=0.001),
                          #seq(from=0.000,  to=0.010, by=0.001)
    #data.frame(parameter="mtry", min=2, max=4, by=1),
    data.frame(parameter="dummy", min=2, max=4, by=1)
        ) 
# or NULL
glb_n_cv_folds <- 10 # or NULL

glb_clf_proba_threshold <- NULL # 0.5

# Model selection criteria
# Regression:
glb_model_evl_criteria <- c("min.RMSE.OOB", "max.R.sq.OOB", "max.Adj.R.sq.fit", "min.aic.fit")
# Classification:    Binomial: add AIC
# Classification: Multinomial:
# glb_model_sel_frmla <- formula(paste0("~ ", 
#     ifelse(!is.null(glb_model_metric), 
#         paste0(ifelse(!glb_model_metric_maximize, "+min.", "-max."), 
#                paste0(glb_model_metric, ".OOB")), 
#            ""), " -max.Accuracy.OOB -max.Kappa.OOB"))

glb_sel_mdl_id <- NULL # or "<model_id>.<model_method>"
glb_fin_mdl_id <- NULL # or glb_sel_mdl_id

# Depict process
glb_analytics_pn <- petrinet(name="glb_analytics_pn",
                        trans_df=data.frame(id=1:6,
    name=c("data.training.all","data.new",
           "model.selected","model.final",
           "data.training.all.prediction","data.new.prediction"),
    x=c(   -5,-5,-15,-25,-25,-35),
    y=c(   -5, 5,  0,  0, -5,  5)
                        ),
                        places_df=data.frame(id=1:4,
    name=c("bgn","fit.data.training.all","predict.data.new","end"),
    x=c(   -0,   -20,                    -30,               -40),
    y=c(    0,     0,                      0,                 0),
    M0=c(   3,     0,                      0,                 0)
                        ),
                        arcs_df=data.frame(
    begin=c("bgn","bgn","bgn",        
            "data.training.all","model.selected","fit.data.training.all",
            "fit.data.training.all","model.final",    
            "data.new","predict.data.new",
            "data.training.all.prediction","data.new.prediction"),
    end  =c("data.training.all","data.new","model.selected",
            "fit.data.training.all","fit.data.training.all","model.final",
            "data.training.all.prediction","predict.data.new",
            "predict.data.new","data.new.prediction",
            "end","end")
                        ))
#print(ggplot.petrinet(glb_analytics_pn))
print(ggplot.petrinet(glb_analytics_pn) + coord_flip())
```

```
## Loading required package: grid
```

![](Boston_Housing_files/figure-html/set_global_options-1.png) 

```r
glb_analytics_avl_objs <- NULL

glb_script_tm <- proc.time()
glb_script_df <- data.frame(chunk_label="import_data", 
                            chunk_step_major=1, chunk_step_minor=0,
                            elapsed=(proc.time() - glb_script_tm)["elapsed"])
print(tail(glb_script_df, 2))
```

```
##         chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed import_data                1                0   0.002
```

## Step `1`: import data

```r
glb_entity_df <- myimport_data(url=glb_trnng_url, 
    comment=ifelse(!glb_is_separate_newent_dataset, "glb_entity_df", "glb_trnent_df"), 
                                force_header=TRUE)
```

```
## [1] "Reading file ./data/boston.csv..."
## [1] "dimensions of data in ./data/boston.csv: 506 rows x 16 cols"
##         TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS   NOX    RM
## 1     Nahant  2011 -70.9550 42.2550 24.0 0.00632 18  2.31    0 0.538 6.575
## 2 Swampscott  2021 -70.9500 42.2875 21.6 0.02731  0  7.07    0 0.469 6.421
## 3 Swampscott  2022 -70.9360 42.2830 34.7 0.02729  0  7.07    0 0.469 7.185
## 4 Marblehead  2031 -70.9280 42.2930 33.4 0.03237  0  2.18    0 0.458 6.998
## 5 Marblehead  2032 -70.9220 42.2980 36.2 0.06905  0  2.18    0 0.458 7.147
## 6 Marblehead  2033 -70.9165 42.3040 28.7 0.02985  0  2.18    0 0.458 6.430
##    AGE    DIS RAD TAX PTRATIO
## 1 65.2 4.0900   1 296    15.3
## 2 78.9 4.9671   2 242    17.8
## 3 61.1 4.9671   2 242    17.8
## 4 45.8 6.0622   3 222    18.7
## 5 54.2 6.0622   3 222    18.7
## 6 58.7 6.0622   3 222    18.7
##                TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS
## 18             Lynn  2055 -70.9780 42.2850 17.5 0.78420  0  8.14    0
## 77           Woburn  3333 -71.0900 42.2835 20.0 0.10153  0 12.83    0
## 165       Cambridge  3543 -71.0918 42.2265 22.7 2.24236  0 19.58    0
## 258       Brookline  4001 -71.0679 42.2073 50.0 0.61154 20  3.97    0
## 367 Boston Back Bay   104 -71.0540 42.2052 21.9 3.69695  0 18.10    0
## 498          Revere  1705 -70.9947 42.2496 18.3 0.26838  0  9.69    0
##       NOX    RM  AGE    DIS RAD TAX PTRATIO
## 18  0.538 5.990 81.7 4.2579   4 307    21.0
## 77  0.437 6.279 74.5 4.0522   5 398    18.7
## 165 0.605 5.854 91.8 2.4220   5 403    14.7
## 258 0.647 8.704 86.9 1.8010   5 264    13.0
## 367 0.718 4.963 91.4 1.7523  24 666    20.2
## 498 0.585 5.794 70.6 2.8927   6 391    19.2
##         TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS   NOX    RM
## 501   Revere  1708 -70.9920 42.2380 16.8 0.22438  0  9.69    0 0.585 6.027
## 502 Winthrop  1801 -70.9860 42.2312 22.4 0.06263  0 11.93    0 0.573 6.593
## 503 Winthrop  1802 -70.9910 42.2275 20.6 0.04527  0 11.93    0 0.573 6.120
## 504 Winthrop  1803 -70.9948 42.2260 23.9 0.06076  0 11.93    0 0.573 6.976
## 505 Winthrop  1804 -70.9875 42.2240 22.0 0.10959  0 11.93    0 0.573 6.794
## 506 Winthrop  1805 -70.9825 42.2210 19.0 0.04741  0 11.93    0 0.573 6.030
##      AGE    DIS RAD TAX PTRATIO
## 501 79.7 2.4982   6 391    19.2
## 502 69.1 2.4786   1 273    21.0
## 503 76.7 2.2875   1 273    21.0
## 504 91.0 2.1675   1 273    21.0
## 505 89.3 2.3889   1 273    21.0
## 506 80.8 2.5050   1 273    21.0
## 'data.frame':	506 obs. of  16 variables:
##  $ TOWN   : chr  "Nahant" "Swampscott" "Swampscott" "Marblehead" ...
##  $ TRACT  : int  2011 2021 2022 2031 2032 2033 2041 2042 2043 2044 ...
##  $ LON    : num  -71 -71 -70.9 -70.9 -70.9 ...
##  $ LAT    : num  42.3 42.3 42.3 42.3 42.3 ...
##  $ MEDV   : num  24 21.6 34.7 33.4 36.2 28.7 22.9 22.1 16.5 18.9 ...
##  $ CRIM   : num  0.00632 0.02731 0.02729 0.03237 0.06905 ...
##  $ ZN     : num  18 0 0 0 0 0 12.5 12.5 12.5 12.5 ...
##  $ INDUS  : num  2.31 7.07 7.07 2.18 2.18 2.18 7.87 7.87 7.87 7.87 ...
##  $ CHAS   : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ NOX    : num  0.538 0.469 0.469 0.458 0.458 0.458 0.524 0.524 0.524 0.524 ...
##  $ RM     : num  6.58 6.42 7.18 7 7.15 ...
##  $ AGE    : num  65.2 78.9 61.1 45.8 54.2 58.7 66.6 96.1 100 85.9 ...
##  $ DIS    : num  4.09 4.97 4.97 6.06 6.06 ...
##  $ RAD    : int  1 2 2 3 3 3 5 5 5 5 ...
##  $ TAX    : int  296 242 242 222 222 222 311 311 311 311 ...
##  $ PTRATIO: num  15.3 17.8 17.8 18.7 18.7 18.7 15.2 15.2 15.2 15.2 ...
##  - attr(*, "comment")= chr "glb_entity_df"
## NULL
```

```r
if (!glb_is_separate_newent_dataset) {
    glb_trnent_df <- glb_entity_df; comment(glb_trnent_df) <- "glb_trnent_df"
} # else glb_entity_df is maintained as is for chunk:inspectORexplore.data
    
if (glb_is_separate_newent_dataset) {
    glb_newent_df <- myimport_data(
        url=glb_newdt_url, 
        comment="glb_newent_df", force_header=TRUE)
    
    # To make plots / stats / checks easier in chunk:inspectORexplore.data
    glb_entity_df <- rbind(glb_trnent_df, glb_newent_df); comment(glb_entity_df) <- "glb_entity_df"
} else {
    if (!glb_split_entity_newent_datasets) {
        stop("Not implemented yet") 
        glb_newent_df <- glb_trnent_df[sample(1:nrow(glb_trnent_df),
                                          max(2, nrow(glb_trnent_df) / 1000)),]                    
    } else      if (glb_split_newdata_method == "condition") {
            glb_newent_df <- do.call("subset", 
                list(glb_trnent_df, parse(text=glb_split_newdata_condition)))
            glb_trnent_df <- do.call("subset", 
                list(glb_trnent_df, parse(text=paste0("!(", 
                                                      glb_split_newdata_condition,
                                                      ")"))))
        } else if (glb_split_newdata_method == "sample") {
                require(caTools)
                
                set.seed(glb_split_sample.seed)
                split <- sample.split(glb_trnent_df[, glb_rsp_var_raw], 
                                      SplitRatio=(1-glb_split_newdata_size_ratio))
                glb_newent_df <- glb_trnent_df[!split, ] 
                glb_trnent_df <- glb_trnent_df[split ,]
        } else stop("glb_split_newdata_method should be %in% c('condition', 'sample')")   

    comment(glb_newent_df) <- "glb_newent_df"
    myprint_df(glb_newent_df)
    str(glb_newent_df)

    if (glb_split_entity_newent_datasets) {
        myprint_df(glb_trnent_df)
        str(glb_trnent_df)        
    }
}         
```

```
## Loading required package: caTools
```

```
##          TOWN TRACT      LON     LAT MEDV    CRIM   ZN INDUS CHAS   NOX
## 5  Marblehead  2032 -70.9220 42.2980 36.2 0.06905  0.0  2.18    0 0.458
## 8       Salem  2042 -70.9375 42.3100 22.1 0.14455 12.5  7.87    0 0.524
## 15       Lynn  2052 -70.9720 42.2870 18.2 0.63796  0.0  8.14    0 0.538
## 16       Lynn  2053 -70.9765 42.2940 19.9 0.62739  0.0  8.14    0 0.538
## 18       Lynn  2055 -70.9780 42.2850 17.5 0.78420  0.0  8.14    0 0.538
## 19       Lynn  2056 -70.9925 42.2825 20.2 0.80271  0.0  8.14    0 0.538
##       RM  AGE    DIS RAD TAX PTRATIO
## 5  7.147 54.2 6.0622   3 222    18.7
## 8  6.172 96.1 5.9505   5 311    15.2
## 15 6.096 84.5 4.4619   4 307    21.0
## 16 5.834 56.5 4.4986   4 307    21.0
## 18 5.990 81.7 4.2579   4 307    21.0
## 19 5.456 36.6 3.7965   4 307    21.0
##                    TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS
## 30                 Lynn  2067 -70.9510 42.2780 21.0 1.00245  0  8.14    0
## 159           Cambridge  3537 -71.0670 42.2245 24.3 1.34284  0 19.58    0
## 238              Newton  3748 -71.1491 42.2030 31.5 0.51183  0  6.20    0
## 260           Brookline  4003 -71.0765 42.2075 30.1 0.65665 20  3.97    0
## 282           Wellesley  4043 -71.1850 42.1848 35.4 0.03705 20  3.33    0
## 477 Boston Forest Hills  1204 -71.0565 42.1880 16.7 4.87141  0 18.10    0
##        NOX    RM   AGE    DIS RAD TAX PTRATIO
## 30  0.5380 6.674  87.3 4.2390   4 307    21.0
## 159 0.6050 6.066 100.0 1.7573   5 403    14.7
## 238 0.5070 7.358  71.6 4.1480   8 307    17.4
## 260 0.6470 6.842 100.0 2.0107   5 264    13.0
## 282 0.4429 6.968  37.2 5.2447   5 216    14.9
## 477 0.6140 6.484  93.6 2.3053  24 666    20.2
##         TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS   NOX    RM
## 492  Chelsea  1605 -71.0160 42.2382 13.6 0.10574  0 27.74    0 0.609 5.983
## 494   Revere  1701 -71.0125 42.2462 21.8 0.17331  0  9.69    0 0.585 5.707
## 497   Revere  1704 -71.0010 42.2525 19.7 0.28960  0  9.69    0 0.585 5.390
## 498   Revere  1705 -70.9947 42.2496 18.3 0.26838  0  9.69    0 0.585 5.794
## 499   Revere  1706 -71.0050 42.2455 21.2 0.23912  0  9.69    0 0.585 6.019
## 505 Winthrop  1804 -70.9875 42.2240 22.0 0.10959  0 11.93    0 0.573 6.794
##      AGE    DIS RAD TAX PTRATIO
## 492 98.8 1.8681   4 711    20.1
## 494 54.0 2.3817   6 391    19.2
## 497 72.9 2.7986   6 391    19.2
## 498 70.6 2.8927   6 391    19.2
## 499 65.3 2.4091   6 391    19.2
## 505 89.3 2.3889   1 273    21.0
## 'data.frame':	142 obs. of  16 variables:
##  $ TOWN   : chr  "Marblehead" "Salem" "Lynn" "Lynn" ...
##  $ TRACT  : int  2032 2042 2052 2053 2055 2056 2060 2061 2062 2064 ...
##  $ LON    : num  -70.9 -70.9 -71 -71 -71 ...
##  $ LAT    : num  42.3 42.3 42.3 42.3 42.3 ...
##  $ MEDV   : num  36.2 22.1 18.2 19.9 17.5 20.2 15.2 14.5 15.6 16.6 ...
##  $ CRIM   : num  0.0691 0.1446 0.638 0.6274 0.7842 ...
##  $ ZN     : num  0 12.5 0 0 0 0 0 0 0 0 ...
##  $ INDUS  : num  2.18 7.87 8.14 8.14 8.14 8.14 8.14 8.14 8.14 8.14 ...
##  $ CHAS   : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ NOX    : num  0.458 0.524 0.538 0.538 0.538 0.538 0.538 0.538 0.538 0.538 ...
##  $ RM     : num  7.15 6.17 6.1 5.83 5.99 ...
##  $ AGE    : num  54.2 96.1 84.5 56.5 81.7 36.6 91.7 100 94.1 90.3 ...
##  $ DIS    : num  6.06 5.95 4.46 4.5 4.26 ...
##  $ RAD    : int  3 5 4 4 4 4 4 4 4 4 ...
##  $ TAX    : int  222 311 307 307 307 307 307 307 307 307 ...
##  $ PTRATIO: num  18.7 15.2 21 21 21 21 21 21 21 21 ...
##  - attr(*, "comment")= chr "glb_newent_df"
##         TOWN TRACT      LON     LAT MEDV    CRIM   ZN INDUS CHAS   NOX
## 1     Nahant  2011 -70.9550 42.2550 24.0 0.00632 18.0  2.31    0 0.538
## 2 Swampscott  2021 -70.9500 42.2875 21.6 0.02731  0.0  7.07    0 0.469
## 3 Swampscott  2022 -70.9360 42.2830 34.7 0.02729  0.0  7.07    0 0.469
## 4 Marblehead  2031 -70.9280 42.2930 33.4 0.03237  0.0  2.18    0 0.458
## 6 Marblehead  2033 -70.9165 42.3040 28.7 0.02985  0.0  2.18    0 0.458
## 7      Salem  2041 -70.9360 42.2970 22.9 0.08829 12.5  7.87    0 0.524
##      RM  AGE    DIS RAD TAX PTRATIO
## 1 6.575 65.2 4.0900   1 296    15.3
## 2 6.421 78.9 4.9671   2 242    17.8
## 3 7.185 61.1 4.9671   2 242    17.8
## 4 6.998 45.8 6.0622   3 222    18.7
## 6 6.430 58.7 6.0622   3 222    18.7
## 7 6.012 66.6 5.5605   5 311    15.2
##                   TOWN TRACT      LON     LAT MEDV     CRIM   ZN INDUS
## 91             Melrose  3363 -71.0300 42.2720 22.6  0.04684  0.0  3.41
## 203            Wayland  3662 -71.2140 42.2180 42.3  0.02177 82.5  2.03
## 226             Newton  3736 -71.1012 42.1975 50.0  0.52693  0.0  6.20
## 367    Boston Back Bay   104 -71.0540 42.2052 21.9  3.69695  0.0 18.10
## 385 Boston East Boston   503 -71.0245 42.2235  8.8 20.08490  0.0 18.10
## 432     Boston Roxbury   820 -71.0505 42.1880 14.1 10.06230  0.0 18.10
##     CHAS   NOX    RM  AGE    DIS RAD TAX PTRATIO
## 91     0 0.489 6.417 66.1 3.0923   2 270    17.8
## 203    0 0.415 7.610 15.7 6.2700   2 348    14.7
## 226    0 0.504 8.725 83.0 2.8944   8 307    17.4
## 367    0 0.718 4.963 91.4 1.7523  24 666    20.2
## 385    0 0.700 4.368 91.2 1.4395  24 666    20.2
## 432    0 0.584 6.833 94.3 2.0882  24 666    20.2
##         TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS   NOX    RM
## 500   Revere  1707 -70.9985 42.2430 17.5 0.17783  0  9.69    0 0.585 5.569
## 501   Revere  1708 -70.9920 42.2380 16.8 0.22438  0  9.69    0 0.585 6.027
## 502 Winthrop  1801 -70.9860 42.2312 22.4 0.06263  0 11.93    0 0.573 6.593
## 503 Winthrop  1802 -70.9910 42.2275 20.6 0.04527  0 11.93    0 0.573 6.120
## 504 Winthrop  1803 -70.9948 42.2260 23.9 0.06076  0 11.93    0 0.573 6.976
## 506 Winthrop  1805 -70.9825 42.2210 19.0 0.04741  0 11.93    0 0.573 6.030
##      AGE    DIS RAD TAX PTRATIO
## 500 73.5 2.3999   6 391    19.2
## 501 79.7 2.4982   6 391    19.2
## 502 69.1 2.4786   1 273    21.0
## 503 76.7 2.2875   1 273    21.0
## 504 91.0 2.1675   1 273    21.0
## 506 80.8 2.5050   1 273    21.0
## 'data.frame':	364 obs. of  16 variables:
##  $ TOWN   : chr  "Nahant" "Swampscott" "Swampscott" "Marblehead" ...
##  $ TRACT  : int  2011 2021 2022 2031 2033 2041 2043 2044 2045 2046 ...
##  $ LON    : num  -71 -71 -70.9 -70.9 -70.9 ...
##  $ LAT    : num  42.3 42.3 42.3 42.3 42.3 ...
##  $ MEDV   : num  24 21.6 34.7 33.4 28.7 22.9 16.5 18.9 15 18.9 ...
##  $ CRIM   : num  0.00632 0.02731 0.02729 0.03237 0.02985 ...
##  $ ZN     : num  18 0 0 0 0 12.5 12.5 12.5 12.5 12.5 ...
##  $ INDUS  : num  2.31 7.07 7.07 2.18 2.18 7.87 7.87 7.87 7.87 7.87 ...
##  $ CHAS   : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ NOX    : num  0.538 0.469 0.469 0.458 0.458 0.524 0.524 0.524 0.524 0.524 ...
##  $ RM     : num  6.58 6.42 7.18 7 6.43 ...
##  $ AGE    : num  65.2 78.9 61.1 45.8 58.7 66.6 100 85.9 94.3 82.9 ...
##  $ DIS    : num  4.09 4.97 4.97 6.06 6.06 ...
##  $ RAD    : int  1 2 2 3 3 5 5 5 5 5 ...
##  $ TAX    : int  296 242 242 222 222 311 311 311 311 311 ...
##  $ PTRATIO: num  15.3 17.8 17.8 18.7 18.7 15.2 15.2 15.2 15.2 15.2 ...
##  - attr(*, "comment")= chr "glb_trnent_df"
```

```r
if (!is.null(glb_max_obs)) {
    if (nrow(glb_trnent_df) > glb_max_obs) {
        warning("glb_trnent_df restricted to glb_max_obs: ", format(glb_max_obs, big.mark=","))
        org_entity_df <- glb_trnent_df
        glb_trnent_df <- org_entity_df[split <- 
            sample.split(org_entity_df[, glb_rsp_var_raw], SplitRatio=glb_max_obs), ]
        org_entity_df <- NULL
    }
    if (nrow(glb_newent_df) > glb_max_obs) {
        warning("glb_newent_df restricted to glb_max_obs: ", format(glb_max_obs, big.mark=","))        
        org_newent_df <- glb_newent_df
        glb_newent_df <- org_newent_df[split <- 
            sample.split(org_newent_df[, glb_rsp_var_raw], SplitRatio=glb_max_obs), ]
        org_newent_df <- NULL
    }    
}

glb_script_df <- rbind(glb_script_df,
                   data.frame(chunk_label="cleanse_data", 
                              chunk_step_major=max(glb_script_df$chunk_step_major)+1, 
                              chunk_step_minor=0,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##           chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed   import_data                1                0   0.002
## elapsed1 cleanse_data                2                0   0.514
```

## Step `2`: cleanse data

```r
glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="inspectORexplore.data", 
                              chunk_step_major=max(glb_script_df$chunk_step_major), 
                              chunk_step_minor=1,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                    chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed1          cleanse_data                2                0   0.514
## elapsed2 inspectORexplore.data                2                1   0.548
```

### Step `2`.`1`: inspect/explore data

```r
#print(str(glb_trnent_df))
#View(glb_trnent_df)

# List info gathered for various columns
# <col_name>:   <description>; <notes>
# RM:   average # of rooms per dwelling

# Create new features that help diagnostics
#   Create factors of string variables
str_vars <- sapply(1:length(names(glb_trnent_df)), 
    function(col) ifelse(class(glb_trnent_df[, names(glb_trnent_df)[col]]) == "character",
                         names(glb_trnent_df)[col], ""))
if (length(str_vars <- setdiff(str_vars[str_vars != ""], 
                               glb_exclude_vars_as_features)) > 0) {
    warning("Creating factors of string variables:", paste0(str_vars, collapse=", "))
    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, str_vars)
    for (var in str_vars) {
        glb_entity_df[, paste0(var, ".fctr")] <- factor(glb_entity_df[, var], 
                        as.factor(unique(glb_entity_df[, var])))
        glb_trnent_df[, paste0(var, ".fctr")] <- factor(glb_trnent_df[, var], 
                        as.factor(unique(glb_entity_df[, var])))
        glb_newent_df[, paste0(var, ".fctr")] <- factor(glb_newent_df[, var], 
                        as.factor(unique(glb_entity_df[, var])))
    }
}
```

```
## Warning: Creating factors of string variables:TOWN
```

```r
#   Convert factors to dummy variables
#   Build splines   require(splines); bsBasis <- bs(training$age, df=3)

add_new_diag_feats <- function(obs_df, ref_df=glb_entity_df) {
    require(plyr)
    
    obs_df <- mutate(obs_df,
#         <col_name>.NA=is.na(<col_name>),

#         <col_name>.fctr=factor(<col_name>, 
#                     as.factor(union(obs_df$<col_name>, obs_twin_df$<col_name>))), 
#         <col_name>.fctr=relevel(factor(<col_name>, 
#                     as.factor(union(obs_df$<col_name>, obs_twin_df$<col_name>))),
#                                   "<ref_val>"), 
#         <col2_name>.fctr=relevel(factor(ifelse(<col1_name> == <val>, "<oth_val>", "<ref_val>")), 
#                               as.factor(c("R", "<ref_val>")),
#                               ref="<ref_val>"),

          # This doesn't work - use sapply instead
#         <col_name>.fctr_num=grep(<col_name>, levels(<col_name>.fctr)), 
#         
#         Date.my=as.Date(strptime(Date, "%m/%d/%y %H:%M")),
#         Year=year(Date.my),
#         Month=months(Date.my),
#         Weekday=weekdays(Date.my)

#         <col_name>.log=log(<col.name>),        
#         <col_name>=<table>[as.character(<col2_name>)],
#         <col_name>=as.numeric(<col2_name>),

        .rnorm=rnorm(n=nrow(obs_df))
                        )

    # If levels of a factor are different across obs_df & glb_newent_df; predict.glm fails  
    # Transformations not handled by mutate
#     obs_df$<col_name>.fctr.num <- sapply(1:nrow(obs_df), 
#         function(row_ix) grep(obs_df[row_ix, "<col_name>"],
#                               levels(obs_df[row_ix, "<col_name>.fctr"])))
    
    print(summary(obs_df))
    print(sapply(names(obs_df), function(col) sum(is.na(obs_df[, col]))))
    return(obs_df)
}

glb_entity_df <- add_new_diag_feats(glb_entity_df)
```

```
## Loading required package: plyr
```

```
##      TOWN               TRACT           LON              LAT       
##  Length:506         Min.   :   1   Min.   :-71.29   Min.   :42.03  
##  Class :character   1st Qu.:1303   1st Qu.:-71.09   1st Qu.:42.18  
##  Mode  :character   Median :3394   Median :-71.05   Median :42.22  
##                     Mean   :2700   Mean   :-71.06   Mean   :42.22  
##                     3rd Qu.:3740   3rd Qu.:-71.02   3rd Qu.:42.25  
##                     Max.   :5082   Max.   :-70.81   Max.   :42.38  
##                                                                    
##       MEDV            CRIM                ZN             INDUS      
##  Min.   : 5.00   Min.   : 0.00632   Min.   :  0.00   Min.   : 0.46  
##  1st Qu.:17.02   1st Qu.: 0.08204   1st Qu.:  0.00   1st Qu.: 5.19  
##  Median :21.20   Median : 0.25651   Median :  0.00   Median : 9.69  
##  Mean   :22.53   Mean   : 3.61352   Mean   : 11.36   Mean   :11.14  
##  3rd Qu.:25.00   3rd Qu.: 3.67708   3rd Qu.: 12.50   3rd Qu.:18.10  
##  Max.   :50.00   Max.   :88.97620   Max.   :100.00   Max.   :27.74  
##                                                                     
##       CHAS              NOX               RM             AGE        
##  Min.   :0.00000   Min.   :0.3850   Min.   :3.561   Min.   :  2.90  
##  1st Qu.:0.00000   1st Qu.:0.4490   1st Qu.:5.886   1st Qu.: 45.02  
##  Median :0.00000   Median :0.5380   Median :6.208   Median : 77.50  
##  Mean   :0.06917   Mean   :0.5547   Mean   :6.285   Mean   : 68.57  
##  3rd Qu.:0.00000   3rd Qu.:0.6240   3rd Qu.:6.623   3rd Qu.: 94.08  
##  Max.   :1.00000   Max.   :0.8710   Max.   :8.780   Max.   :100.00  
##                                                                     
##       DIS              RAD              TAX           PTRATIO     
##  Min.   : 1.130   Min.   : 1.000   Min.   :187.0   Min.   :12.60  
##  1st Qu.: 2.100   1st Qu.: 4.000   1st Qu.:279.0   1st Qu.:17.40  
##  Median : 3.207   Median : 5.000   Median :330.0   Median :19.05  
##  Mean   : 3.795   Mean   : 9.549   Mean   :408.2   Mean   :18.46  
##  3rd Qu.: 5.188   3rd Qu.:24.000   3rd Qu.:666.0   3rd Qu.:20.20  
##  Max.   :12.127   Max.   :24.000   Max.   :711.0   Max.   :22.00  
##                                                                   
##              TOWN.fctr       .rnorm        
##  Cambridge        : 30   Min.   :-2.80977  
##  Boston Savin Hill: 23   1st Qu.:-0.60689  
##  Lynn             : 22   Median : 0.06943  
##  Boston Roxbury   : 19   Mean   : 0.01940  
##  Newton           : 18   3rd Qu.: 0.70401  
##  Somerville       : 15   Max.   : 2.69171  
##  (Other)          :379                     
##      TOWN     TRACT       LON       LAT      MEDV      CRIM        ZN 
##         0         0         0         0         0         0         0 
##     INDUS      CHAS       NOX        RM       AGE       DIS       RAD 
##         0         0         0         0         0         0         0 
##       TAX   PTRATIO TOWN.fctr    .rnorm 
##         0         0         0         0
```

```r
glb_trnent_df <- add_new_diag_feats(glb_trnent_df)
```

```
##      TOWN               TRACT           LON              LAT       
##  Length:364         Min.   :   1   Min.   :-71.29   Min.   :42.05  
##  Class :character   1st Qu.:1304   1st Qu.:-71.10   1st Qu.:42.18  
##  Mode  :character   Median :3398   Median :-71.06   Median :42.22  
##                     Mean   :2718   Mean   :-71.06   Mean   :42.22  
##                     3rd Qu.:3740   3rd Qu.:-71.02   3rd Qu.:42.25  
##                     Max.   :5081   Max.   :-70.83   Max.   :42.38  
##                                                                    
##       MEDV            CRIM                ZN             INDUS       
##  Min.   : 5.00   Min.   : 0.00632   Min.   :  0.00   Min.   : 0.740  
##  1st Qu.:17.18   1st Qu.: 0.07119   1st Qu.:  0.00   1st Qu.: 5.175  
##  Median :21.40   Median : 0.23746   Median :  0.00   Median : 9.690  
##  Mean   :22.93   Mean   : 3.57692   Mean   : 11.72   Mean   :11.092  
##  3rd Qu.:26.52   3rd Qu.: 3.68194   3rd Qu.: 17.62   3rd Qu.:18.100  
##  Max.   :50.00   Max.   :88.97620   Max.   :100.00   Max.   :27.740  
##                                                                      
##       CHAS              NOX               RM             AGE        
##  Min.   :0.00000   Min.   :0.3850   Min.   :3.561   Min.   :  6.60  
##  1st Qu.:0.00000   1st Qu.:0.4480   1st Qu.:5.879   1st Qu.: 42.55  
##  Median :0.00000   Median :0.5240   Median :6.199   Median : 76.70  
##  Mean   :0.06868   Mean   :0.5534   Mean   :6.284   Mean   : 68.05  
##  3rd Qu.:0.00000   3rd Qu.:0.6240   3rd Qu.:6.629   3rd Qu.: 92.92  
##  Max.   :1.00000   Max.   :0.8710   Max.   :8.725   Max.   :100.00  
##                                                                     
##       DIS              RAD              TAX           PTRATIO     
##  Min.   : 1.130   Min.   : 1.000   Min.   :187.0   Min.   :12.60  
##  1st Qu.: 2.112   1st Qu.: 4.000   1st Qu.:277.0   1st Qu.:16.90  
##  Median : 3.207   Median : 5.000   Median :329.0   Median :18.70  
##  Mean   : 3.804   Mean   : 9.533   Mean   :403.1   Mean   :18.35  
##  3rd Qu.: 5.118   3rd Qu.:24.000   3rd Qu.:666.0   3rd Qu.:20.20  
##  Max.   :12.127   Max.   :24.000   Max.   :711.0   Max.   :22.00  
##                                                                   
##              TOWN.fctr       .rnorm        
##  Cambridge        : 21   Min.   :-2.54934  
##  Boston Roxbury   : 15   1st Qu.:-0.64329  
##  Newton           : 14   Median : 0.05482  
##  Boston Savin Hill: 14   Mean   : 0.05520  
##  Lynn             : 10   3rd Qu.: 0.69298  
##  Quincy           : 10   Max.   : 3.18404  
##  (Other)          :280                     
##      TOWN     TRACT       LON       LAT      MEDV      CRIM        ZN 
##         0         0         0         0         0         0         0 
##     INDUS      CHAS       NOX        RM       AGE       DIS       RAD 
##         0         0         0         0         0         0         0 
##       TAX   PTRATIO TOWN.fctr    .rnorm 
##         0         0         0         0
```

```r
glb_newent_df <- add_new_diag_feats(glb_newent_df)
```

```
##      TOWN               TRACT           LON              LAT       
##  Length:142         Min.   :   8   Min.   :-71.27   Min.   :42.03  
##  Class :character   1st Qu.:1328   1st Qu.:-71.08   1st Qu.:42.18  
##  Mode  :character   Median :3357   Median :-71.05   Median :42.22  
##                     Mean   :2655   Mean   :-71.05   Mean   :42.22  
##                     3rd Qu.:3736   3rd Qu.:-71.01   3rd Qu.:42.25  
##                     Max.   :5082   Max.   :-70.81   Max.   :42.37  
##                                                                    
##       MEDV            CRIM                ZN            INDUS      
##  Min.   : 5.00   Min.   : 0.01096   Min.   : 0.00   Min.   : 0.46  
##  1st Qu.:16.62   1st Qu.: 0.10082   1st Qu.: 0.00   1st Qu.: 5.19  
##  Median :20.75   Median : 0.31002   Median : 0.00   Median : 9.69  
##  Mean   :21.49   Mean   : 3.70736   Mean   :10.45   Mean   :11.25  
##  3rd Qu.:24.18   3rd Qu.: 3.54508   3rd Qu.: 0.00   3rd Qu.:18.10  
##  Max.   :50.00   Max.   :73.53410   Max.   :95.00   Max.   :27.74  
##                                                                    
##       CHAS              NOX               RM             AGE        
##  Min.   :0.00000   Min.   :0.3890   Min.   :4.652   Min.   :  2.90  
##  1st Qu.:0.00000   1st Qu.:0.4585   1st Qu.:5.907   1st Qu.: 47.95  
##  Median :0.00000   Median :0.5380   Median :6.210   Median : 80.80  
##  Mean   :0.07042   Mean   :0.5580   Mean   :6.286   Mean   : 69.93  
##  3rd Qu.:0.00000   3rd Qu.:0.6240   3rd Qu.:6.560   3rd Qu.: 95.62  
##  Max.   :1.00000   Max.   :0.8710   Max.   :8.780   Max.   :100.00  
##                                                                     
##       DIS              RAD              TAX           PTRATIO     
##  Min.   : 1.174   Min.   : 1.000   Min.   :188.0   Min.   :13.00  
##  1st Qu.: 1.992   1st Qu.: 4.000   1st Qu.:301.0   1st Qu.:17.60  
##  Median : 3.184   Median : 5.000   Median :387.5   Median :19.20  
##  Mean   : 3.772   Mean   : 9.592   Mean   :421.4   Mean   :18.74  
##  3rd Qu.: 5.237   3rd Qu.:24.000   3rd Qu.:666.0   3rd Qu.:20.20  
##  Max.   :10.710   Max.   :24.000   Max.   :711.0   Max.   :22.00  
##                                                                   
##              TOWN.fctr      .rnorm        
##  Lynn             :12   Min.   :-2.19892  
##  Cambridge        : 9   1st Qu.:-0.72116  
##  Boston Savin Hill: 9   Median : 0.03080  
##  Somerville       : 6   Mean   :-0.01093  
##  Malden           : 4   3rd Qu.: 0.62214  
##  Newton           : 4   Max.   : 2.19359  
##  (Other)          :98                     
##      TOWN     TRACT       LON       LAT      MEDV      CRIM        ZN 
##         0         0         0         0         0         0         0 
##     INDUS      CHAS       NOX        RM       AGE       DIS       RAD 
##         0         0         0         0         0         0         0 
##       TAX   PTRATIO TOWN.fctr    .rnorm 
##         0         0         0         0
```

```r
# Histogram of predictor in glb_trnent_df & glb_newent_df
plot_df <- rbind(cbind(glb_trnent_df[, glb_rsp_var_raw, FALSE], data.frame(.data="Training")),
                 cbind(glb_trnent_df[, glb_rsp_var_raw, FALSE], data.frame(.data="New")))
print(myplot_histogram(plot_df, glb_rsp_var_raw) + facet_wrap(~ .data))
```

```
## stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust this.
## stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust this.
```

![](Boston_Housing_files/figure-html/inspectORexplore_data-1.png) 

```r
if (glb_is_classification) {
    #print(table(df[, glb_rsp_var_raw]) / nrow(df))    
    print((xtab_df <- mycreate_xtab(plot_df, c(".data", glb_rsp_var_raw))) / 
              sum(xtab_df))    
}    

# Check for duplicates in glb_id_vars
if (length(glb_id_vars) > 0) {
    id_vars_dups_df <- subset(id_vars_df <- 
            mycreate_tbl_df(glb_entity_df[, glb_id_vars, FALSE], glb_id_vars),
                                .freq > 1)
    if (nrow(id_vars_dups_df) > 0) {
        warning("Duplicates found in glb_id_vars data:", nrow(id_vars_dups_df))
        myprint_df(id_vars_dups_df)
    } else {
        # glb_id_vars are unique across obs in both glb_<>_df
        glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, glb_id_vars)
    }
}

#pairs(subset(glb_trnent_df, select=-c(col_symbol)))
# Check for glb_newent_df & glb_trnent_df features range mismatches

# Other diagnostics:
# print(subset(glb_trnent_df, <col1_name> == max(glb_trnent_df$<col1_name>, na.rm=TRUE) & 
#                         <col2_name> <= mean(glb_trnent_df$<col1_name>, na.rm=TRUE)))

# print(glb_trnent_df[which.max(glb_trnent_df$<col_name>),])

# print(<col_name>_freq_glb_trnent_df <- mycreate_tbl_df(glb_trnent_df, "<col_name>"))
# print(which.min(table(glb_trnent_df$<col_name>)))
# print(which.max(table(glb_trnent_df$<col_name>)))
# print(which.max(table(glb_trnent_df$<col1_name>, glb_trnent_df$<col2_name>)[, 2]))
# print(table(glb_trnent_df$<col1_name>, glb_trnent_df$<col2_name>))
# print(table(is.na(glb_trnent_df$<col1_name>), glb_trnent_df$<col2_name>))
# print(table(sign(glb_trnent_df$<col1_name>), glb_trnent_df$<col2_name>))
# print(mycreate_xtab(glb_trnent_df, <col1_name>))
# print(mycreate_xtab(glb_trnent_df, c(<col1_name>, <col2_name>)))
# print(<col1_name>_<col2_name>_xtab_glb_trnent_df <- 
#   mycreate_xtab(glb_trnent_df, c("<col1_name>", "<col2_name>")))
# <col1_name>_<col2_name>_xtab_glb_trnent_df[is.na(<col1_name>_<col2_name>_xtab_glb_trnent_df)] <- 0
# print(<col1_name>_<col2_name>_xtab_glb_trnent_df <- 
#   mutate(<col1_name>_<col2_name>_xtab_glb_trnent_df, 
#             <col3_name>=(<col1_name> * 1.0) / (<col1_name> + <col2_name>))) 

# print(<col2_name>_min_entity_arr <- 
#    sort(tapply(glb_trnent_df$<col1_name>, glb_trnent_df$<col2_name>, min, na.rm=TRUE)))
# print(<col1_name>_na_by_<col2_name>_arr <- 
#    sort(tapply(glb_trnent_df$<col1_name>.NA, glb_trnent_df$<col2_name>, mean, na.rm=TRUE)))

# Other plots:
# print(myplot_box(df=glb_trnent_df, ycol_names="<col1_name>"))
# print(myplot_box(df=glb_trnent_df, ycol_names="<col1_name>", xcol_name="<col2_name>"))
# print(myplot_line(subset(glb_trnent_df, Symbol %in% c("KO", "PG")), 
#                   "Date.my", "StockPrice", facet_row_colnames="Symbol") + 
#     geom_vline(xintercept=as.numeric(as.Date("2003-03-01"))) +
#     geom_vline(xintercept=as.numeric(as.Date("1983-01-01")))        
#         )
# print(myplot_scatter(glb_trnent_df, "<col1_name>", "<col2_name>", smooth=TRUE))
# print(myplot_scatter(glb_entity_df, "<col1_name>", "<col2_name>", colorcol_name="<Pred.fctr>"))
print(myplot_scatter(df=glb_entity_df, xcol_name="LON", ycol_name="LAT", colorcol_name="CHAS"))
```

```
## Warning in myplot_scatter(df = glb_entity_df, xcol_name = "LON", ycol_name
## = "LAT", : converting CHAS to class:factor
```

![](Boston_Housing_files/figure-html/inspectORexplore_data-2.png) 

```r
print(myplot_scatter(df=glb_entity_df, xcol_name="LON", ycol_name="LAT") + 
        geom_point(data=subset(glb_entity_df, CHAS == 1), 
                    mapping=aes(x=LON, y=LAT), color="blue") +
        geom_point(data=subset(glb_entity_df, TRACT == 3531), 
                    mapping=aes(x=LON, y=LAT), color="red", shape=4, size=5) + 
        geom_point(data=subset(glb_entity_df, NOX > mean(glb_entity_df$NOX)), 
                    mapping=aes(x=LON, y=LAT), color="green"))
```

![](Boston_Housing_files/figure-html/inspectORexplore_data-3.png) 

```r
glb_script_df <- rbind(glb_script_df, 
    data.frame(chunk_label="manage_missing_data", 
        chunk_step_major=max(glb_script_df$chunk_step_major), 
        chunk_step_minor=glb_script_df[nrow(glb_script_df), "chunk_step_minor"]+1,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                    chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed2 inspectORexplore.data                2                1   0.548
## elapsed3   manage_missing_data                2                2   2.415
```

### Step `2`.`2`: manage missing data

```r
# print(sapply(names(glb_trnent_df), function(col) sum(is.na(glb_trnent_df[, col]))))
# print(sapply(names(glb_newent_df), function(col) sum(is.na(glb_newent_df[, col]))))
# glb_trnent_df <- na.omit(glb_trnent_df)
# glb_newent_df <- na.omit(glb_newent_df)
# df[is.na(df)] <- 0

# Not refactored into mydsutils.R since glb_*_df might be reassigned
glb_impute_missing_data <- function(entity_df, newent_df) {
    if (!glb_is_separate_newent_dataset) {
        # Combine entity & newent
        union_df <- rbind(mutate(entity_df, .src = "entity"),
                          mutate(newent_df, .src = "newent"))
        union_imputed_df <- union_df[, setdiff(setdiff(names(entity_df), 
                                                       glb_rsp_var), 
                                               glb_exclude_vars_as_features)]
        print(summary(union_imputed_df))
    
        require(mice)
        set.seed(glb_mice_complete.seed)
        union_imputed_df <- complete(mice(union_imputed_df))
        print(summary(union_imputed_df))
    
        union_df[, names(union_imputed_df)] <- union_imputed_df[, names(union_imputed_df)]
        print(summary(union_df))
#         union_df$.rownames <- rownames(union_df)
#         union_df <- orderBy(~.rownames, union_df)
#         
#         imp_entity_df <- myimport_data(
#             url="<imputed_trnng_url>", 
#             comment="imp_entity_df", force_header=TRUE, print_diagn=TRUE)
#         print(all.equal(subset(union_df, select=-c(.src, .rownames, .rnorm)), 
#                         imp_entity_df))
        
        # Partition again
        glb_trnent_df <<- subset(union_df, .src == "entity", select=-c(.src, .rownames))
        comment(glb_trnent_df) <- "entity_df"
        glb_newent_df <<- subset(union_df, .src == "newent", select=-c(.src, .rownames))
        comment(glb_newent_df) <- "newent_df"
        
        # Generate summaries
        print(summary(entity_df))
        print(sapply(names(entity_df), function(col) sum(is.na(entity_df[, col]))))
        print(summary(newent_df))
        print(sapply(names(newent_df), function(col) sum(is.na(newent_df[, col]))))
    
    } else stop("Not implemented yet")
}

if (glb_impute_na_data) {
    if ((sum(sapply(names(glb_trnent_df), 
                    function(col) sum(is.na(glb_trnent_df[, col])))) > 0) | 
        (sum(sapply(names(glb_newent_df), 
                    function(col) sum(is.na(glb_newent_df[, col])))) > 0))
        glb_impute_missing_data(glb_trnent_df, glb_newent_df)
}    

glb_script_df <- rbind(glb_script_df, 
    data.frame(chunk_label="encode_retype_data", 
        chunk_step_major=max(glb_script_df$chunk_step_major), 
        chunk_step_minor=glb_script_df[nrow(glb_script_df), "chunk_step_minor"]+1,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                  chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed3 manage_missing_data                2                2   2.415
## elapsed4  encode_retype_data                2                3   2.949
```

### Step `2`.`3`: encode/retype data

```r
# map_<col_name>_df <- myimport_data(
#     url="<map_url>", 
#     comment="map_<col_name>_df", print_diagn=TRUE)
# map_<col_name>_df <- read.csv(paste0(getwd(), "/data/<file_name>.csv"), strip.white=TRUE)

# glb_trnent_df <- mymap_codes(glb_trnent_df, "<from_col_name>", "<to_col_name>", 
#     map_<to_col_name>_df, map_join_col_name="<map_join_col_name>", 
#                           map_tgt_col_name="<to_col_name>")
# glb_newent_df <- mymap_codes(glb_newent_df, "<from_col_name>", "<to_col_name>", 
#     map_<to_col_name>_df, map_join_col_name="<map_join_col_name>", 
#                           map_tgt_col_name="<to_col_name>")
    					
# glb_trnent_df$<col_name>.fctr <- factor(glb_trnent_df$<col_name>, 
#                     as.factor(union(glb_trnent_df$<col_name>, glb_newent_df$<col_name>)))
# glb_newent_df$<col_name>.fctr <- factor(glb_newent_df$<col_name>, 
#                     as.factor(union(glb_trnent_df$<col_name>, glb_newent_df$<col_name>)))

if (!is.null(glb_map_rsp_raw_to_var)) {
    glb_trnent_df[, glb_rsp_var] <- 
        glb_map_rsp_raw_to_var(glb_trnent_df[, glb_rsp_var_raw])
    mycheck_map_results(mapd_df=glb_trnent_df, 
                        from_col_name=glb_rsp_var_raw, to_col_name=glb_rsp_var)
        
    glb_newent_df[, glb_rsp_var] <- 
        glb_map_rsp_raw_to_var(glb_newent_df[, glb_rsp_var_raw])
    mycheck_map_results(mapd_df=glb_newent_df, 
                        from_col_name=glb_rsp_var_raw, to_col_name=glb_rsp_var)    
}

glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="extract_features", 
                              chunk_step_major=max(glb_script_df$chunk_step_major)+1, 
                              chunk_step_minor=0,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                 chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed4 encode_retype_data                2                3   2.949
## elapsed5   extract_features                3                0   3.007
```

## Step `3`: extract features

```r
# Create new features that help prediction
# <col_name>.lag.2 <- lag(zoo(glb_trnent_df$<col_name>), -2, na.pad=TRUE)
# glb_trnent_df[, "<col_name>.lag.2"] <- coredata(<col_name>.lag.2)
# <col_name>.lag.2 <- lag(zoo(glb_newent_df$<col_name>), -2, na.pad=TRUE)
# glb_newent_df[, "<col_name>.lag.2"] <- coredata(<col_name>.lag.2)
# 
# glb_newent_df[1, "<col_name>.lag.2"] <- glb_trnent_df[nrow(glb_trnent_df) - 1, 
#                                                    "<col_name>"]
# glb_newent_df[2, "<col_name>.lag.2"] <- glb_trnent_df[nrow(glb_trnent_df), 
#                                                    "<col_name>"]
                                                   
# glb_trnent_df <- mutate(glb_trnent_df,
#     <new_col_name>=
#                     )

# glb_newent_df <- mutate(glb_newent_df,
#     <new_col_name>=
#                     )

# print(summary(glb_trnent_df))
# print(summary(glb_newent_df))

# print(sapply(names(glb_trnent_df), function(col) sum(is.na(glb_trnent_df[, col]))))
# print(sapply(names(glb_newent_df), function(col) sum(is.na(glb_newent_df[, col]))))

# print(myplot_scatter(glb_trnent_df, "<col1_name>", "<col2_name>", smooth=TRUE))

replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "data.training.all","data.new")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0
```

![](Boston_Housing_files/figure-html/extract_features-1.png) 

```r
glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="select_features", 
                              chunk_step_major=max(glb_script_df$chunk_step_major)+1, 
                              chunk_step_minor=0,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##               chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed5 extract_features                3                0   3.007
## elapsed6  select_features                4                0   3.799
```

## Step `4`: select features

```r
print(glb_feats_df <- myselect_features(entity_df=glb_trnent_df, 
                       exclude_vars_as_features=glb_exclude_vars_as_features, 
                       rsp_var=glb_rsp_var))
```

```
##                  id       cor.y exclude.as.feat  cor.y.abs
## RM               RM  0.70872979               0 0.70872979
## PTRATIO     PTRATIO -0.50539619               0 0.50539619
## INDUS         INDUS -0.49397282               0 0.49397282
## TAX             TAX -0.47940636               0 0.47940636
## TRACT         TRACT  0.44111242               1 0.44111242
## NOX             NOX -0.43202955               0 0.43202955
## RAD             RAD -0.40830448               0 0.40830448
## CRIM           CRIM -0.38466671               0 0.38466671
## AGE             AGE -0.37857924               0 0.37857924
## ZN               ZN  0.36446804               0 0.36446804
## LON             LON -0.33341021               0 0.33341021
## TOWN.fctr TOWN.fctr -0.28659819               1 0.28659819
## DIS             DIS  0.25834984               0 0.25834984
## CHAS           CHAS  0.16496009               0 0.16496009
## .rnorm       .rnorm  0.02567817               0 0.02567817
## LAT             LAT  0.01854813               0 0.01854813
```

```r
glb_script_df <- rbind(glb_script_df, 
    data.frame(chunk_label="remove_correlated_features", 
        chunk_step_major=max(glb_script_df$chunk_step_major),
        chunk_step_minor=glb_script_df[nrow(glb_script_df), "chunk_step_minor"]+1,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))        
print(tail(glb_script_df, 2))
```

```
##                         chunk_label chunk_step_major chunk_step_minor
## elapsed6            select_features                4                0
## elapsed7 remove_correlated_features                4                1
##          elapsed
## elapsed6   3.799
## elapsed7   3.983
```

### Step `4`.`1`: remove correlated features

```r
print(glb_feats_df <- orderBy(~-cor.y, 
          myfind_cor_features(feats_df=glb_feats_df, entity_df=glb_trnent_df, 
                                rsp_var=glb_rsp_var)))
```

```
## Loading required package: reshape2
```

```
##                  RM     PTRATIO       INDUS          TAX         NOX
## RM       1.00000000 -0.37984925 -0.43197466 -0.339950064 -0.33472738
## PTRATIO -0.37984925  1.00000000  0.40434610  0.485502357  0.20282393
## INDUS   -0.43197466  0.40434610  1.00000000  0.715219161  0.75758099
## TAX     -0.33995006  0.48550236  0.71521916  1.000000000  0.66979694
## NOX     -0.33472738  0.20282393  0.75758099  0.669796937  1.00000000
## RAD     -0.27199832  0.49572747  0.59917116  0.921134676  0.61203716
## CRIM    -0.23575073  0.30376946  0.40387028  0.580683778  0.41068006
## AGE     -0.26820079  0.26522697  0.64676404  0.514797156  0.72190662
## ZN       0.33045445 -0.42268028 -0.53829542 -0.325469616 -0.51100975
## LON     -0.28360091  0.28584885  0.08569838  0.049851547  0.16959303
## DIS      0.24315562 -0.26817765 -0.69668199 -0.534139751 -0.76220420
## CHAS     0.08847498 -0.14452838  0.03578039 -0.073053205  0.07199604
## .rnorm   0.03550487 -0.02668252  0.03323732 -0.001100587  0.05833104
## LAT     -0.05378927 -0.06148204 -0.08015774 -0.176870069 -0.09904524
##                  RAD         CRIM         AGE          ZN         LON
## RM      -0.271998315 -0.235750727 -0.26820079  0.33045445 -0.28360091
## PTRATIO  0.495727470  0.303769465  0.26522697 -0.42268028  0.28584885
## INDUS    0.599171160  0.403870283  0.64676404 -0.53829542  0.08569838
## TAX      0.921134676  0.580683778  0.51479716 -0.32546962  0.04985155
## NOX      0.612037164  0.410680057  0.72190662 -0.51100975  0.16959303
## RAD      1.000000000  0.616701423  0.46688639 -0.31860255  0.05037899
## CRIM     0.616701423  1.000000000  0.35315440 -0.19902174  0.07799185
## AGE      0.466886394  0.353154400  1.00000000 -0.57047282  0.23164927
## ZN      -0.318602546 -0.199021738 -0.57047282  1.00000000 -0.23489301
## LON      0.050378987  0.077991852  0.23164927 -0.23489301  1.00000000
## DIS     -0.492788820 -0.379592128 -0.73563604  0.65732096 -0.03822171
## CHAS    -0.047783085 -0.062760814  0.06568424 -0.02878224 -0.18878306
## .rnorm   0.002990504  0.004542403  0.06193895 -0.01767417  0.07746727
## LAT     -0.223844364 -0.087829088  0.05045751 -0.05195114  0.17124146
##                 DIS         CHAS       .rnorm         LAT
## RM       0.24315562  0.088474975  0.035504870 -0.05378927
## PTRATIO -0.26817765 -0.144528380 -0.026682516 -0.06148204
## INDUS   -0.69668199  0.035780390  0.033237322 -0.08015774
## TAX     -0.53413975 -0.073053205 -0.001100587 -0.17687007
## NOX     -0.76220420  0.071996036  0.058331040 -0.09904524
## RAD     -0.49278882 -0.047783085  0.002990504 -0.22384436
## CRIM    -0.37959213 -0.062760814  0.004542403 -0.08782909
## AGE     -0.73563604  0.065684238  0.061938951  0.05045751
## ZN       0.65732096 -0.028782240 -0.017674169 -0.05195114
## LON     -0.03822171 -0.188783063  0.077467273  0.17124146
## DIS      1.00000000 -0.072548035 -0.020291355 -0.03324755
## CHAS    -0.07254804  1.000000000  0.004021079 -0.05537654
## .rnorm  -0.02029136  0.004021079  1.000000000 -0.01017963
## LAT     -0.03324755 -0.055376536 -0.010179628  1.00000000
##                 RM    PTRATIO      INDUS         TAX        NOX
## RM      0.00000000 0.37984925 0.43197466 0.339950064 0.33472738
## PTRATIO 0.37984925 0.00000000 0.40434610 0.485502357 0.20282393
## INDUS   0.43197466 0.40434610 0.00000000 0.715219161 0.75758099
## TAX     0.33995006 0.48550236 0.71521916 0.000000000 0.66979694
## NOX     0.33472738 0.20282393 0.75758099 0.669796937 0.00000000
## RAD     0.27199832 0.49572747 0.59917116 0.921134676 0.61203716
## CRIM    0.23575073 0.30376946 0.40387028 0.580683778 0.41068006
## AGE     0.26820079 0.26522697 0.64676404 0.514797156 0.72190662
## ZN      0.33045445 0.42268028 0.53829542 0.325469616 0.51100975
## LON     0.28360091 0.28584885 0.08569838 0.049851547 0.16959303
## DIS     0.24315562 0.26817765 0.69668199 0.534139751 0.76220420
## CHAS    0.08847498 0.14452838 0.03578039 0.073053205 0.07199604
## .rnorm  0.03550487 0.02668252 0.03323732 0.001100587 0.05833104
## LAT     0.05378927 0.06148204 0.08015774 0.176870069 0.09904524
##                 RAD        CRIM        AGE         ZN        LON
## RM      0.271998315 0.235750727 0.26820079 0.33045445 0.28360091
## PTRATIO 0.495727470 0.303769465 0.26522697 0.42268028 0.28584885
## INDUS   0.599171160 0.403870283 0.64676404 0.53829542 0.08569838
## TAX     0.921134676 0.580683778 0.51479716 0.32546962 0.04985155
## NOX     0.612037164 0.410680057 0.72190662 0.51100975 0.16959303
## RAD     0.000000000 0.616701423 0.46688639 0.31860255 0.05037899
## CRIM    0.616701423 0.000000000 0.35315440 0.19902174 0.07799185
## AGE     0.466886394 0.353154400 0.00000000 0.57047282 0.23164927
## ZN      0.318602546 0.199021738 0.57047282 0.00000000 0.23489301
## LON     0.050378987 0.077991852 0.23164927 0.23489301 0.00000000
## DIS     0.492788820 0.379592128 0.73563604 0.65732096 0.03822171
## CHAS    0.047783085 0.062760814 0.06568424 0.02878224 0.18878306
## .rnorm  0.002990504 0.004542403 0.06193895 0.01767417 0.07746727
## LAT     0.223844364 0.087829088 0.05045751 0.05195114 0.17124146
##                DIS        CHAS      .rnorm        LAT
## RM      0.24315562 0.088474975 0.035504870 0.05378927
## PTRATIO 0.26817765 0.144528380 0.026682516 0.06148204
## INDUS   0.69668199 0.035780390 0.033237322 0.08015774
## TAX     0.53413975 0.073053205 0.001100587 0.17687007
## NOX     0.76220420 0.071996036 0.058331040 0.09904524
## RAD     0.49278882 0.047783085 0.002990504 0.22384436
## CRIM    0.37959213 0.062760814 0.004542403 0.08782909
## AGE     0.73563604 0.065684238 0.061938951 0.05045751
## ZN      0.65732096 0.028782240 0.017674169 0.05195114
## LON     0.03822171 0.188783063 0.077467273 0.17124146
## DIS     0.00000000 0.072548035 0.020291355 0.03324755
## CHAS    0.07254804 0.000000000 0.004021079 0.05537654
## .rnorm  0.02029136 0.004021079 0.000000000 0.01017963
## LAT     0.03324755 0.055376536 0.010179628 0.00000000
## [1] "cor(TAX, RAD)=0.9211"
```

![](Boston_Housing_files/figure-html/remove_correlated_features-1.png) 

```
## [1] "cor(MEDV, TAX)=-0.4794"
## [1] "cor(MEDV, RAD)=-0.4083"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, entity_df =
## glb_trnent_df, : Identified RAD as highly correlated with other features
```

![](Boston_Housing_files/figure-html/remove_correlated_features-2.png) 

```
## [1] "checking correlations for features:"
##  [1] "RM"      "PTRATIO" "INDUS"   "TAX"     "NOX"     "CRIM"    "AGE"    
##  [8] "ZN"      "LON"     "DIS"     "CHAS"    ".rnorm"  "LAT"    
##                  RM     PTRATIO       INDUS          TAX         NOX
## RM       1.00000000 -0.37984925 -0.43197466 -0.339950064 -0.33472738
## PTRATIO -0.37984925  1.00000000  0.40434610  0.485502357  0.20282393
## INDUS   -0.43197466  0.40434610  1.00000000  0.715219161  0.75758099
## TAX     -0.33995006  0.48550236  0.71521916  1.000000000  0.66979694
## NOX     -0.33472738  0.20282393  0.75758099  0.669796937  1.00000000
## CRIM    -0.23575073  0.30376946  0.40387028  0.580683778  0.41068006
## AGE     -0.26820079  0.26522697  0.64676404  0.514797156  0.72190662
## ZN       0.33045445 -0.42268028 -0.53829542 -0.325469616 -0.51100975
## LON     -0.28360091  0.28584885  0.08569838  0.049851547  0.16959303
## DIS      0.24315562 -0.26817765 -0.69668199 -0.534139751 -0.76220420
## CHAS     0.08847498 -0.14452838  0.03578039 -0.073053205  0.07199604
## .rnorm   0.03550487 -0.02668252  0.03323732 -0.001100587  0.05833104
## LAT     -0.05378927 -0.06148204 -0.08015774 -0.176870069 -0.09904524
##                 CRIM         AGE          ZN         LON         DIS
## RM      -0.235750727 -0.26820079  0.33045445 -0.28360091  0.24315562
## PTRATIO  0.303769465  0.26522697 -0.42268028  0.28584885 -0.26817765
## INDUS    0.403870283  0.64676404 -0.53829542  0.08569838 -0.69668199
## TAX      0.580683778  0.51479716 -0.32546962  0.04985155 -0.53413975
## NOX      0.410680057  0.72190662 -0.51100975  0.16959303 -0.76220420
## CRIM     1.000000000  0.35315440 -0.19902174  0.07799185 -0.37959213
## AGE      0.353154400  1.00000000 -0.57047282  0.23164927 -0.73563604
## ZN      -0.199021738 -0.57047282  1.00000000 -0.23489301  0.65732096
## LON      0.077991852  0.23164927 -0.23489301  1.00000000 -0.03822171
## DIS     -0.379592128 -0.73563604  0.65732096 -0.03822171  1.00000000
## CHAS    -0.062760814  0.06568424 -0.02878224 -0.18878306 -0.07254804
## .rnorm   0.004542403  0.06193895 -0.01767417  0.07746727 -0.02029136
## LAT     -0.087829088  0.05045751 -0.05195114  0.17124146 -0.03324755
##                 CHAS       .rnorm         LAT
## RM       0.088474975  0.035504870 -0.05378927
## PTRATIO -0.144528380 -0.026682516 -0.06148204
## INDUS    0.035780390  0.033237322 -0.08015774
## TAX     -0.073053205 -0.001100587 -0.17687007
## NOX      0.071996036  0.058331040 -0.09904524
## CRIM    -0.062760814  0.004542403 -0.08782909
## AGE      0.065684238  0.061938951  0.05045751
## ZN      -0.028782240 -0.017674169 -0.05195114
## LON     -0.188783063  0.077467273  0.17124146
## DIS     -0.072548035 -0.020291355 -0.03324755
## CHAS     1.000000000  0.004021079 -0.05537654
## .rnorm   0.004021079  1.000000000 -0.01017963
## LAT     -0.055376536 -0.010179628  1.00000000
##                 RM    PTRATIO      INDUS         TAX        NOX
## RM      0.00000000 0.37984925 0.43197466 0.339950064 0.33472738
## PTRATIO 0.37984925 0.00000000 0.40434610 0.485502357 0.20282393
## INDUS   0.43197466 0.40434610 0.00000000 0.715219161 0.75758099
## TAX     0.33995006 0.48550236 0.71521916 0.000000000 0.66979694
## NOX     0.33472738 0.20282393 0.75758099 0.669796937 0.00000000
## CRIM    0.23575073 0.30376946 0.40387028 0.580683778 0.41068006
## AGE     0.26820079 0.26522697 0.64676404 0.514797156 0.72190662
## ZN      0.33045445 0.42268028 0.53829542 0.325469616 0.51100975
## LON     0.28360091 0.28584885 0.08569838 0.049851547 0.16959303
## DIS     0.24315562 0.26817765 0.69668199 0.534139751 0.76220420
## CHAS    0.08847498 0.14452838 0.03578039 0.073053205 0.07199604
## .rnorm  0.03550487 0.02668252 0.03323732 0.001100587 0.05833104
## LAT     0.05378927 0.06148204 0.08015774 0.176870069 0.09904524
##                CRIM        AGE         ZN        LON        DIS
## RM      0.235750727 0.26820079 0.33045445 0.28360091 0.24315562
## PTRATIO 0.303769465 0.26522697 0.42268028 0.28584885 0.26817765
## INDUS   0.403870283 0.64676404 0.53829542 0.08569838 0.69668199
## TAX     0.580683778 0.51479716 0.32546962 0.04985155 0.53413975
## NOX     0.410680057 0.72190662 0.51100975 0.16959303 0.76220420
## CRIM    0.000000000 0.35315440 0.19902174 0.07799185 0.37959213
## AGE     0.353154400 0.00000000 0.57047282 0.23164927 0.73563604
## ZN      0.199021738 0.57047282 0.00000000 0.23489301 0.65732096
## LON     0.077991852 0.23164927 0.23489301 0.00000000 0.03822171
## DIS     0.379592128 0.73563604 0.65732096 0.03822171 0.00000000
## CHAS    0.062760814 0.06568424 0.02878224 0.18878306 0.07254804
## .rnorm  0.004542403 0.06193895 0.01767417 0.07746727 0.02029136
## LAT     0.087829088 0.05045751 0.05195114 0.17124146 0.03324755
##                CHAS      .rnorm        LAT
## RM      0.088474975 0.035504870 0.05378927
## PTRATIO 0.144528380 0.026682516 0.06148204
## INDUS   0.035780390 0.033237322 0.08015774
## TAX     0.073053205 0.001100587 0.17687007
## NOX     0.071996036 0.058331040 0.09904524
## CRIM    0.062760814 0.004542403 0.08782909
## AGE     0.065684238 0.061938951 0.05045751
## ZN      0.028782240 0.017674169 0.05195114
## LON     0.188783063 0.077467273 0.17124146
## DIS     0.072548035 0.020291355 0.03324755
## CHAS    0.000000000 0.004021079 0.05537654
## .rnorm  0.004021079 0.000000000 0.01017963
## LAT     0.055376536 0.010179628 0.00000000
## [1] "cor(NOX, DIS)=-0.7622"
```

![](Boston_Housing_files/figure-html/remove_correlated_features-3.png) 

```
## [1] "cor(MEDV, NOX)=-0.4320"
## [1] "cor(MEDV, DIS)=0.2583"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, entity_df =
## glb_trnent_df, : Identified DIS as highly correlated with other features
```

![](Boston_Housing_files/figure-html/remove_correlated_features-4.png) 

```
## [1] "checking correlations for features:"
##  [1] "RM"      "PTRATIO" "INDUS"   "TAX"     "NOX"     "CRIM"    "AGE"    
##  [8] "ZN"      "LON"     "CHAS"    ".rnorm"  "LAT"    
##                  RM     PTRATIO       INDUS          TAX         NOX
## RM       1.00000000 -0.37984925 -0.43197466 -0.339950064 -0.33472738
## PTRATIO -0.37984925  1.00000000  0.40434610  0.485502357  0.20282393
## INDUS   -0.43197466  0.40434610  1.00000000  0.715219161  0.75758099
## TAX     -0.33995006  0.48550236  0.71521916  1.000000000  0.66979694
## NOX     -0.33472738  0.20282393  0.75758099  0.669796937  1.00000000
## CRIM    -0.23575073  0.30376946  0.40387028  0.580683778  0.41068006
## AGE     -0.26820079  0.26522697  0.64676404  0.514797156  0.72190662
## ZN       0.33045445 -0.42268028 -0.53829542 -0.325469616 -0.51100975
## LON     -0.28360091  0.28584885  0.08569838  0.049851547  0.16959303
## CHAS     0.08847498 -0.14452838  0.03578039 -0.073053205  0.07199604
## .rnorm   0.03550487 -0.02668252  0.03323732 -0.001100587  0.05833104
## LAT     -0.05378927 -0.06148204 -0.08015774 -0.176870069 -0.09904524
##                 CRIM         AGE          ZN         LON         CHAS
## RM      -0.235750727 -0.26820079  0.33045445 -0.28360091  0.088474975
## PTRATIO  0.303769465  0.26522697 -0.42268028  0.28584885 -0.144528380
## INDUS    0.403870283  0.64676404 -0.53829542  0.08569838  0.035780390
## TAX      0.580683778  0.51479716 -0.32546962  0.04985155 -0.073053205
## NOX      0.410680057  0.72190662 -0.51100975  0.16959303  0.071996036
## CRIM     1.000000000  0.35315440 -0.19902174  0.07799185 -0.062760814
## AGE      0.353154400  1.00000000 -0.57047282  0.23164927  0.065684238
## ZN      -0.199021738 -0.57047282  1.00000000 -0.23489301 -0.028782240
## LON      0.077991852  0.23164927 -0.23489301  1.00000000 -0.188783063
## CHAS    -0.062760814  0.06568424 -0.02878224 -0.18878306  1.000000000
## .rnorm   0.004542403  0.06193895 -0.01767417  0.07746727  0.004021079
## LAT     -0.087829088  0.05045751 -0.05195114  0.17124146 -0.055376536
##               .rnorm         LAT
## RM       0.035504870 -0.05378927
## PTRATIO -0.026682516 -0.06148204
## INDUS    0.033237322 -0.08015774
## TAX     -0.001100587 -0.17687007
## NOX      0.058331040 -0.09904524
## CRIM     0.004542403 -0.08782909
## AGE      0.061938951  0.05045751
## ZN      -0.017674169 -0.05195114
## LON      0.077467273  0.17124146
## CHAS     0.004021079 -0.05537654
## .rnorm   1.000000000 -0.01017963
## LAT     -0.010179628  1.00000000
##                 RM    PTRATIO      INDUS         TAX        NOX
## RM      0.00000000 0.37984925 0.43197466 0.339950064 0.33472738
## PTRATIO 0.37984925 0.00000000 0.40434610 0.485502357 0.20282393
## INDUS   0.43197466 0.40434610 0.00000000 0.715219161 0.75758099
## TAX     0.33995006 0.48550236 0.71521916 0.000000000 0.66979694
## NOX     0.33472738 0.20282393 0.75758099 0.669796937 0.00000000
## CRIM    0.23575073 0.30376946 0.40387028 0.580683778 0.41068006
## AGE     0.26820079 0.26522697 0.64676404 0.514797156 0.72190662
## ZN      0.33045445 0.42268028 0.53829542 0.325469616 0.51100975
## LON     0.28360091 0.28584885 0.08569838 0.049851547 0.16959303
## CHAS    0.08847498 0.14452838 0.03578039 0.073053205 0.07199604
## .rnorm  0.03550487 0.02668252 0.03323732 0.001100587 0.05833104
## LAT     0.05378927 0.06148204 0.08015774 0.176870069 0.09904524
##                CRIM        AGE         ZN        LON        CHAS
## RM      0.235750727 0.26820079 0.33045445 0.28360091 0.088474975
## PTRATIO 0.303769465 0.26522697 0.42268028 0.28584885 0.144528380
## INDUS   0.403870283 0.64676404 0.53829542 0.08569838 0.035780390
## TAX     0.580683778 0.51479716 0.32546962 0.04985155 0.073053205
## NOX     0.410680057 0.72190662 0.51100975 0.16959303 0.071996036
## CRIM    0.000000000 0.35315440 0.19902174 0.07799185 0.062760814
## AGE     0.353154400 0.00000000 0.57047282 0.23164927 0.065684238
## ZN      0.199021738 0.57047282 0.00000000 0.23489301 0.028782240
## LON     0.077991852 0.23164927 0.23489301 0.00000000 0.188783063
## CHAS    0.062760814 0.06568424 0.02878224 0.18878306 0.000000000
## .rnorm  0.004542403 0.06193895 0.01767417 0.07746727 0.004021079
## LAT     0.087829088 0.05045751 0.05195114 0.17124146 0.055376536
##              .rnorm        LAT
## RM      0.035504870 0.05378927
## PTRATIO 0.026682516 0.06148204
## INDUS   0.033237322 0.08015774
## TAX     0.001100587 0.17687007
## NOX     0.058331040 0.09904524
## CRIM    0.004542403 0.08782909
## AGE     0.061938951 0.05045751
## ZN      0.017674169 0.05195114
## LON     0.077467273 0.17124146
## CHAS    0.004021079 0.05537654
## .rnorm  0.000000000 0.01017963
## LAT     0.010179628 0.00000000
## [1] "cor(INDUS, NOX)=0.7576"
```

![](Boston_Housing_files/figure-html/remove_correlated_features-5.png) 

```
## [1] "cor(MEDV, INDUS)=-0.4940"
## [1] "cor(MEDV, NOX)=-0.4320"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, entity_df =
## glb_trnent_df, : Identified NOX as highly correlated with other features
```

![](Boston_Housing_files/figure-html/remove_correlated_features-6.png) 

```
## [1] "checking correlations for features:"
##  [1] "RM"      "PTRATIO" "INDUS"   "TAX"     "CRIM"    "AGE"     "ZN"     
##  [8] "LON"     "CHAS"    ".rnorm"  "LAT"    
##                  RM     PTRATIO       INDUS          TAX         CRIM
## RM       1.00000000 -0.37984925 -0.43197466 -0.339950064 -0.235750727
## PTRATIO -0.37984925  1.00000000  0.40434610  0.485502357  0.303769465
## INDUS   -0.43197466  0.40434610  1.00000000  0.715219161  0.403870283
## TAX     -0.33995006  0.48550236  0.71521916  1.000000000  0.580683778
## CRIM    -0.23575073  0.30376946  0.40387028  0.580683778  1.000000000
## AGE     -0.26820079  0.26522697  0.64676404  0.514797156  0.353154400
## ZN       0.33045445 -0.42268028 -0.53829542 -0.325469616 -0.199021738
## LON     -0.28360091  0.28584885  0.08569838  0.049851547  0.077991852
## CHAS     0.08847498 -0.14452838  0.03578039 -0.073053205 -0.062760814
## .rnorm   0.03550487 -0.02668252  0.03323732 -0.001100587  0.004542403
## LAT     -0.05378927 -0.06148204 -0.08015774 -0.176870069 -0.087829088
##                 AGE          ZN         LON         CHAS       .rnorm
## RM      -0.26820079  0.33045445 -0.28360091  0.088474975  0.035504870
## PTRATIO  0.26522697 -0.42268028  0.28584885 -0.144528380 -0.026682516
## INDUS    0.64676404 -0.53829542  0.08569838  0.035780390  0.033237322
## TAX      0.51479716 -0.32546962  0.04985155 -0.073053205 -0.001100587
## CRIM     0.35315440 -0.19902174  0.07799185 -0.062760814  0.004542403
## AGE      1.00000000 -0.57047282  0.23164927  0.065684238  0.061938951
## ZN      -0.57047282  1.00000000 -0.23489301 -0.028782240 -0.017674169
## LON      0.23164927 -0.23489301  1.00000000 -0.188783063  0.077467273
## CHAS     0.06568424 -0.02878224 -0.18878306  1.000000000  0.004021079
## .rnorm   0.06193895 -0.01767417  0.07746727  0.004021079  1.000000000
## LAT      0.05045751 -0.05195114  0.17124146 -0.055376536 -0.010179628
##                 LAT
## RM      -0.05378927
## PTRATIO -0.06148204
## INDUS   -0.08015774
## TAX     -0.17687007
## CRIM    -0.08782909
## AGE      0.05045751
## ZN      -0.05195114
## LON      0.17124146
## CHAS    -0.05537654
## .rnorm  -0.01017963
## LAT      1.00000000
##                 RM    PTRATIO      INDUS         TAX        CRIM
## RM      0.00000000 0.37984925 0.43197466 0.339950064 0.235750727
## PTRATIO 0.37984925 0.00000000 0.40434610 0.485502357 0.303769465
## INDUS   0.43197466 0.40434610 0.00000000 0.715219161 0.403870283
## TAX     0.33995006 0.48550236 0.71521916 0.000000000 0.580683778
## CRIM    0.23575073 0.30376946 0.40387028 0.580683778 0.000000000
## AGE     0.26820079 0.26522697 0.64676404 0.514797156 0.353154400
## ZN      0.33045445 0.42268028 0.53829542 0.325469616 0.199021738
## LON     0.28360091 0.28584885 0.08569838 0.049851547 0.077991852
## CHAS    0.08847498 0.14452838 0.03578039 0.073053205 0.062760814
## .rnorm  0.03550487 0.02668252 0.03323732 0.001100587 0.004542403
## LAT     0.05378927 0.06148204 0.08015774 0.176870069 0.087829088
##                AGE         ZN        LON        CHAS      .rnorm
## RM      0.26820079 0.33045445 0.28360091 0.088474975 0.035504870
## PTRATIO 0.26522697 0.42268028 0.28584885 0.144528380 0.026682516
## INDUS   0.64676404 0.53829542 0.08569838 0.035780390 0.033237322
## TAX     0.51479716 0.32546962 0.04985155 0.073053205 0.001100587
## CRIM    0.35315440 0.19902174 0.07799185 0.062760814 0.004542403
## AGE     0.00000000 0.57047282 0.23164927 0.065684238 0.061938951
## ZN      0.57047282 0.00000000 0.23489301 0.028782240 0.017674169
## LON     0.23164927 0.23489301 0.00000000 0.188783063 0.077467273
## CHAS    0.06568424 0.02878224 0.18878306 0.000000000 0.004021079
## .rnorm  0.06193895 0.01767417 0.07746727 0.004021079 0.000000000
## LAT     0.05045751 0.05195114 0.17124146 0.055376536 0.010179628
##                LAT
## RM      0.05378927
## PTRATIO 0.06148204
## INDUS   0.08015774
## TAX     0.17687007
## CRIM    0.08782909
## AGE     0.05045751
## ZN      0.05195114
## LON     0.17124146
## CHAS    0.05537654
## .rnorm  0.01017963
## LAT     0.00000000
## [1] "cor(INDUS, TAX)=0.7152"
```

![](Boston_Housing_files/figure-html/remove_correlated_features-7.png) 

```
## [1] "cor(MEDV, INDUS)=-0.4940"
## [1] "cor(MEDV, TAX)=-0.4794"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, entity_df =
## glb_trnent_df, : Identified TAX as highly correlated with other features
```

![](Boston_Housing_files/figure-html/remove_correlated_features-8.png) 

```
## [1] "checking correlations for features:"
##  [1] "RM"      "PTRATIO" "INDUS"   "CRIM"    "AGE"     "ZN"      "LON"    
##  [8] "CHAS"    ".rnorm"  "LAT"    
##                  RM     PTRATIO       INDUS         CRIM         AGE
## RM       1.00000000 -0.37984925 -0.43197466 -0.235750727 -0.26820079
## PTRATIO -0.37984925  1.00000000  0.40434610  0.303769465  0.26522697
## INDUS   -0.43197466  0.40434610  1.00000000  0.403870283  0.64676404
## CRIM    -0.23575073  0.30376946  0.40387028  1.000000000  0.35315440
## AGE     -0.26820079  0.26522697  0.64676404  0.353154400  1.00000000
## ZN       0.33045445 -0.42268028 -0.53829542 -0.199021738 -0.57047282
## LON     -0.28360091  0.28584885  0.08569838  0.077991852  0.23164927
## CHAS     0.08847498 -0.14452838  0.03578039 -0.062760814  0.06568424
## .rnorm   0.03550487 -0.02668252  0.03323732  0.004542403  0.06193895
## LAT     -0.05378927 -0.06148204 -0.08015774 -0.087829088  0.05045751
##                  ZN         LON         CHAS       .rnorm         LAT
## RM       0.33045445 -0.28360091  0.088474975  0.035504870 -0.05378927
## PTRATIO -0.42268028  0.28584885 -0.144528380 -0.026682516 -0.06148204
## INDUS   -0.53829542  0.08569838  0.035780390  0.033237322 -0.08015774
## CRIM    -0.19902174  0.07799185 -0.062760814  0.004542403 -0.08782909
## AGE     -0.57047282  0.23164927  0.065684238  0.061938951  0.05045751
## ZN       1.00000000 -0.23489301 -0.028782240 -0.017674169 -0.05195114
## LON     -0.23489301  1.00000000 -0.188783063  0.077467273  0.17124146
## CHAS    -0.02878224 -0.18878306  1.000000000  0.004021079 -0.05537654
## .rnorm  -0.01767417  0.07746727  0.004021079  1.000000000 -0.01017963
## LAT     -0.05195114  0.17124146 -0.055376536 -0.010179628  1.00000000
##                 RM    PTRATIO      INDUS        CRIM        AGE         ZN
## RM      0.00000000 0.37984925 0.43197466 0.235750727 0.26820079 0.33045445
## PTRATIO 0.37984925 0.00000000 0.40434610 0.303769465 0.26522697 0.42268028
## INDUS   0.43197466 0.40434610 0.00000000 0.403870283 0.64676404 0.53829542
## CRIM    0.23575073 0.30376946 0.40387028 0.000000000 0.35315440 0.19902174
## AGE     0.26820079 0.26522697 0.64676404 0.353154400 0.00000000 0.57047282
## ZN      0.33045445 0.42268028 0.53829542 0.199021738 0.57047282 0.00000000
## LON     0.28360091 0.28584885 0.08569838 0.077991852 0.23164927 0.23489301
## CHAS    0.08847498 0.14452838 0.03578039 0.062760814 0.06568424 0.02878224
## .rnorm  0.03550487 0.02668252 0.03323732 0.004542403 0.06193895 0.01767417
## LAT     0.05378927 0.06148204 0.08015774 0.087829088 0.05045751 0.05195114
##                LON        CHAS      .rnorm        LAT
## RM      0.28360091 0.088474975 0.035504870 0.05378927
## PTRATIO 0.28584885 0.144528380 0.026682516 0.06148204
## INDUS   0.08569838 0.035780390 0.033237322 0.08015774
## CRIM    0.07799185 0.062760814 0.004542403 0.08782909
## AGE     0.23164927 0.065684238 0.061938951 0.05045751
## ZN      0.23489301 0.028782240 0.017674169 0.05195114
## LON     0.00000000 0.188783063 0.077467273 0.17124146
## CHAS    0.18878306 0.000000000 0.004021079 0.05537654
## .rnorm  0.07746727 0.004021079 0.000000000 0.01017963
## LAT     0.17124146 0.055376536 0.010179628 0.00000000
##                  id       cor.y exclude.as.feat  cor.y.abs cor.low
## RM               RM  0.70872979               0 0.70872979       1
## TRACT         TRACT  0.44111242               1 0.44111242       0
## ZN               ZN  0.36446804               0 0.36446804       1
## DIS             DIS  0.25834984               0 0.25834984       0
## CHAS           CHAS  0.16496009               0 0.16496009       1
## .rnorm       .rnorm  0.02567817               0 0.02567817       1
## LAT             LAT  0.01854813               0 0.01854813       1
## TOWN.fctr TOWN.fctr -0.28659819               1 0.28659819       0
## LON             LON -0.33341021               0 0.33341021       1
## AGE             AGE -0.37857924               0 0.37857924       1
## CRIM           CRIM -0.38466671               0 0.38466671       1
## RAD             RAD -0.40830448               0 0.40830448       0
## NOX             NOX -0.43202955               0 0.43202955       0
## TAX             TAX -0.47940636               0 0.47940636       0
## INDUS         INDUS -0.49397282               0 0.49397282       1
## PTRATIO     PTRATIO -0.50539619               0 0.50539619       1
```

```r
glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="fit.models", 
                              chunk_step_major=max(glb_script_df$chunk_step_major)+1, 
                              chunk_step_minor=0,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                         chunk_label chunk_step_major chunk_step_minor
## elapsed7 remove_correlated_features                4                1
## elapsed8                 fit.models                5                0
##          elapsed
## elapsed7   3.983
## elapsed8   7.029
```

## Step `5`: fit models

```r
max_cor_y_x_var <- subset(glb_feats_df, cor.low == 1)[1, "id"]
if (!is.null(glb_Baseline_mdl_var)) {
    if ((max_cor_y_x_var != glb_Baseline_mdl_var) & 
        (glb_feats_df[max_cor_y_x_var, "cor.y.abs"] > 
         glb_feats_df[glb_Baseline_mdl_var, "cor.y.abs"]))
        stop(max_cor_y_x_var, " has a lower correlation with ", glb_rsp_var, 
             " than the Baseline var: ", glb_Baseline_mdl_var)
}

glb_model_type <- ifelse(glb_is_regression, "regression", "classification")
#   Identify Binomial Classification:
if (glb_is_classification)
    glb_is_binomial <- (length(unique(glb_trnent_df[, glb_rsp_var])) <= 2)
    
# Any models that have tuning parameters has "better" results with cross-validation (except rf)
#   & "different" results for different outcome metrics

# Baseline
if (!is.null(glb_Baseline_mdl_var)) {
#     lm_mdl <- lm(reformulate(glb_Baseline_mdl_var, 
#                             response="bucket2009"), data=glb_trnent_df)
#     print(summary(lm_mdl))
#     plot(lm_mdl, ask=FALSE)
#     ret_lst <- myfit_mdl_fn(model_id="Baseline", 
#                             model_method=ifelse(glb_is_regression, "lm", 
#                                         ifelse(glb_is_binomial, "glm", "rpart")),
#                             indep_vars_vctr=glb_Baseline_mdl_var,
#                             rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
#                             fit_df=glb_trnent_df, OOB_df=glb_newent_df,
#                             n_cv_folds=0, tune_models_df=NULL,
#                             model_loss_mtrx=glb_model_metric_terms,
#                             model_summaryFunction=glb_model_metric_smmry,
#                             model_metric=glb_model_metric,
#                             model_metric_maximize=glb_model_metric_maximize)
    ret_lst <- myfit_mdl_fn(model_id="Baseline", model_method="mybaseln_classfr",
                            indep_vars_vctr=glb_Baseline_mdl_var,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_trnent_df, OOB_df=glb_newent_df)
}

# Most Frequent Outcome "MFO" model: mean(y) for regression
#   Not using caret's nullModel since model stats not avl
#   Cannot use rpart for multinomial classification since it predicts non-MFO
ret_lst <- myfit_mdl(model_id="MFO", 
                     model_method=ifelse(glb_is_regression, "lm", "myMFO_classfr"), 
                     model_type=glb_model_type,
                        indep_vars_vctr=".rnorm",
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_trnent_df, OOB_df=glb_newent_df)
```

```
## Loading required package: caret
## Loading required package: lattice
## 
## Attaching package: 'caret'
## 
## The following object is masked from 'package:survival':
## 
##     cluster
```

```
## [1] "fitting model: MFO.lm"
## [1] "    indep_vars: .rnorm"
## Fitting parameter = none on full training set
```

![](Boston_Housing_files/figure-html/fit.models-1.png) ![](Boston_Housing_files/figure-html/fit.models-2.png) ![](Boston_Housing_files/figure-html/fit.models-3.png) ![](Boston_Housing_files/figure-html/fit.models-4.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -17.934  -5.753  -1.603   3.502  27.574 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  22.9211     0.4987  45.958   <2e-16 ***
## .rnorm        0.2346     0.4800   0.489    0.625    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 9.502 on 362 degrees of freedom
## Multiple R-squared:  0.0006594,	Adjusted R-squared:  -0.002101 
## F-statistic: 0.2388 on 1 and 362 DF,  p-value: 0.6253
## 
##   model_id model_method  feats max.nTuningRuns min.elapsedtime.everything
## 1   MFO.lm           lm .rnorm               0                      0.605
##   min.elapsedtime.final max.R.sq.fit min.RMSE.fit max.R.sq.OOB
## 1                 0.004 0.0006593684     9.475775 -0.002734968
##   min.RMSE.OOB max.Adj.R.sq.fit
## 1     8.384599     -0.002101241
```

```r
if (glb_is_classification)
    # "random" model - only for classification; none needed for regression since it is same as MFO
    ret_lst <- myfit_mdl_fn(model_id="Random", model_method="myrandom_classfr",
                            indep_vars_vctr=".rnorm",
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_trnent_df, OOB_df=glb_newent_df)

# latlon.lm similar to recitation - video 3
ret_lst <- myfit_mdl(model_id="latlon", 
                     model_method="lm", 
                     model_type=glb_model_type,
                        indep_vars_vctr=c("LAT", "LON"),
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_entity_df, OOB_df=NULL)
```

```
## [1] "fitting model: latlon.lm"
## [1] "    indep_vars: LAT, LON"
## Fitting parameter = none on full training set
```

![](Boston_Housing_files/figure-html/fit.models-5.png) ![](Boston_Housing_files/figure-html/fit.models-6.png) ![](Boston_Housing_files/figure-html/fit.models-7.png) ![](Boston_Housing_files/figure-html/fit.models-8.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -16.460  -5.590  -1.299   3.695  28.129 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -3178.472    484.937  -6.554 1.39e-10 ***
## LAT             8.046      6.327   1.272    0.204    
## LON           -40.268      5.184  -7.768 4.50e-14 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 8.693 on 503 degrees of freedom
## Multiple R-squared:  0.1072,	Adjusted R-squared:  0.1036 
## F-statistic: 30.19 on 2 and 503 DF,  p-value: 4.159e-13
## 
##    model_id model_method    feats max.nTuningRuns
## 1 latlon.lm           lm LAT, LON               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.414                 0.003    0.1071649
##   min.RMSE.fit max.Adj.R.sq.fit
## 1     8.667656        0.1036149
```

```r
# latlon.rpart similar to recitation - video 4
tmp_tune_models_df <- 
   rbind(
    data.frame(parameter="cp", min=0.01, max=0.01, by=0.01),
                          #seq(from=0.01,  to=0.01, by=0.01)
    #data.frame(parameter="mtry", min=2, max=4, by=1),
    data.frame(parameter="dummy", min=2, max=4, by=1)
        ) 
ret_lst <- myfit_mdl(model_id="latlon", 
                     model_method="rpart", 
                     model_type=glb_model_type,
                        indep_vars_vctr=c("LAT", "LON"),
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_entity_df, OOB_df=NULL,
                    n_cv_folds=0, tune_models_df=tmp_tune_models_df)
```

```
## [1] "fitting model: latlon.rpart"
## [1] "    indep_vars: LAT, LON"
```

```
## Loading required package: rpart
```

```
## Fitting cp = 0.01 on full training set
```

```
## Loading required package: rpart.plot
```

![](Boston_Housing_files/figure-html/fit.models-9.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 506 
## 
##           CP nsplit rel error
## 1 0.25936559      0 1.0000000
## 2 0.03614203      1 0.7406344
## 3 0.02403175      5 0.5960663
## 4 0.01623213      8 0.5230710
## 5 0.01611549      9 0.5068389
## 6 0.01486293     11 0.4746079
## 7 0.01313577     12 0.4597450
## 8 0.01014972     13 0.4466092
## 9 0.01000000     14 0.4364595
## 
## Variable importance
## LON LAT 
##  58  42 
## 
## Node number 1: 506 observations,    complexity param=0.2593656
##   mean=22.52885, MSE=84.14573 
##   left son=2 (309 obs) right son=3 (197 obs)
##   Primary splits:
##       LON < -71.0678  to the right, improve=0.259365600, (0 missing)
##       LAT < 42.2019   to the left,  improve=0.008323488, (0 missing)
## 
## Node number 2: 309 observations,    complexity param=0.03614203
##   mean=18.79871, MSE=50.90764 
##   left son=4 (252 obs) right son=5 (57 obs)
##   Primary splits:
##       LAT < 42.28275  to the left,  improve=0.07809841, (0 missing)
##       LON < -70.93925 to the left,  improve=0.06938084, (0 missing)
##   Surrogate splits:
##       LON < -70.96475 to the left,  agree=0.845, adj=0.158, (0 split)
## 
## Node number 3: 197 observations,    complexity param=0.02403175
##   mean=28.3797, MSE=80.22375 
##   left son=6 (46 obs) right son=7 (151 obs)
##   Primary splits:
##       LAT < 42.1726   to the left,  improve=0.03881385, (0 missing)
##       LON < -71.0981  to the right, improve=0.02469454, (0 missing)
##   Surrogate splits:
##       LON < -71.26875 to the left,  agree=0.772, adj=0.022, (0 split)
## 
## Node number 4: 252 observations,    complexity param=0.03614203
##   mean=17.8504, MSE=50.98853 
##   left son=8 (197 obs) right son=9 (55 obs)
##   Primary splits:
##       LAT < 42.168    to the right, improve=0.10903040, (0 missing)
##       LON < -71.0155  to the left,  improve=0.04948701, (0 missing)
##   Surrogate splits:
##       LON < -70.95055 to the left,  agree=0.841, adj=0.273, (0 split)
## 
## Node number 5: 57 observations
##   mean=22.99123, MSE=28.99694 
## 
## Node number 6: 46 observations
##   mean=25.18261, MSE=44.06839 
## 
## Node number 7: 151 observations,    complexity param=0.02403175
##   mean=29.35364, MSE=87.1756 
##   left son=14 (104 obs) right son=15 (47 obs)
##   Primary splits:
##       LAT < 42.20785  to the right, improve=0.10886290, (0 missing)
##       LON < -71.16465 to the right, improve=0.04323153, (0 missing)
##   Surrogate splits:
##       LON < -71.16465 to the right, agree=0.728, adj=0.128, (0 split)
## 
## Node number 8: 197 observations,    complexity param=0.03614203
##   mean=16.60457, MSE=52.88389 
##   left son=16 (76 obs) right son=17 (121 obs)
##   Primary splits:
##       LAT < 42.20585  to the left,  improve=0.12846190, (0 missing)
##       LON < -71.01425 to the left,  improve=0.01561267, (0 missing)
##   Surrogate splits:
##       LON < -71.0429  to the left,  agree=0.629, adj=0.039, (0 split)
## 
## Node number 9: 55 observations
##   mean=22.31273, MSE=18.72802 
## 
## Node number 14: 104 observations,    complexity param=0.02403175
##   mean=27.28269, MSE=74.14085 
##   left son=28 (96 obs) right son=29 (8 obs)
##   Primary splits:
##       LON < -71.17615 to the right, improve=0.13767180, (0 missing)
##       LAT < 42.21785  to the left,  improve=0.05389777, (0 missing)
## 
## Node number 15: 47 observations,    complexity param=0.01623213
##   mean=33.93617, MSE=85.52869 
##   left son=30 (10 obs) right son=31 (37 obs)
##   Primary splits:
##       LON < -71.20125 to the left,  improve=0.17192870, (0 missing)
##       LAT < 42.20365  to the left,  improve=0.09849123, (0 missing)
## 
## Node number 16: 76 observations
##   mean=13.31579, MSE=16.01212 
## 
## Node number 17: 121 observations,    complexity param=0.03614203
##   mean=18.67025, MSE=64.98242 
##   left son=34 (106 obs) right son=35 (15 obs)
##   Primary splits:
##       LAT < 42.21845  to the right, improve=0.27821690, (0 missing)
##       LON < -71.03965 to the right, improve=0.07741201, (0 missing)
## 
## Node number 28: 96 observations,    complexity param=0.01611549
##   mean=26.36042, MSE=65.09156 
##   left son=56 (22 obs) right son=57 (74 obs)
##   Primary splits:
##       LAT < 42.22325  to the left,  improve=0.06089301, (0 missing)
##       LON < -71.09575 to the right, improve=0.02541116, (0 missing)
## 
## Node number 29: 8 observations
##   mean=38.35, MSE=50.04 
## 
## Node number 30: 10 observations
##   mean=26.56, MSE=32.8984 
## 
## Node number 31: 37 observations
##   mean=35.92973, MSE=81.07398 
## 
## Node number 34: 106 observations,    complexity param=0.01313577
##   mean=17.07075, MSE=20.7332 
##   left son=68 (54 obs) right son=69 (52 obs)
##   Primary splits:
##       LAT < 42.241    to the left,  improve=0.25448710, (0 missing)
##       LON < -71.01425 to the left,  improve=0.04378652, (0 missing)
##   Surrogate splits:
##       LON < -71.01425 to the left,  agree=0.67, adj=0.327, (0 split)
## 
## Node number 35: 15 observations
##   mean=29.97333, MSE=231.838 
## 
## Node number 56: 22 observations
##   mean=22.70909, MSE=13.94264 
## 
## Node number 57: 74 observations,    complexity param=0.01611549
##   mean=27.44595, MSE=75.156 
##   left son=114 (59 obs) right son=115 (15 obs)
##   Primary splits:
##       LAT < 42.23065  to the right, improve=0.17833440, (0 missing)
##       LON < -71.0929  to the right, improve=0.02950775, (0 missing)
## 
## Node number 68: 54 observations
##   mean=14.81667, MSE=19.16065 
## 
## Node number 69: 52 observations
##   mean=19.41154, MSE=11.61064 
## 
## Node number 114: 59 observations,    complexity param=0.01486293
##   mean=25.6, MSE=48.05559 
##   left son=228 (30 obs) right son=229 (29 obs)
##   Primary splits:
##       LON < -71.0925  to the right, improve=0.22319840, (0 missing)
##       LAT < 42.281    to the right, improve=0.06797711, (0 missing)
##   Surrogate splits:
##       LAT < 42.2501   to the left,  agree=0.627, adj=0.241, (0 split)
## 
## Node number 115: 15 observations
##   mean=34.70667, MSE=115.63 
## 
## Node number 228: 30 observations
##   mean=22.38, MSE=22.11227 
## 
## Node number 229: 29 observations,    complexity param=0.01014972
##   mean=28.93103, MSE=53.0718 
##   left son=458 (10 obs) right son=459 (19 obs)
##   Primary splits:
##       LAT < 42.283    to the right, improve=0.28078560, (0 missing)
##       LON < -71.107   to the left,  improve=0.06804633, (0 missing)
##   Surrogate splits:
##       LON < -71.15125 to the left,  agree=0.724, adj=0.2, (0 split)
## 
## Node number 458: 10 observations
##   mean=23.61, MSE=14.1969 
## 
## Node number 459: 19 observations
##   mean=31.73158, MSE=50.78742 
## 
## n= 506 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
##   1) root 506 42577.7400 22.52885  
##     2) LON>=-71.0678 309 15730.4600 18.79871  
##       4) LAT< 42.28275 252 12849.1100 17.85040  
##         8) LAT>=42.168 197 10418.1300 16.60457  
##          16) LAT< 42.20585 76  1216.9210 13.31579 *
##          17) LAT>=42.20585 121  7862.8730 18.67025  
##            34) LAT>=42.21845 106  2197.7190 17.07075  
##              68) LAT< 42.241 54  1034.6750 14.81667 *
##              69) LAT>=42.241 52   603.7531 19.41154 *
##            35) LAT< 42.21845 15  3477.5690 29.97333 *
##         9) LAT< 42.168 55  1030.0410 22.31273 *
##       5) LAT>=42.28275 57  1652.8260 22.99123 *
##     3) LON< -71.0678 197 15804.0800 28.37970  
##       6) LAT< 42.1726 46  2027.1460 25.18261 *
##       7) LAT>=42.1726 151 13163.5200 29.35364  
##        14) LAT>=42.20785 104  7710.6490 27.28269  
##          28) LON>=-71.17615 96  6248.7900 26.36042  
##            56) LAT< 42.22325 22   306.7382 22.70909 *
##            57) LAT>=42.22325 74  5561.5440 27.44595  
##             114) LAT>=42.23065 59  2835.2800 25.60000  
##               228) LON>=-71.0925 30   663.3680 22.38000 *
##               229) LON< -71.0925 29  1539.0820 28.93103  
##                 458) LAT>=42.283 10   141.9690 23.61000 *
##                 459) LAT< 42.283 19   964.9611 31.73158 *
##             115) LAT< 42.23065 15  1734.4490 34.70667 *
##          29) LON< -71.17615 8   400.3200 38.35000 *
##        15) LAT< 42.20785 47  4019.8490 33.93617  
##          30) LON< -71.20125 10   328.9840 26.56000 *
##          31) LON>=-71.20125 37  2999.7370 35.92973 *
##       model_id model_method    feats max.nTuningRuns
## 1 latlon.rpart        rpart LAT, LON               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.533                 0.018    0.5635405
##   min.RMSE.fit
## 1     6.060215
```

```r
# Max.cor.Y
ret_lst <- myfit_mdl(model_id="Max.cor.Y.cv.0", 
                        model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                     model_type=glb_model_type,
                        indep_vars_vctr=max_cor_y_x_var,
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_trnent_df, OOB_df=glb_newent_df)
```

```
## [1] "fitting model: Max.cor.Y.cv.0.lm"
## [1] "    indep_vars: RM"
## Fitting parameter = none on full training set
```

![](Boston_Housing_files/figure-html/fit.models-10.png) ![](Boston_Housing_files/figure-html/fit.models-11.png) ![](Boston_Housing_files/figure-html/fit.models-12.png) ![](Boston_Housing_files/figure-html/fit.models-13.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -20.011  -2.590  -0.129   2.987  39.138 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -34.8034     3.0411  -11.44   <2e-16 ***
## RM            9.1881     0.4807   19.11   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 6.706 on 362 degrees of freedom
## Multiple R-squared:  0.5023,	Adjusted R-squared:  0.5009 
## F-statistic: 365.3 on 1 and 362 DF,  p-value: < 2.2e-16
## 
##            model_id model_method feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.lm           lm    RM               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.431                 0.002    0.5022979
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1     6.687175    0.4229655     6.360483         0.500923
```

```r
ret_lst <- myfit_mdl(model_id="Max.cor.Y.cv.G", 
                        model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                     model_type=glb_model_type,
                        indep_vars_vctr=max_cor_y_x_var,
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_trnent_df, OOB_df=glb_newent_df,
                        n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
```

```
## [1] "fitting model: Max.cor.Y.cv.G.lm"
## [1] "    indep_vars: RM"
## + Fold01: parameter=none 
## - Fold01: parameter=none 
## + Fold02: parameter=none 
## - Fold02: parameter=none 
## + Fold03: parameter=none 
## - Fold03: parameter=none 
## + Fold04: parameter=none 
## - Fold04: parameter=none 
## + Fold05: parameter=none 
## - Fold05: parameter=none 
## + Fold06: parameter=none 
## - Fold06: parameter=none 
## + Fold07: parameter=none 
## - Fold07: parameter=none 
## + Fold08: parameter=none 
## - Fold08: parameter=none 
## + Fold09: parameter=none 
## - Fold09: parameter=none 
## + Fold10: parameter=none 
## - Fold10: parameter=none 
## Aggregating results
## Fitting final model on full training set
```

![](Boston_Housing_files/figure-html/fit.models-14.png) ![](Boston_Housing_files/figure-html/fit.models-15.png) ![](Boston_Housing_files/figure-html/fit.models-16.png) ![](Boston_Housing_files/figure-html/fit.models-17.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -20.011  -2.590  -0.129   2.987  39.138 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -34.8034     3.0411  -11.44   <2e-16 ***
## RM            9.1881     0.4807   19.11   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 6.706 on 362 degrees of freedom
## Multiple R-squared:  0.5023,	Adjusted R-squared:  0.5009 
## F-statistic: 365.3 on 1 and 362 DF,  p-value: < 2.2e-16
## 
##            model_id model_method feats max.nTuningRuns
## 1 Max.cor.Y.cv.G.lm           lm    RM               1
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.805                 0.003    0.5022979
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit max.Rsquared.fit
## 1     6.625483    0.4229655     6.360483         0.500923        0.5335055
##   min.RMSESD.fit max.RsquaredSD.fit
## 1        1.33581          0.1836241
```

```r
# Interactions.High.cor.Y
if (nrow(int_feats_df <- subset(glb_feats_df, (cor.low == 0) & 
                                              (exclude.as.feat == 0))) > 0) {
    # lm & glm handle interaction terms; rpart & rf do not
    #   This does not work - why ???
#     indep_vars_vctr <- ifelse(glb_is_binomial, 
#         c(max_cor_y_x_var, paste(max_cor_y_x_var, 
#                         subset(glb_feats_df, is.na(cor.low))[, "id"], sep=":")),
#         union(max_cor_y_x_var, subset(glb_feats_df, is.na(cor.low))[, "id"]))
    if (glb_is_regression || glb_is_binomial) {
        indep_vars_vctr <- 
            c(max_cor_y_x_var, paste(max_cor_y_x_var, int_feats_df[, "id"], sep=":"))       
    } else { indep_vars_vctr <- union(max_cor_y_x_var, int_feats_df[, "id"]) }
    
    ret_lst <- myfit_mdl(model_id="Interact.High.cor.y", 
                            model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                         model_type=glb_model_type,
                            indep_vars_vctr,
                            glb_rsp_var, glb_rsp_var_out,
                            fit_df=glb_trnent_df, OOB_df=glb_newent_df,
                            n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)                        
}    
```

```
## [1] "fitting model: Interact.High.cor.y.lm"
## [1] "    indep_vars: RM, RM:DIS, RM:RAD, RM:NOX, RM:TAX"
## + Fold01: parameter=none 
## - Fold01: parameter=none 
## + Fold02: parameter=none 
## - Fold02: parameter=none 
## + Fold03: parameter=none 
## - Fold03: parameter=none 
## + Fold04: parameter=none 
## - Fold04: parameter=none 
## + Fold05: parameter=none 
## - Fold05: parameter=none 
## + Fold06: parameter=none 
## - Fold06: parameter=none 
## + Fold07: parameter=none 
## - Fold07: parameter=none 
## + Fold08: parameter=none 
## - Fold08: parameter=none 
## + Fold09: parameter=none 
## - Fold09: parameter=none 
## + Fold10: parameter=none 
## - Fold10: parameter=none 
## Aggregating results
## Fitting final model on full training set
```

![](Boston_Housing_files/figure-html/fit.models-18.png) ![](Boston_Housing_files/figure-html/fit.models-19.png) ![](Boston_Housing_files/figure-html/fit.models-20.png) ![](Boston_Housing_files/figure-html/fit.models-21.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -14.217  -3.233  -0.840   2.164  39.851 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.499e+01  2.951e+00  -8.468 6.51e-16 ***
## RM           1.049e+01  6.161e-01  17.029  < 2e-16 ***
## `RM:DIS`    -1.129e-01  3.664e-02  -3.082 0.002217 ** 
## `RM:RAD`     3.976e-03  1.528e-02   0.260 0.794830    
## `RM:NOX`    -2.668e+00  7.919e-01  -3.369 0.000836 ***
## `RM:TAX`    -2.528e-03  8.284e-04  -3.052 0.002444 ** 
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 6.022 on 358 degrees of freedom
## Multiple R-squared:  0.603,	Adjusted R-squared:  0.5974 
## F-statistic: 108.7 on 5 and 358 DF,  p-value: < 2.2e-16
## 
##                 model_id model_method                              feats
## 1 Interact.High.cor.y.lm           lm RM, RM:DIS, RM:RAD, RM:NOX, RM:TAX
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.981                 0.004
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1     0.602982     5.868191    0.5910614     5.354492        0.5974371
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.6342187       1.804358          0.2221433
```

```r
# Low.cor.X
ret_lst <- myfit_mdl(model_id="Low.cor.X", 
                        model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                        indep_vars_vctr=subset(glb_feats_df, cor.low == 1)[, "id"],
                         model_type=glb_model_type,                     
                        glb_rsp_var, glb_rsp_var_out,
                        fit_df=glb_trnent_df, OOB_df=glb_newent_df,
                        n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
```

```
## [1] "fitting model: Low.cor.X.lm"
## [1] "    indep_vars: RM, ZN, CHAS, .rnorm, LAT, LON, AGE, CRIM, INDUS, PTRATIO"
## + Fold01: parameter=none 
## - Fold01: parameter=none 
## + Fold02: parameter=none 
## - Fold02: parameter=none 
## + Fold03: parameter=none 
## - Fold03: parameter=none 
## + Fold04: parameter=none 
## - Fold04: parameter=none 
## + Fold05: parameter=none 
## - Fold05: parameter=none 
## + Fold06: parameter=none 
## - Fold06: parameter=none 
## + Fold07: parameter=none 
## - Fold07: parameter=none 
## + Fold08: parameter=none 
## - Fold08: parameter=none 
## + Fold09: parameter=none 
## - Fold09: parameter=none 
## + Fold10: parameter=none 
## - Fold10: parameter=none 
## Aggregating results
## Fitting final model on full training set
```

![](Boston_Housing_files/figure-html/fit.models-22.png) ![](Boston_Housing_files/figure-html/fit.models-23.png) ![](Boston_Housing_files/figure-html/fit.models-24.png) ![](Boston_Housing_files/figure-html/fit.models-25.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -14.495  -3.128  -0.658   1.849  39.800 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -1.143e+03  4.256e+02  -2.686 0.007574 ** 
## RM           6.786e+00  5.015e-01  13.532  < 2e-16 ***
## ZN          -1.061e-02  1.741e-02  -0.610 0.542393    
## CHAS         2.901e+00  1.269e+00   2.287 0.022816 *  
## .rnorm       1.700e-01  3.002e-01   0.566 0.571616    
## LAT          6.337e+00  5.232e+00   1.211 0.226598    
## LON         -1.230e+01  4.712e+00  -2.610 0.009446 ** 
## AGE         -2.936e-02  1.651e-02  -1.778 0.076257 .  
## CRIM        -1.406e-01  4.049e-02  -3.474 0.000577 ***
## INDUS       -1.331e-01  6.936e-02  -1.919 0.055740 .  
## PTRATIO     -7.671e-01  1.751e-01  -4.382 1.55e-05 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 5.891 on 353 degrees of freedom
## Multiple R-squared:  0.6254,	Adjusted R-squared:  0.6148 
## F-statistic: 58.93 on 10 and 353 DF,  p-value: < 2.2e-16
## 
##       model_id model_method
## 1 Low.cor.X.lm           lm
##                                                       feats
## 1 RM, ZN, CHAS, .rnorm, LAT, LON, AGE, CRIM, INDUS, PTRATIO
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.793                 0.004
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.6253947     5.793811    0.6489184     4.961275        0.6147826
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.6393438       1.602839          0.2061768
```

```r
# All.X.cv.0.lm similar to recitation - video 5
model_id_pfx <- "All.X";
    ret_lst <- myfit_mdl(model_id=paste0(model_id_pfx, ".cv.0"), 
                         model_method="lm",
        indep_vars_vctr=setdiff(names(glb_trnent_df), c(glb_exclude_vars_as_features, ".rnorm", grep(glb_rsp_var, names(glb_trnent_df), value=TRUE))),
                            model_type=glb_model_type,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_trnent_df, OOB_df=glb_newent_df,
                            n_cv_folds=0, tune_models_df=NULL)
```

```
## [1] "fitting model: All.X.cv.0.lm"
## [1] "    indep_vars: LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO"
## Fitting parameter = none on full training set
```

![](Boston_Housing_files/figure-html/fit.models-26.png) ![](Boston_Housing_files/figure-html/fit.models-27.png) ![](Boston_Housing_files/figure-html/fit.models-28.png) ![](Boston_Housing_files/figure-html/fit.models-29.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -14.511  -2.712  -0.676   1.793  36.883 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.523e+02  4.367e+02  -0.578   0.5638    
## LON         -2.987e+00  4.786e+00  -0.624   0.5329    
## LAT          1.544e+00  5.192e+00   0.297   0.7664    
## CRIM        -1.808e-01  4.390e-02  -4.118 4.77e-05 ***
## ZN           3.250e-02  1.877e-02   1.731   0.0843 .  
## INDUS       -4.297e-02  8.473e-02  -0.507   0.6124    
## CHAS         2.904e+00  1.220e+00   2.380   0.0178 *  
## NOX         -2.161e+01  5.414e+00  -3.992 7.98e-05 ***
## RM           6.284e+00  4.827e-01  13.019  < 2e-16 ***
## AGE         -4.430e-02  1.785e-02  -2.482   0.0135 *  
## DIS         -1.577e+00  2.842e-01  -5.551 5.63e-08 ***
## RAD          2.451e-01  9.728e-02   2.519   0.0122 *  
## TAX         -1.112e-02  5.452e-03  -2.040   0.0421 *  
## PTRATIO     -9.835e-01  1.939e-01  -5.072 6.38e-07 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 5.595 on 350 degrees of freedom
## Multiple R-squared:  0.665,	Adjusted R-squared:  0.6525 
## F-statistic: 53.43 on 13 and 350 DF,  p-value: < 2.2e-16
## 
##        model_id model_method
## 1 All.X.cv.0.lm           lm
##                                                                   feats
## 1 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               0                      0.414                 0.005
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.6649543     5.486684    0.6949362      4.62471        0.6525098
```

```r
# User specified
for (method in glb_models_method_vctr) {
    print(sprintf("iterating over method:%s", method))

    # All X that is not user excluded
    indep_vars_vctr <- setdiff(names(glb_trnent_df), 
        union(glb_rsp_var, glb_exclude_vars_as_features))
    
    # easier to exclude features
#     indep_vars_vctr <- setdiff(names(glb_trnent_df), 
#         union(union(glb_rsp_var, glb_exclude_vars_as_features), 
#               c("<feat1_name>", "<feat2_name>")))
    
    # easier to include features
#     indep_vars_vctr <- c("<feat1_name>", "<feat2_name>")

    # User specified bivariate models
#     indep_vars_vctr_lst <- list()
#     for (feat in setdiff(names(glb_trnent_df), 
#                          union(glb_rsp_var, glb_exclude_vars_as_features)))
#         indep_vars_vctr_lst[["feat"]] <- feat

    # User specified combinatorial models
#     indep_vars_vctr_lst <- list()
#     combn_mtrx <- combn(c("<feat1_name>", "<feat2_name>", "<featn_name>"), 
#                           <num_feats_to_choose>)
#     for (combn_ix in 1:ncol(combn_mtrx))
#         #print(combn_mtrx[, combn_ix])
#         indep_vars_vctr_lst[[combn_ix]] <- combn_mtrx[, combn_ix]

#     glb_sel_mdl <- glb_sel_wlm_mdl <- ret_lst[["model"]]
#     rpart_sel_wlm_mdl <- rpart(reformulate(indep_vars_vctr, response=glb_rsp_var), 
#                                data=glb_trnent_df, method="class", 
#                                control=rpart.control(cp=glb_sel_wlm_mdl$bestTune$cp),
#                            parms=list(loss=glb_model_metric_terms))
#     print("rpart_sel_wlm_mdl"); prp(rpart_sel_wlm_mdl)
# 
    model_id_pfx <- "All.X";
    ret_lst <- myfit_mdl(model_id=paste0(model_id_pfx, ""), model_method=method,
                            indep_vars_vctr=indep_vars_vctr,
                            model_type=glb_model_type,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_trnent_df, OOB_df=glb_newent_df,
                n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df)
    
    # rf is hard-coded in caret to recognize only Accuracy / Kappa evaluation metrics
    #   only for OOB in trainControl ?
#     if (method == "rpart")
#     ret_lst <- myfit_mdl_fn(model_id=paste0(model_id_pfx, ""), model_method=method,
#                             indep_vars_vctr=indep_vars_vctr,
#                             rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
#                             fit_df=glb_trnent_df, OOB_df=glb_newent_df,
#                             n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df,
#                             model_loss_mtrx=glb_model_metric_terms,
#                             model_summaryFunction=glb_model_metric_smmry,
#                             model_metric=glb_model_metric,
#                             model_metric_maximize=glb_model_metric_maximize)
}
```

```
## [1] "iterating over method:lm"
## [1] "fitting model: All.X.lm"
## [1] "    indep_vars: LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm"
## + Fold01: parameter=none 
## - Fold01: parameter=none 
## + Fold02: parameter=none 
## - Fold02: parameter=none 
## + Fold03: parameter=none 
## - Fold03: parameter=none 
## + Fold04: parameter=none 
## - Fold04: parameter=none 
## + Fold05: parameter=none 
## - Fold05: parameter=none 
## + Fold06: parameter=none 
## - Fold06: parameter=none 
## + Fold07: parameter=none 
## - Fold07: parameter=none 
## + Fold08: parameter=none 
## - Fold08: parameter=none 
## + Fold09: parameter=none 
## - Fold09: parameter=none 
## + Fold10: parameter=none 
## - Fold10: parameter=none 
## Aggregating results
## Fitting final model on full training set
```

![](Boston_Housing_files/figure-html/fit.models-30.png) ![](Boston_Housing_files/figure-html/fit.models-31.png) ![](Boston_Housing_files/figure-html/fit.models-32.png) ![](Boston_Housing_files/figure-html/fit.models-33.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -14.575  -2.882  -0.706   1.723  36.964 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.736e+02  4.379e+02  -0.625   0.5325    
## LON         -3.250e+00  4.802e+00  -0.677   0.4989    
## LAT          1.611e+00  5.196e+00   0.310   0.7567    
## CRIM        -1.809e-01  4.393e-02  -4.117 4.79e-05 ***
## ZN           3.241e-02  1.878e-02   1.726   0.0853 .  
## INDUS       -4.557e-02  8.486e-02  -0.537   0.5916    
## CHAS         2.908e+00  1.221e+00   2.382   0.0178 *  
## NOX         -2.174e+01  5.420e+00  -4.011 7.39e-05 ***
## RM           6.261e+00  4.840e-01  12.935  < 2e-16 ***
## AGE         -4.476e-02  1.787e-02  -2.504   0.0127 *  
## DIS         -1.584e+00  2.845e-01  -5.568 5.14e-08 ***
## RAD          2.440e-01  9.736e-02   2.506   0.0127 *  
## TAX         -1.099e-02  5.459e-03  -2.014   0.0448 *  
## PTRATIO     -9.796e-01  1.941e-01  -5.047 7.22e-07 ***
## .rnorm       2.137e-01  2.856e-01   0.748   0.4549    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 5.599 on 349 degrees of freedom
## Multiple R-squared:  0.6655,	Adjusted R-squared:  0.6521 
## F-statistic: 49.59 on 14 and 349 DF,  p-value: < 2.2e-16
## 
##   model_id model_method
## 1 All.X.lm           lm
##                                                                           feats
## 1 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.806                 0.006
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.6654907     5.541975     0.695457     4.620761         0.652072
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.6648438       1.600558            0.19777
## [1] "iterating over method:glm"
## [1] "fitting model: All.X.glm"
## [1] "    indep_vars: LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm"
## + Fold01: parameter=none 
## - Fold01: parameter=none 
## + Fold02: parameter=none 
## - Fold02: parameter=none 
## + Fold03: parameter=none 
## - Fold03: parameter=none 
## + Fold04: parameter=none 
## - Fold04: parameter=none 
## + Fold05: parameter=none 
## - Fold05: parameter=none 
## + Fold06: parameter=none 
## - Fold06: parameter=none 
## + Fold07: parameter=none 
## - Fold07: parameter=none 
## + Fold08: parameter=none 
## - Fold08: parameter=none 
## + Fold09: parameter=none 
## - Fold09: parameter=none 
## + Fold10: parameter=none 
## - Fold10: parameter=none 
## Aggregating results
## Fitting final model on full training set
```

![](Boston_Housing_files/figure-html/fit.models-34.png) ![](Boston_Housing_files/figure-html/fit.models-35.png) ![](Boston_Housing_files/figure-html/fit.models-36.png) ![](Boston_Housing_files/figure-html/fit.models-37.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##     Min       1Q   Median       3Q      Max  
## -14.575   -2.882   -0.706    1.723   36.964  
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.736e+02  4.379e+02  -0.625   0.5325    
## LON         -3.250e+00  4.802e+00  -0.677   0.4989    
## LAT          1.611e+00  5.196e+00   0.310   0.7567    
## CRIM        -1.809e-01  4.393e-02  -4.117 4.79e-05 ***
## ZN           3.241e-02  1.878e-02   1.726   0.0853 .  
## INDUS       -4.557e-02  8.486e-02  -0.537   0.5916    
## CHAS         2.908e+00  1.221e+00   2.382   0.0178 *  
## NOX         -2.174e+01  5.420e+00  -4.011 7.39e-05 ***
## RM           6.261e+00  4.840e-01  12.935  < 2e-16 ***
## AGE         -4.476e-02  1.787e-02  -2.504   0.0127 *  
## DIS         -1.584e+00  2.845e-01  -5.568 5.14e-08 ***
## RAD          2.440e-01  9.736e-02   2.506   0.0127 *  
## TAX         -1.099e-02  5.459e-03  -2.014   0.0448 *  
## PTRATIO     -9.796e-01  1.941e-01  -5.047 7.22e-07 ***
## .rnorm       2.137e-01  2.856e-01   0.748   0.4549    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for gaussian family taken to be 31.34729)
## 
##     Null deviance: 32705  on 363  degrees of freedom
## Residual deviance: 10940  on 349  degrees of freedom
## AIC: 2303.7
## 
## Number of Fisher Scoring iterations: 2
## 
##    model_id model_method
## 1 All.X.glm          glm
##                                                                           feats
## 1 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.964                  0.02
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB min.aic.fit
## 1    0.6654907     5.541975     0.695457     4.620761    2303.696
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.6648438       1.600558            0.19777
## [1] "iterating over method:rpart"
## [1] "fitting model: All.X.rpart"
## [1] "    indep_vars: LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO"
## + Fold01: cp=0.001 
## - Fold01: cp=0.001 
## + Fold02: cp=0.001 
## - Fold02: cp=0.001 
## + Fold03: cp=0.001 
## - Fold03: cp=0.001 
## + Fold04: cp=0.001 
## - Fold04: cp=0.001 
## + Fold05: cp=0.001 
## - Fold05: cp=0.001 
## + Fold06: cp=0.001 
## - Fold06: cp=0.001 
## + Fold07: cp=0.001 
## - Fold07: cp=0.001 
## + Fold08: cp=0.001 
## - Fold08: cp=0.001 
## + Fold09: cp=0.001 
## - Fold09: cp=0.001 
## + Fold10: cp=0.001 
## - Fold10: cp=0.001 
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.001 on full training set
```

```
## Warning in myfit_mdl(model_id = paste0(model_id_pfx, ""), model_method =
## method, : model's bestTune found at an extreme of tuneGrid for parameter:
## cp
```

![](Boston_Housing_files/figure-html/fit.models-38.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 364 
## 
##             CP nsplit rel error
## 1  0.502867534      0 1.0000000
## 2  0.109118325      1 0.4971325
## 3  0.063785018      2 0.3880141
## 4  0.055250497      3 0.3242291
## 5  0.027537041      4 0.2689786
## 6  0.025696175      5 0.2414416
## 7  0.022805757      6 0.2157454
## 8  0.006658586      7 0.1929397
## 9  0.006029867      8 0.1862811
## 10 0.005753869      9 0.1802512
## 11 0.004668334     10 0.1744973
## 12 0.004637271     11 0.1698290
## 13 0.004133845     12 0.1651917
## 14 0.003909661     13 0.1610579
## 15 0.003454681     14 0.1571482
## 16 0.003083094     15 0.1536935
## 17 0.002564086     16 0.1506104
## 18 0.001950367     17 0.1480464
## 19 0.001915924     18 0.1460960
## 20 0.001808365     19 0.1441801
## 21 0.001573639     20 0.1423717
## 22 0.001057732     21 0.1407981
## 23 0.001037443     23 0.1386826
## 24 0.001000000     24 0.1376452
## 
## Variable importance
##      RM     NOX    CRIM   INDUS PTRATIO     DIS      ZN     LON     RAD 
##      39      11       9       8       8       7       4       4       3 
##     TAX     LAT     AGE 
##       3       2       2 
## 
## Node number 1: 364 observations,    complexity param=0.5028675
##   mean=22.93407, MSE=89.84955 
##   left son=2 (313 obs) right son=3 (51 obs)
##   Primary splits:
##       RM      < 6.978     to the left,  improve=0.5028675, (0 missing)
##       INDUS   < 6.66      to the right, improve=0.2745342, (0 missing)
##       LON     < -71.06825 to the right, improve=0.2713269, (0 missing)
##       PTRATIO < 19.65     to the right, improve=0.2539395, (0 missing)
##       NOX     < 0.6695    to the right, improve=0.2181593, (0 missing)
##   Surrogate splits:
##       PTRATIO < 14.15     to the right, agree=0.890, adj=0.216, (0 split)
##       ZN      < 87.5      to the left,  agree=0.874, adj=0.098, (0 split)
##       INDUS   < 1.605     to the right, agree=0.874, adj=0.098, (0 split)
##       CRIM    < 0.013715  to the right, agree=0.865, adj=0.039, (0 split)
## 
## Node number 2: 313 observations,    complexity param=0.1091183
##   mean=20.22077, MSE=43.45264 
##   left son=4 (72 obs) right son=5 (241 obs)
##   Primary splits:
##       NOX     < 0.6695    to the right, improve=0.2623944, (0 missing)
##       CRIM    < 5.84803   to the right, improve=0.2320796, (0 missing)
##       PTRATIO < 19.9      to the right, improve=0.2063838, (0 missing)
##       INDUS   < 16.57     to the right, improve=0.2044279, (0 missing)
##       AGE     < 74.45     to the right, improve=0.2027771, (0 missing)
##   Surrogate splits:
##       CRIM  < 1.40092   to the right, agree=0.872, adj=0.444, (0 split)
##       DIS   < 1.94      to the left,  agree=0.853, adj=0.361, (0 split)
##       RAD   < 16        to the right, agree=0.853, adj=0.361, (0 split)
##       TAX   < 551.5     to the right, agree=0.843, adj=0.319, (0 split)
##       INDUS < 16.57     to the right, agree=0.818, adj=0.208, (0 split)
## 
## Node number 3: 51 observations,    complexity param=0.0552505
##   mean=39.58627, MSE=52.12079 
##   left son=6 (25 obs) right son=7 (26 obs)
##   Primary splits:
##       RM      < 7.435     to the left,  improve=0.6797862, (0 missing)
##       LON     < -71.0731  to the right, improve=0.2403908, (0 missing)
##       LAT     < 42.25465  to the right, improve=0.1307498, (0 missing)
##       DIS     < 6.40795   to the right, improve=0.1278173, (0 missing)
##       PTRATIO < 18.15     to the right, improve=0.1189954, (0 missing)
##   Surrogate splits:
##       LON   < -71.076   to the right, agree=0.725, adj=0.44, (0 split)
##       LAT   < 42.23765  to the right, agree=0.686, adj=0.36, (0 split)
##       CRIM  < 0.10593   to the left,  agree=0.667, adj=0.32, (0 split)
##       INDUS < 5.405     to the left,  agree=0.667, adj=0.32, (0 split)
##       DIS   < 6.40795   to the right, agree=0.667, adj=0.32, (0 split)
## 
## Node number 4: 72 observations,    complexity param=0.02753704
##   mean=14.04306, MSE=25.1244 
##   left son=8 (39 obs) right son=9 (33 obs)
##   Primary splits:
##       CRIM < 6.99237   to the right, improve=0.4978591, (0 missing)
##       LON  < -71.05925 to the right, improve=0.3511717, (0 missing)
##       NOX  < 0.7065    to the left,  improve=0.3345863, (0 missing)
##       DIS  < 2.03385   to the left,  improve=0.2476719, (0 missing)
##       LAT  < 42.20325  to the left,  improve=0.1446017, (0 missing)
##   Surrogate splits:
##       NOX   < 0.755     to the left,  agree=0.778, adj=0.515, (0 split)
##       LON   < -71.049   to the right, agree=0.764, adj=0.485, (0 split)
##       DIS   < 2.24675   to the left,  agree=0.722, adj=0.394, (0 split)
##       INDUS < 18.84     to the left,  agree=0.708, adj=0.364, (0 split)
##       RAD   < 14.5      to the right, agree=0.708, adj=0.364, (0 split)
## 
## Node number 5: 241 observations,    complexity param=0.06378502
##   mean=22.06639, MSE=34.12024 
##   left son=10 (188 obs) right son=11 (53 obs)
##   Primary splits:
##       RM      < 6.484     to the left,  improve=0.2536921, (0 missing)
##       NOX     < 0.5125    to the right, improve=0.1273391, (0 missing)
##       LON     < -71.09365 to the right, improve=0.1255078, (0 missing)
##       INDUS   < 6.66      to the right, improve=0.1153673, (0 missing)
##       PTRATIO < 19.65     to the right, improve=0.1038293, (0 missing)
##   Surrogate splits:
##       ZN    < 29        to the left,  agree=0.817, adj=0.170, (0 split)
##       CRIM  < 0.01837   to the right, agree=0.801, adj=0.094, (0 split)
##       INDUS < 3.095     to the right, agree=0.801, adj=0.094, (0 split)
##       LON   < -70.92075 to the left,  agree=0.788, adj=0.038, (0 split)
##       DIS   < 8.19015   to the left,  agree=0.788, adj=0.038, (0 split)
## 
## Node number 6: 25 observations,    complexity param=0.001950367
##   mean=33.516, MSE=14.61254 
##   left son=12 (8 obs) right son=13 (17 obs)
##   Primary splits:
##       LAT  < 42.2005   to the left,  improve=0.1746094, (0 missing)
##       NOX  < 0.496     to the right, improve=0.1590396, (0 missing)
##       CRIM < 0.276995  to the right, improve=0.1590396, (0 missing)
##       TAX  < 258       to the right, improve=0.1583882, (0 missing)
##       RM   < 7.2615    to the right, improve=0.1571862, (0 missing)
##   Surrogate splits:
##       CRIM    < 0.724605  to the right, agree=0.80, adj=0.375, (0 split)
##       RAD     < 6         to the right, agree=0.80, adj=0.375, (0 split)
##       INDUS   < 4.46      to the right, agree=0.76, adj=0.250, (0 split)
##       PTRATIO < 18.95     to the right, agree=0.76, adj=0.250, (0 split)
##       NOX     < 0.496     to the right, agree=0.72, adj=0.125, (0 split)
## 
## Node number 7: 26 observations,    complexity param=0.003454681
##   mean=45.42308, MSE=18.68716 
##   left son=14 (13 obs) right son=15 (13 obs)
##   Primary splits:
##       PTRATIO < 15.4      to the right, improve=0.2325458, (0 missing)
##       RAD     < 3.5       to the left,  improve=0.1425359, (0 missing)
##       DIS     < 3.20745   to the right, improve=0.1342571, (0 missing)
##       NOX     < 0.541     to the left,  improve=0.1223549, (0 missing)
##       CRIM    < 0.45114   to the left,  improve=0.1170314, (0 missing)
##   Surrogate splits:
##       NOX < 0.541     to the left,  agree=0.769, adj=0.538, (0 split)
##       RAD < 6         to the right, agree=0.769, adj=0.538, (0 split)
##       LON < -71.09535 to the left,  agree=0.731, adj=0.462, (0 split)
##       ZN  < 10        to the left,  agree=0.731, adj=0.462, (0 split)
##       AGE < 87.95     to the left,  agree=0.731, adj=0.462, (0 split)
## 
## Node number 8: 39 observations,    complexity param=0.004133845
##   mean=10.78974, MSE=9.676305 
##   left son=16 (20 obs) right son=17 (19 obs)
##   Primary splits:
##       CRIM < 11.36915  to the right, improve=0.3582592, (0 missing)
##       DIS  < 2.03385   to the left,  improve=0.3306720, (0 missing)
##       NOX  < 0.7065    to the left,  improve=0.2876345, (0 missing)
##       LAT  < 42.1905   to the right, improve=0.1591293, (0 missing)
##       RM   < 5.853     to the left,  improve=0.1359366, (0 missing)
##   Surrogate splits:
##       DIS < 1.9249    to the left,  agree=0.718, adj=0.421, (0 split)
##       LON < -71.04675 to the right, agree=0.667, adj=0.316, (0 split)
##       LAT < 42.188    to the right, agree=0.667, adj=0.316, (0 split)
##       NOX < 0.7065    to the left,  agree=0.641, adj=0.263, (0 split)
##       RM  < 5.3695    to the left,  agree=0.641, adj=0.263, (0 split)
## 
## Node number 9: 33 observations,    complexity param=0.004668334
##   mean=17.88788, MSE=16.09016 
##   left son=18 (21 obs) right son=19 (12 obs)
##   Primary splits:
##       LON < -71.05925 to the right, improve=0.2875445, (0 missing)
##       AGE < 92        to the right, improve=0.1845894, (0 missing)
##       DIS < 1.75085   to the left,  improve=0.1150895, (0 missing)
##       LAT < 42.2013   to the left,  improve=0.1107536, (0 missing)
##       RAD < 14.5      to the left,  improve=0.1062345, (0 missing)
##   Surrogate splits:
##       NOX  < 0.7155    to the left,  agree=0.758, adj=0.333, (0 split)
##       CHAS < 0.5       to the left,  agree=0.727, adj=0.250, (0 split)
##       LAT  < 42.2013   to the left,  agree=0.697, adj=0.167, (0 split)
##       CRIM < 1.825685  to the right, agree=0.697, adj=0.167, (0 split)
##       AGE  < 82.8      to the right, agree=0.667, adj=0.083, (0 split)
## 
## Node number 10: 188 observations,    complexity param=0.02569617
##   mean=20.50426, MSE=27.90817 
##   left son=20 (179 obs) right son=21 (9 obs)
##   Primary splits:
##       NOX     < 0.6275    to the left,  improve=0.16017570, (0 missing)
##       DIS     < 1.4262    to the right, improve=0.15937760, (0 missing)
##       AGE     < 69.95     to the right, improve=0.07905465, (0 missing)
##       RM      < 5.858     to the left,  improve=0.07458818, (0 missing)
##       PTRATIO < 20.95     to the right, improve=0.06294100, (0 missing)
##   Surrogate splits:
##       DIS < 1.37275   to the right, agree=0.984, adj=0.667, (0 split)
##       RM  < 4.383     to the right, agree=0.968, adj=0.333, (0 split)
## 
## Node number 11: 53 observations,    complexity param=0.004637271
##   mean=27.60755, MSE=16.79504 
##   left son=22 (38 obs) right son=23 (15 obs)
##   Primary splits:
##       TAX     < 269       to the right, improve=0.17038170, (0 missing)
##       PTRATIO < 18.35     to the right, improve=0.16541530, (0 missing)
##       LON     < -71.05375 to the right, improve=0.12893610, (0 missing)
##       NOX     < 0.515     to the right, improve=0.10740180, (0 missing)
##       AGE     < 34        to the right, improve=0.09853561, (0 missing)
##   Surrogate splits:
##       LAT   < 42.28825  to the left,  agree=0.792, adj=0.267, (0 split)
##       INDUS < 3.385     to the right, agree=0.792, adj=0.267, (0 split)
##       CRIM  < 0.02831   to the right, agree=0.774, adj=0.200, (0 split)
##       ZN    < 57.5      to the left,  agree=0.774, adj=0.200, (0 split)
##       NOX   < 0.4185    to the right, agree=0.774, adj=0.200, (0 split)
## 
## Node number 12: 8 observations
##   mean=31.1875, MSE=30.57109 
## 
## Node number 13: 17 observations
##   mean=34.61176, MSE=3.35045 
## 
## Node number 14: 13 observations
##   mean=43.33846, MSE=20.27929 
## 
## Node number 15: 13 observations
##   mean=47.50769, MSE=8.403787 
## 
## Node number 16: 20 observations,    complexity param=0.001037443
##   mean=8.975, MSE=4.310875 
##   left son=32 (7 obs) right son=33 (13 obs)
##   Primary splits:
##       AGE  < 98.5      to the right, improve=0.39353740, (0 missing)
##       CRIM < 24.22495  to the right, improve=0.12940850, (0 missing)
##       DIS  < 1.7364    to the left,  improve=0.12231370, (0 missing)
##       RM   < 5.7505    to the left,  improve=0.11504330, (0 missing)
##       LON  < -71.03725 to the right, improve=0.05454991, (0 missing)
##   Surrogate splits:
##       CRIM < 43.63765  to the right, agree=0.7, adj=0.143, (0 split)
##       RM   < 4.94      to the left,  agree=0.7, adj=0.143, (0 split)
## 
## Node number 17: 19 observations
##   mean=12.7, MSE=8.208421 
## 
## Node number 18: 21 observations,    complexity param=0.001573639
##   mean=16.2619, MSE=9.47093 
##   left son=36 (10 obs) right son=37 (11 obs)
##   Primary splits:
##       AGE     < 92        to the right, improve=0.25876800, (0 missing)
##       LON     < -71.0445  to the left,  improve=0.17667160, (0 missing)
##       DIS     < 1.71455   to the left,  improve=0.15347860, (0 missing)
##       PTRATIO < 17.45     to the left,  improve=0.07845466, (0 missing)
##       RAD     < 14.5      to the left,  improve=0.07845466, (0 missing)
##   Surrogate splits:
##       LAT   < 42.17875  to the right, agree=0.810, adj=0.6, (0 split)
##       DIS   < 1.5699    to the left,  agree=0.810, adj=0.6, (0 split)
##       INDUS < 18.84     to the right, agree=0.762, adj=0.5, (0 split)
##       NOX   < 0.7945    to the right, agree=0.762, adj=0.5, (0 split)
##       RAD   < 14.5      to the left,  agree=0.762, adj=0.5, (0 split)
## 
## Node number 19: 12 observations
##   mean=20.73333, MSE=14.95056 
## 
## Node number 20: 179 observations,    complexity param=0.02280576
##   mean=20.03017, MSE=13.46624 
##   left son=40 (85 obs) right son=41 (94 obs)
##   Primary splits:
##       AGE     < 69.95     to the right, improve=0.3094300, (0 missing)
##       PTRATIO < 19.65     to the right, improve=0.2615820, (0 missing)
##       NOX     < 0.5125    to the right, improve=0.2479085, (0 missing)
##       DIS     < 2.2085    to the left,  improve=0.2304457, (0 missing)
##       TAX     < 434.5     to the right, improve=0.2018875, (0 missing)
##   Surrogate splits:
##       NOX   < 0.5175    to the right, agree=0.816, adj=0.612, (0 split)
##       DIS   < 3.1073    to the left,  agree=0.788, adj=0.553, (0 split)
##       INDUS < 7.625     to the right, agree=0.754, adj=0.482, (0 split)
##       CRIM  < 0.11048   to the right, agree=0.715, adj=0.400, (0 split)
##       TAX   < 371       to the right, agree=0.709, adj=0.388, (0 split)
## 
## Node number 21: 9 observations
##   mean=29.93333, MSE=221.7644 
## 
## Node number 22: 38 observations,    complexity param=0.003909661
##   mean=26.54474, MSE=16.79879 
##   left son=44 (16 obs) right son=45 (22 obs)
##   Primary splits:
##       PTRATIO < 17.7      to the right, improve=0.2003064, (0 missing)
##       LON     < -71.0565  to the right, improve=0.1814292, (0 missing)
##       RM      < 6.8425    to the left,  improve=0.1155244, (0 missing)
##       LAT     < 42.19225  to the left,  improve=0.1092606, (0 missing)
##       AGE     < 34        to the right, improve=0.0887140, (0 missing)
##   Surrogate splits:
##       LON   < -71.08425 to the right, agree=0.763, adj=0.438, (0 split)
##       NOX   < 0.515     to the right, agree=0.737, adj=0.375, (0 split)
##       INDUS < 7.38      to the right, agree=0.711, adj=0.312, (0 split)
##       DIS   < 2.8589    to the left,  agree=0.711, adj=0.312, (0 split)
##       TAX   < 329.5     to the right, agree=0.711, adj=0.312, (0 split)
## 
## Node number 23: 15 observations
##   mean=30.3, MSE=6.674667 
## 
## Node number 32: 7 observations
##   mean=7.2, MSE=2.491429 
## 
## Node number 33: 13 observations
##   mean=9.930769, MSE=2.680592 
## 
## Node number 36: 10 observations
##   mean=14.62, MSE=2.7816 
## 
## Node number 37: 11 observations
##   mean=17.75455, MSE=10.87339 
## 
## Node number 40: 85 observations,    complexity param=0.006029867
##   mean=17.88353, MSE=9.762317 
##   left son=80 (67 obs) right son=81 (18 obs)
##   Primary splits:
##       CRIM    < 0.140955  to the right, improve=0.2376584, (0 missing)
##       LON     < -71.0621  to the right, improve=0.2261618, (0 missing)
##       PTRATIO < 19.15     to the right, improve=0.2189265, (0 missing)
##       AGE     < 93.2      to the right, improve=0.2073137, (0 missing)
##       RM      < 5.7695    to the left,  improve=0.1936222, (0 missing)
##   Surrogate splits:
##       TAX   < 276.5     to the right, agree=0.835, adj=0.222, (0 split)
##       INDUS < 4.59      to the right, agree=0.824, adj=0.167, (0 split)
##       RAD   < 2.5       to the right, agree=0.824, adj=0.167, (0 split)
##       RM    < 6.3605    to the left,  agree=0.800, adj=0.056, (0 split)
## 
## Node number 41: 94 observations,    complexity param=0.006658586
##   mean=21.97128, MSE=8.880771 
##   left son=82 (48 obs) right son=83 (46 obs)
##   Primary splits:
##       RM      < 6.0615    to the left,  improve=0.26086800, (0 missing)
##       PTRATIO < 19.65     to the right, improve=0.12362920, (0 missing)
##       LON     < -71.0925  to the right, improve=0.12281530, (0 missing)
##       LAT     < 42.1946   to the left,  improve=0.08135231, (0 missing)
##       CRIM    < 1.71625   to the right, improve=0.07998606, (0 missing)
##   Surrogate splits:
##       NOX     < 0.496     to the right, agree=0.691, adj=0.370, (0 split)
##       LON     < -71.02875 to the right, agree=0.649, adj=0.283, (0 split)
##       ZN      < 21.5      to the left,  agree=0.649, adj=0.283, (0 split)
##       PTRATIO < 18.75     to the right, agree=0.617, adj=0.217, (0 split)
##       INDUS   < 5.03      to the right, agree=0.606, adj=0.196, (0 split)
## 
## Node number 44: 16 observations
##   mean=24.39375, MSE=11.17184 
## 
## Node number 45: 22 observations,    complexity param=0.002564086
##   mean=28.10909, MSE=15.07901 
##   left son=90 (13 obs) right son=91 (9 obs)
##   Primary splits:
##       DIS  < 3.87095   to the right, improve=0.2527868, (0 missing)
##       LAT  < 42.21395  to the left,  improve=0.2203815, (0 missing)
##       ZN   < 9         to the right, improve=0.1710104, (0 missing)
##       NOX  < 0.4605    to the left,  improve=0.1614270, (0 missing)
##       CRIM < 0.11954   to the left,  improve=0.1565339, (0 missing)
##   Surrogate splits:
##       NOX  < 0.494     to the left,  agree=0.909, adj=0.778, (0 split)
##       ZN   < 9         to the right, agree=0.864, adj=0.667, (0 split)
##       AGE  < 65.85     to the left,  agree=0.864, adj=0.667, (0 split)
##       LAT  < 42.2122   to the left,  agree=0.818, adj=0.556, (0 split)
##       CRIM < 0.243705  to the left,  agree=0.818, adj=0.556, (0 split)
## 
## Node number 80: 67 observations,    complexity param=0.005753869
##   mean=17.09403, MSE=8.92683 
##   left son=160 (45 obs) right son=161 (22 obs)
##   Primary splits:
##       LON     < -71.0621  to the right, improve=0.3146337, (0 missing)
##       PTRATIO < 19.15     to the right, improve=0.1806497, (0 missing)
##       RM      < 5.7695    to the left,  improve=0.1741352, (0 missing)
##       AGE     < 93.2      to the right, improve=0.1705091, (0 missing)
##       DIS     < 2.2085    to the left,  improve=0.1504167, (0 missing)
##   Surrogate splits:
##       NOX     < 0.522     to the right, agree=0.731, adj=0.182, (0 split)
##       PTRATIO < 14.95     to the right, agree=0.731, adj=0.182, (0 split)
##       CHAS    < 0.5       to the left,  agree=0.701, adj=0.091, (0 split)
##       DIS     < 7.3867    to the left,  agree=0.701, adj=0.091, (0 split)
##       LAT     < 42.1805   to the right, agree=0.687, adj=0.045, (0 split)
## 
## Node number 81: 18 observations
##   mean=20.82222, MSE=1.916173 
## 
## Node number 82: 48 observations,    complexity param=0.001915924
##   mean=20.48125, MSE=6.407357 
##   left son=164 (21 obs) right son=165 (27 obs)
##   Primary splits:
##       LAT     < 42.20835  to the left,  improve=0.20373960, (0 missing)
##       LON     < -71.08185 to the right, improve=0.16651880, (0 missing)
##       RM      < 5.8435    to the left,  improve=0.14735910, (0 missing)
##       PTRATIO < 19.65     to the right, improve=0.12008380, (0 missing)
##       CRIM    < 0.67778   to the right, improve=0.09574128, (0 missing)
##   Surrogate splits:
##       PTRATIO < 19.4      to the right, agree=0.708, adj=0.333, (0 split)
##       TAX     < 228.5     to the left,  agree=0.688, adj=0.286, (0 split)
##       CRIM    < 0.0434    to the left,  agree=0.667, adj=0.238, (0 split)
##       INDUS   < 5.415     to the left,  agree=0.667, adj=0.238, (0 split)
##       RM      < 5.967     to the right, agree=0.646, adj=0.190, (0 split)
## 
## Node number 83: 46 observations,    complexity param=0.001057732
##   mean=23.52609, MSE=6.72758 
##   left son=166 (39 obs) right son=167 (7 obs)
##   Primary splits:
##       TAX   < 242       to the right, improve=0.11005720, (0 missing)
##       NOX   < 0.4555    to the left,  improve=0.10385270, (0 missing)
##       INDUS < 3.13      to the right, improve=0.09226275, (0 missing)
##       RM    < 6.142     to the left,  improve=0.06935965, (0 missing)
##       RAD   < 4.5       to the right, improve=0.05863127, (0 missing)
## 
## Node number 90: 13 observations
##   mean=26.48462, MSE=7.661302 
## 
## Node number 91: 9 observations
##   mean=30.45556, MSE=16.4758 
## 
## Node number 160: 45 observations,    complexity param=0.003083094
##   mean=15.92222, MSE=6.999062 
##   left son=320 (27 obs) right son=321 (18 obs)
##   Primary splits:
##       PTRATIO < 19.45     to the right, improve=0.3201487, (0 missing)
##       TAX     < 434.5     to the right, improve=0.2272027, (0 missing)
##       CRIM    < 0.903905  to the right, improve=0.2175012, (0 missing)
##       AGE     < 91.95     to the right, improve=0.2060130, (0 missing)
##       NOX     < 0.5825    to the right, improve=0.1835506, (0 missing)
##   Surrogate splits:
##       CRIM  < 0.26266   to the right, agree=0.844, adj=0.611, (0 split)
##       TAX   < 305.5     to the right, agree=0.800, adj=0.500, (0 split)
##       LON   < -71.04165 to the left,  agree=0.778, adj=0.444, (0 split)
##       NOX   < 0.5825    to the right, agree=0.756, adj=0.389, (0 split)
##       INDUS < 14.055    to the right, agree=0.733, adj=0.333, (0 split)
## 
## Node number 161: 22 observations
##   mean=19.49091, MSE=4.316281 
## 
## Node number 164: 21 observations,    complexity param=0.001808365
##   mean=19.18571, MSE=7.430748 
##   left son=328 (13 obs) right son=329 (8 obs)
##   Primary splits:
##       LON     < -71.0624  to the right, improve=0.37901090, (0 missing)
##       RM      < 5.8465    to the left,  improve=0.15139790, (0 missing)
##       TAX     < 320.5     to the right, improve=0.09538651, (0 missing)
##       PTRATIO < 18.7      to the right, improve=0.06549328, (0 missing)
##       NOX     < 0.5095    to the right, improve=0.05586000, (0 missing)
##   Surrogate splits:
##       CRIM  < 0.07408   to the left,  agree=0.857, adj=0.625, (0 split)
##       INDUS < 6.13      to the left,  agree=0.857, adj=0.625, (0 split)
##       DIS   < 4.57505   to the right, agree=0.810, adj=0.500, (0 split)
##       LAT   < 42.15215  to the left,  agree=0.762, adj=0.375, (0 split)
##       RAD   < 6.5       to the left,  agree=0.762, adj=0.375, (0 split)
## 
## Node number 165: 27 observations
##   mean=21.48889, MSE=3.290617 
## 
## Node number 166: 39 observations,    complexity param=0.001057732
##   mean=23.16154, MSE=2.359803 
##   left son=332 (22 obs) right son=333 (17 obs)
##   Primary splits:
##       AGE  < 43.7      to the right, improve=0.3816866, (0 missing)
##       TAX  < 353       to the right, improve=0.2019025, (0 missing)
##       RM   < 6.3665    to the left,  improve=0.1718166, (0 missing)
##       LON  < -71.12225 to the right, improve=0.1431431, (0 missing)
##       CRIM < 0.125455  to the left,  improve=0.1267532, (0 missing)
##   Surrogate splits:
##       DIS < 5.2074    to the left,  agree=0.769, adj=0.471, (0 split)
##       NOX < 0.434     to the right, agree=0.744, adj=0.412, (0 split)
##       LON < -71.1049  to the right, agree=0.718, adj=0.353, (0 split)
##       ZN  < 21.5      to the left,  agree=0.718, adj=0.353, (0 split)
##       LAT < 42.12775  to the right, agree=0.692, adj=0.294, (0 split)
## 
## Node number 167: 7 observations
##   mean=25.55714, MSE=26.19673 
## 
## Node number 320: 27 observations
##   mean=14.7, MSE=5.69037 
## 
## Node number 321: 18 observations
##   mean=17.75556, MSE=3.360247 
## 
## Node number 328: 13 observations
##   mean=17.86923, MSE=6.474438 
## 
## Node number 329: 8 observations
##   mean=21.325, MSE=1.591875 
## 
## Node number 332: 22 observations
##   mean=22.32727, MSE=1.236529 
## 
## Node number 333: 17 observations
##   mean=24.24118, MSE=1.747128 
## 
## n= 364 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
##   1) root 364 32705.24000 22.934070  
##     2) RM< 6.978 313 13600.68000 20.220770  
##       4) NOX>=0.6695 72  1808.95700 14.043060  
##         8) CRIM>=6.99237 39   377.37590 10.789740  
##          16) CRIM>=11.36915 20    86.21750  8.975000  
##            32) AGE>=98.5 7    17.44000  7.200000 *
##            33) AGE< 98.5 13    34.84769  9.930769 *
##          17) CRIM< 11.36915 19   155.96000 12.700000 *
##         9) CRIM< 6.99237 33   530.97520 17.887880  
##          18) LON>=-71.05925 21   198.88950 16.261900  
##            36) AGE>=92 10    27.81600 14.620000 *
##            37) AGE< 92 11   119.60730 17.754550 *
##          19) LON< -71.05925 12   179.40670 20.733330 *
##       5) NOX< 0.6695 241  8222.97800 22.066390  
##        10) RM< 6.484 188  5246.73700 20.504260  
##          20) NOX< 0.6275 179  2410.45700 20.030170  
##            40) AGE>=69.95 85   829.79690 17.883530  
##              80) CRIM>=0.140955 67   598.09760 17.094030  
##               160) LON>=-71.0621 45   314.95780 15.922220  
##                 320) PTRATIO>=19.45 27   153.64000 14.700000 *
##                 321) PTRATIO< 19.45 18    60.48444 17.755560 *
##               161) LON< -71.0621 22    94.95818 19.490910 *
##              81) CRIM< 0.140955 18    34.49111 20.822220 *
##            41) AGE< 69.95 94   834.79240 21.971280  
##              82) RM< 6.0615 48   307.55310 20.481250  
##               164) LAT< 42.20835 21   156.04570 19.185710  
##                 328) LON>=-71.0624 13    84.16769 17.869230 *
##                 329) LON< -71.0624 8    12.73500 21.325000 *
##               165) LAT>=42.20835 27    88.84667 21.488890 *
##              83) RM>=6.0615 46   309.46870 23.526090  
##               166) TAX>=242 39    92.03231 23.161540  
##                 332) AGE>=43.7 22    27.20364 22.327270 *
##                 333) AGE< 43.7 17    29.70118 24.241180 *
##               167) TAX< 242 7   183.37710 25.557140 *
##          21) NOX>=0.6275 9  1995.88000 29.933330 *
##        11) RM>=6.484 53   890.13700 27.607550  
##          22) TAX>=269 38   638.35390 26.544740  
##            44) PTRATIO>=17.7 16   178.74940 24.393750 *
##            45) PTRATIO< 17.7 22   331.73820 28.109090  
##              90) DIS>=3.87095 13    99.59692 26.484620 *
##              91) DIS< 3.87095 9   148.28220 30.455560 *
##          23) TAX< 269 15   100.12000 30.300000 *
##     3) RM>=6.978 51  2658.16000 39.586270  
##       6) RM< 7.435 25   365.31360 33.516000  
##        12) LAT< 42.2005 8   244.56870 31.187500 *
##        13) LAT>=42.2005 17    56.95765 34.611760 *
##       7) RM>=7.435 26   485.86620 45.423080  
##        14) PTRATIO>=15.4 13   263.63080 43.338460 *
##        15) PTRATIO< 15.4 13   109.24920 47.507690 *
##      model_id model_method
## 1 All.X.rpart        rpart
##                                                                   feats
## 1 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1              10                      1.553                 0.025
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit
## 1    0.8623548     4.607105    0.6307835     5.087797        0.7465918
##   min.RMSESD.fit max.RsquaredSD.fit
## 1       1.690352          0.2140281
## [1] "iterating over method:rf"
## [1] "fitting model: All.X.rf"
## [1] "    indep_vars: LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm"
```

```
## Loading required package: randomForest
## randomForest 4.6-10
## Type rfNews() to see new features/changes/bug fixes.
```

![](Boston_Housing_files/figure-html/fit.models-39.png) 

```
## + : mtry= 2 
## - : mtry= 2 
## + : mtry= 8 
## - : mtry= 8 
## + : mtry=14 
## - : mtry=14 
## Aggregating results
## Selecting tuning parameters
## Fitting mtry = 8 on full training set
```

![](Boston_Housing_files/figure-html/fit.models-40.png) ![](Boston_Housing_files/figure-html/fit.models-41.png) 

```
##                 Length Class      Mode     
## call              4    -none-     call     
## type              1    -none-     character
## predicted       364    -none-     numeric  
## mse             500    -none-     numeric  
## rsq             500    -none-     numeric  
## oob.times       364    -none-     numeric  
## importance       14    -none-     numeric  
## importanceSD      0    -none-     NULL     
## localImportance   0    -none-     NULL     
## proximity         0    -none-     NULL     
## ntree             1    -none-     numeric  
## mtry              1    -none-     numeric  
## forest           11    -none-     list     
## coefs             0    -none-     NULL     
## y               364    -none-     numeric  
## test              0    -none-     NULL     
## inbag             0    -none-     NULL     
## xNames           14    -none-     character
## problemType       1    -none-     character
## tuneValue         1    data.frame list     
## obsLevels         1    -none-     logical  
##   model_id model_method
## 1 All.X.rf           rf
##                                                                           feats
## 1 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                      5.132                 0.961
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit
## 1    0.9619153      4.28035    0.7940646     3.797181         0.796088
```

```r
# Simplify a model
# fit_df <- glb_trnent_df; glb_mdl <- step(<complex>_mdl)

print(glb_models_df)
```

```
##                  model_id model_method
## 1                  MFO.lm           lm
## 2               latlon.lm           lm
## 3            latlon.rpart        rpart
## 4       Max.cor.Y.cv.0.lm           lm
## 5       Max.cor.Y.cv.G.lm           lm
## 6  Interact.High.cor.y.lm           lm
## 7            Low.cor.X.lm           lm
## 8           All.X.cv.0.lm           lm
## 9                All.X.lm           lm
## 10              All.X.glm          glm
## 11            All.X.rpart        rpart
## 12               All.X.rf           rf
##                                                                            feats
## 1                                                                         .rnorm
## 2                                                                       LAT, LON
## 3                                                                       LAT, LON
## 4                                                                             RM
## 5                                                                             RM
## 6                                             RM, RM:DIS, RM:RAD, RM:NOX, RM:TAX
## 7                      RM, ZN, CHAS, .rnorm, LAT, LON, AGE, CRIM, INDUS, PTRATIO
## 8          LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO
## 9  LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
## 10 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
## 11         LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO
## 12 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
##    max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1                0                      0.605                 0.004
## 2                0                      0.414                 0.003
## 3                0                      0.533                 0.018
## 4                0                      0.431                 0.002
## 5                1                      0.805                 0.003
## 6                1                      0.981                 0.004
## 7                1                      0.793                 0.004
## 8                0                      0.414                 0.005
## 9                1                      0.806                 0.006
## 10               1                      0.964                 0.020
## 11              10                      1.553                 0.025
## 12               3                      5.132                 0.961
##    max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1  0.0006593684     9.475775 -0.002734968     8.384599     -0.002101241
## 2  0.1071648958     8.667656           NA           NA      0.103614856
## 3  0.5635405103     6.060215           NA           NA               NA
## 4  0.5022979132     6.687175  0.422965492     6.360483      0.500923046
## 5  0.5022979132     6.625483  0.422965492     6.360483      0.500923046
## 6  0.6029820329     5.868191  0.591061401     5.354492      0.597437089
## 7  0.6253946970     5.793811  0.648918353     4.961275      0.614782649
## 8  0.6649543413     5.486684  0.694936215     4.624710      0.652509788
## 9  0.6654907244     5.541975  0.695457041     4.620761      0.652072014
## 10 0.6654907244     5.541975  0.695457041     4.620761               NA
## 11 0.8623548418     4.607105  0.630783469     5.087797               NA
## 12 0.9619152730     4.280350  0.794064554     3.797181               NA
##    max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit min.aic.fit
## 1                NA             NA                 NA          NA
## 2                NA             NA                 NA          NA
## 3                NA             NA                 NA          NA
## 4                NA             NA                 NA          NA
## 5         0.5335055       1.335810          0.1836241          NA
## 6         0.6342187       1.804358          0.2221433          NA
## 7         0.6393438       1.602839          0.2061768          NA
## 8                NA             NA                 NA          NA
## 9         0.6648438       1.600558          0.1977700          NA
## 10        0.6648438       1.600558          0.1977700    2303.696
## 11        0.7465918       1.690352          0.2140281          NA
## 12        0.7960880             NA                 NA          NA
```

```r
if (!is.null(glb_model_metric_smmry)) {
    stats_df <- glb_models_df[, "model_id", FALSE]

    stats_mdl_df <- data.frame()
    for (model_id in stats_df$model_id) {
        stats_mdl_df <- rbind(stats_mdl_df, 
            mypredict_mdl(glb_models_lst[[model_id]], glb_trnent_df, glb_rsp_var, 
                          glb_rsp_var_out, model_id, "fit",
        						glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="stats"))
    }
    stats_df <- merge(stats_df, stats_mdl_df, all.x=TRUE)
    
    stats_mdl_df <- data.frame()
    for (model_id in stats_df$model_id) {
        stats_mdl_df <- rbind(stats_mdl_df, 
            mypredict_mdl(glb_models_lst[[model_id]], glb_newent_df, glb_rsp_var, 
                          glb_rsp_var_out, model_id, "OOB",
            					glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="stats"))
    }
    stats_df <- merge(stats_df, stats_mdl_df, all.x=TRUE)
    
#     tmp_models_df <- orderBy(~model_id, glb_models_df)
#     rownames(tmp_models_df) <- seq(1, nrow(tmp_models_df))
#     all.equal(subset(tmp_models_df[, names(stats_df)], model_id != "Random.myrandom_classfr"),
#               subset(stats_df, model_id != "Random.myrandom_classfr"))
#     print(subset(tmp_models_df[, names(stats_df)], model_id != "Random.myrandom_classfr")[, c("model_id", "max.Accuracy.fit")])
#     print(subset(stats_df, model_id != "Random.myrandom_classfr")[, c("model_id", "max.Accuracy.fit")])

    print("Merging following data into glb_models_df:")
    print(stats_mrg_df <- stats_df[, c(1, grep(glb_model_metric, names(stats_df)))])
    print(tmp_models_df <- orderBy(~model_id, glb_models_df[, c("model_id", grep(glb_model_metric, names(stats_df), value=TRUE))]))

    tmp2_models_df <- glb_models_df[, c("model_id", setdiff(names(glb_models_df), grep(glb_model_metric, names(stats_df), value=TRUE)))]
    tmp3_models_df <- merge(tmp2_models_df, stats_mrg_df, all.x=TRUE, sort=FALSE)
    print(tmp3_models_df)
    print(names(tmp3_models_df))
    print(glb_models_df <- subset(tmp3_models_df, select=-model_id.1))
}

plt_models_df <- glb_models_df[, -grep("SD|Upper|Lower", names(glb_models_df))]
for (var in grep("^min.", names(plt_models_df), value=TRUE)) {
    plt_models_df[, sub("min.", "inv.", var)] <- 
        #ifelse(all(is.na(tmp <- plt_models_df[, var])), NA, 1.0 / tmp)
        1.0 / plt_models_df[, var]
    plt_models_df <- plt_models_df[ , -grep(var, names(plt_models_df))]
}
print(plt_models_df)
```

```
##                  model_id model_method
## 1                  MFO.lm           lm
## 2               latlon.lm           lm
## 3            latlon.rpart        rpart
## 4       Max.cor.Y.cv.0.lm           lm
## 5       Max.cor.Y.cv.G.lm           lm
## 6  Interact.High.cor.y.lm           lm
## 7            Low.cor.X.lm           lm
## 8           All.X.cv.0.lm           lm
## 9                All.X.lm           lm
## 10              All.X.glm          glm
## 11            All.X.rpart        rpart
## 12               All.X.rf           rf
##                                                                            feats
## 1                                                                         .rnorm
## 2                                                                       LAT, LON
## 3                                                                       LAT, LON
## 4                                                                             RM
## 5                                                                             RM
## 6                                             RM, RM:DIS, RM:RAD, RM:NOX, RM:TAX
## 7                      RM, ZN, CHAS, .rnorm, LAT, LON, AGE, CRIM, INDUS, PTRATIO
## 8          LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO
## 9  LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
## 10 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
## 11         LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO
## 12 LON, LAT, CRIM, ZN, INDUS, CHAS, NOX, RM, AGE, DIS, RAD, TAX, PTRATIO, .rnorm
##    max.nTuningRuns max.R.sq.fit max.R.sq.OOB max.Adj.R.sq.fit
## 1                0 0.0006593684 -0.002734968     -0.002101241
## 2                0 0.1071648958           NA      0.103614856
## 3                0 0.5635405103           NA               NA
## 4                0 0.5022979132  0.422965492      0.500923046
## 5                1 0.5022979132  0.422965492      0.500923046
## 6                1 0.6029820329  0.591061401      0.597437089
## 7                1 0.6253946970  0.648918353      0.614782649
## 8                0 0.6649543413  0.694936215      0.652509788
## 9                1 0.6654907244  0.695457041      0.652072014
## 10               1 0.6654907244  0.695457041               NA
## 11              10 0.8623548418  0.630783469               NA
## 12               3 0.9619152730  0.794064554               NA
##    max.Rsquared.fit inv.elapsedtime.everything inv.elapsedtime.final
## 1                NA                  1.6528926            250.000000
## 2                NA                  2.4154589            333.333333
## 3                NA                  1.8761726             55.555556
## 4                NA                  2.3201856            500.000000
## 5         0.5335055                  1.2422360            333.333333
## 6         0.6342187                  1.0193680            250.000000
## 7         0.6393438                  1.2610340            250.000000
## 8                NA                  2.4154589            200.000000
## 9         0.6648438                  1.2406948            166.666667
## 10        0.6648438                  1.0373444             50.000000
## 11        0.7465918                  0.6439150             40.000000
## 12        0.7960880                  0.1948558              1.040583
##    inv.RMSE.fit inv.RMSE.OOB  inv.aic.fit
## 1     0.1055323    0.1192663           NA
## 2     0.1153714           NA           NA
## 3     0.1650107           NA           NA
## 4     0.1495400    0.1572208           NA
## 5     0.1509324    0.1572208           NA
## 6     0.1704103    0.1867591           NA
## 7     0.1725980    0.2015611           NA
## 8     0.1822594    0.2162298           NA
## 9     0.1804411    0.2164146           NA
## 10    0.1804411    0.2164146 0.0004340851
## 11    0.2170561    0.1965487           NA
## 12    0.2336257    0.2633532           NA
```

```r
print(myplot_radar(radar_inp_df=plt_models_df))
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

```
## Warning: Removed 10 rows containing missing values (geom_path).
```

```
## Warning: Removed 69 rows containing missing values (geom_point).
```

```
## Warning: Removed 24 rows containing missing values (geom_text).
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

![](Boston_Housing_files/figure-html/fit.models-42.png) 

```r
# print(myplot_radar(radar_inp_df=subset(plt_models_df, 
#         !(model_id %in% grep("random|MFO", plt_models_df$model_id, value=TRUE)))))

# Compute CI for <metric>SD
glb_models_df <- mutate(glb_models_df, 
                max.df = ifelse(max.nTuningRuns > 1, max.nTuningRuns - 1, NA),
                min.sd2ci.scaler = ifelse(is.na(max.df), NA, qt(0.975, max.df)))
for (var in grep("SD", names(glb_models_df), value=TRUE)) {
    # Does CI alredy exist ?
    var_components <- unlist(strsplit(var, "SD"))
    varActul <- paste0(var_components[1],          var_components[2])
    varUpper <- paste0(var_components[1], "Upper", var_components[2])
    varLower <- paste0(var_components[1], "Lower", var_components[2])
    if (varUpper %in% names(glb_models_df)) {
        warning(varUpper, " already exists in glb_models_df")
        # Assuming Lower also exists
        next
    }    
    print(sprintf("var:%s", var))
    # CI is dependent on sample size in t distribution; df=n-1
    glb_models_df[, varUpper] <- glb_models_df[, varActul] + 
        glb_models_df[, "min.sd2ci.scaler"] * glb_models_df[, var]
    glb_models_df[, varLower] <- glb_models_df[, varActul] - 
        glb_models_df[, "min.sd2ci.scaler"] * glb_models_df[, var]
}
```

```
## [1] "var:min.RMSESD.fit"
## [1] "var:max.RsquaredSD.fit"
```

```r
# Plot metrics with CI
plt_models_df <- glb_models_df[, "model_id", FALSE]
pltCI_models_df <- glb_models_df[, "model_id", FALSE]
for (var in grep("Upper", names(glb_models_df), value=TRUE)) {
    var_components <- unlist(strsplit(var, "Upper"))
    col_name <- unlist(paste(var_components, collapse=""))
    plt_models_df[, col_name] <- glb_models_df[, col_name]
    for (name in paste0(var_components[1], c("Upper", "Lower"), var_components[2]))
        pltCI_models_df[, name] <- glb_models_df[, name]
}

build_statsCI_data <- function(plt_models_df) {
    mltd_models_df <- melt(plt_models_df, id.vars="model_id")
    mltd_models_df$data <- sapply(1:nrow(mltd_models_df), 
        function(row_ix) tail(unlist(strsplit(as.character(
            mltd_models_df[row_ix, "variable"]), "[.]")), 1))
    mltd_models_df$label <- sapply(1:nrow(mltd_models_df), 
        function(row_ix) head(unlist(strsplit(as.character(
            mltd_models_df[row_ix, "variable"]), paste0(".", mltd_models_df[row_ix, "data"]))), 1))
    #print(mltd_models_df)
    
    return(mltd_models_df)
}
mltd_models_df <- build_statsCI_data(plt_models_df)

mltdCI_models_df <- melt(pltCI_models_df, id.vars="model_id")
for (row_ix in 1:nrow(mltdCI_models_df)) {
    for (type in c("Upper", "Lower")) {
        if (length(var_components <- unlist(strsplit(
                as.character(mltdCI_models_df[row_ix, "variable"]), type))) > 1) {
            #print(sprintf("row_ix:%d; type:%s; ", row_ix, type))
            mltdCI_models_df[row_ix, "label"] <- var_components[1]
            mltdCI_models_df[row_ix, "data"] <- unlist(strsplit(var_components[2], "[.]"))[2]
            mltdCI_models_df[row_ix, "type"] <- type
            break
        }
    }    
}
#print(mltdCI_models_df)
# castCI_models_df <- dcast(mltdCI_models_df, value ~ type, fun.aggregate=sum)
# print(castCI_models_df)
wideCI_models_df <- reshape(subset(mltdCI_models_df, select=-variable), 
                            timevar="type", 
        idvar=setdiff(names(mltdCI_models_df), c("type", "value", "variable")), 
                            direction="wide")
#print(wideCI_models_df)
mrgdCI_models_df <- merge(wideCI_models_df, mltd_models_df, all.x=TRUE)
#print(mrgdCI_models_df)

# Merge stats back in if CIs don't exist
goback_vars <- c()
for (var in unique(mltd_models_df$label)) {
    for (type in unique(mltd_models_df$data)) {
        var_type <- paste0(var, ".", type)
        # if this data is already present, next
        if (var_type %in% unique(paste(mltd_models_df$label, mltd_models_df$data, sep=".")))
            next
        #print(sprintf("var_type:%s", var_type))
        goback_vars <- c(goback_vars, var_type)
    }
}

if (length(goback_vars) > 0) {
    mltd_goback_df <- build_statsCI_data(glb_models_df[, c("model_id", goback_vars)])
    mltd_models_df <- rbind(mltd_models_df, mltd_goback_df)
}

mltd_models_df <- merge(mltd_models_df, glb_models_df[, c("model_id", "model_method")], all.x=TRUE)

# print(myplot_bar(mltd_models_df, "model_id", "value", colorcol_name="data") + 
#         geom_errorbar(data=mrgdCI_models_df, 
#             mapping=aes(x=model_id, ymax=value.Upper, ymin=value.Lower), width=0.5) + 
#           facet_grid(label ~ data, scales="free") + 
#           theme(axis.text.x = element_text(angle = 45,vjust = 1)))
# mltd_models_df <- orderBy(~ value +variable +data +label + model_method + model_id, 
#                           mltd_models_df)
print(myplot_bar(mltd_models_df, "model_id", "value", colorcol_name="model_method") + 
        geom_errorbar(data=mrgdCI_models_df, 
            mapping=aes(x=model_id, ymax=value.Upper, ymin=value.Lower), width=0.5) + 
          facet_grid(label ~ data, scales="free") + 
          theme(axis.text.x = element_text(angle = 90,vjust = 0.5)))
```

```
## Warning: Removed 5 rows containing missing values (position_stack).
```

![](Boston_Housing_files/figure-html/fit.models-43.png) 

```r
model_evl_terms <- c(NULL)
for (metric in glb_model_evl_criteria)
    model_evl_terms <- c(model_evl_terms, 
                    ifelse(length(grep("max", metric)) > 0, "-", "+"), metric)
model_sel_frmla <- as.formula(paste(c("~ ", model_evl_terms), collapse=" "))
print(tmp_models_df <- orderBy(model_sel_frmla, glb_models_df)[, c("model_id", glb_model_evl_criteria)])
```

```
##                  model_id min.RMSE.OOB max.R.sq.OOB max.Adj.R.sq.fit
## 12               All.X.rf     3.797181  0.794064554               NA
## 9                All.X.lm     4.620761  0.695457041      0.652072014
## 10              All.X.glm     4.620761  0.695457041               NA
## 8           All.X.cv.0.lm     4.624710  0.694936215      0.652509788
## 7            Low.cor.X.lm     4.961275  0.648918353      0.614782649
## 11            All.X.rpart     5.087797  0.630783469               NA
## 6  Interact.High.cor.y.lm     5.354492  0.591061401      0.597437089
## 4       Max.cor.Y.cv.0.lm     6.360483  0.422965492      0.500923046
## 5       Max.cor.Y.cv.G.lm     6.360483  0.422965492      0.500923046
## 1                  MFO.lm     8.384599 -0.002734968     -0.002101241
## 2               latlon.lm           NA           NA      0.103614856
## 3            latlon.rpart           NA           NA               NA
##    min.aic.fit
## 12          NA
## 9           NA
## 10    2303.696
## 8           NA
## 7           NA
## 11          NA
## 6           NA
## 4           NA
## 5           NA
## 1           NA
## 2           NA
## 3           NA
```

```r
print(myplot_radar(radar_inp_df=tmp_models_df))
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

```
## Warning: Removed 11 rows containing missing values (geom_path).
```

```
## Warning: Removed 32 rows containing missing values (geom_point).
```

```
## Warning: Removed 19 rows containing missing values (geom_text).
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

![](Boston_Housing_files/figure-html/fit.models-44.png) 

```r
print("Metrics used for model selection:"); print(model_sel_frmla)
```

```
## [1] "Metrics used for model selection:"
```

```
## ~+min.RMSE.OOB - max.R.sq.OOB - max.Adj.R.sq.fit + min.aic.fit
```

```r
print(sprintf("Best model id: %s", tmp_models_df[1, "model_id"]))
```

```
## [1] "Best model id: All.X.rf"
```

```r
if (is.null(glb_sel_mdl_id)) 
    { glb_sel_mdl_id <- tmp_models_df[1, "model_id"] } else 
        print(sprintf("User specified selection: %s", glb_sel_mdl_id))   

myprint_mdl(glb_sel_mdl <- glb_models_lst[[glb_sel_mdl_id]])
```

![](Boston_Housing_files/figure-html/fit.models-45.png) 

```
##                 Length Class      Mode     
## call              4    -none-     call     
## type              1    -none-     character
## predicted       364    -none-     numeric  
## mse             500    -none-     numeric  
## rsq             500    -none-     numeric  
## oob.times       364    -none-     numeric  
## importance       14    -none-     numeric  
## importanceSD      0    -none-     NULL     
## localImportance   0    -none-     NULL     
## proximity         0    -none-     NULL     
## ntree             1    -none-     numeric  
## mtry              1    -none-     numeric  
## forest           11    -none-     list     
## coefs             0    -none-     NULL     
## y               364    -none-     numeric  
## test              0    -none-     NULL     
## inbag             0    -none-     NULL     
## xNames           14    -none-     character
## problemType       1    -none-     character
## tuneValue         1    data.frame list     
## obsLevels         1    -none-     logical
```

```
## [1] TRUE
```

```r
replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "model.selected")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0 
## 2.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction data.new.prediction 	firing:  model.selected 
## 3.0000 	 3 	 0 2 1 0
```

![](Boston_Housing_files/figure-html/fit.models-46.png) 

```r
glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="fit.data.training.all", 
                              chunk_step_major=max(glb_script_df$chunk_step_major)+1, 
                              chunk_step_minor=0,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                    chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed8            fit.models                5                0   7.029
## elapsed9 fit.data.training.all                6                0  44.222
```

## Step `6`: fit.data.training.all

```r
if (!is.null(glb_fin_mdl_id) && (glb_fin_mdl_id %in% names(glb_models_lst))) {
    warning("Final model same as user selected model")
    glb_fin_mdl <- glb_sel_mdl
} else {    
    print(mdl_feats_df <- myextract_mdl_feats(sel_mdl=glb_sel_mdl, entity_df=glb_trnent_df))
    
    # Sync with parameters in mydsutils.R
    ret_lst <- myfit_mdl(model_id="Final", model_method=glb_sel_mdl$method,
                            indep_vars_vctr=mdl_feats_df$id, model_type=glb_model_type,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out, 
                            fit_df=glb_trnent_df, OOB_df=NULL,
                         # Automate from here
                         #  Issues if glb_sel_mdl$method == "rf" b/c trainControl is "oob"; not "cv"
                         n_cv_folds=glb_n_cv_folds, tune_models_df=NULL,
                            model_loss_mtrx=glb_model_metric_terms,
                            model_summaryFunction=glb_sel_mdl$control$summaryFunction,
                            model_metric=glb_sel_mdl$metric,
                            model_metric_maximize=glb_sel_mdl$maximize)
    glb_fin_mdl <- glb_models_lst[[length(glb_models_lst)]] 
}
```

```
##          importance      id fit.feat
## RM      100.0000000      RM     TRUE
## LON      15.9025634     LON     TRUE
## DIS      15.6103526     DIS     TRUE
## CRIM     13.6693313    CRIM     TRUE
## NOX      13.5587394     NOX     TRUE
## INDUS    13.2350664   INDUS     TRUE
## PTRATIO   8.6891243 PTRATIO     TRUE
## AGE       6.2746862     AGE     TRUE
## TAX       3.4886430     TAX     TRUE
## LAT       2.8376951     LAT     TRUE
## .rnorm    2.7975689  .rnorm     TRUE
## ZN        0.7252122      ZN     TRUE
## RAD       0.7135818     RAD     TRUE
## CHAS      0.5521026    CHAS     TRUE
## [1] "fitting model: Final.rf"
## [1] "    indep_vars: RM, LON, DIS, CRIM, NOX, INDUS, PTRATIO, AGE, TAX, LAT, .rnorm, ZN, RAD, CHAS"
## + : mtry= 2 
## - : mtry= 2 
## + : mtry= 8 
## - : mtry= 8 
## + : mtry=14 
## - : mtry=14 
## Aggregating results
## Selecting tuning parameters
## Fitting mtry = 8 on full training set
```

![](Boston_Housing_files/figure-html/fit.data.training.all_0-1.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_0-2.png) 

```
##                 Length Class      Mode     
## call              4    -none-     call     
## type              1    -none-     character
## predicted       364    -none-     numeric  
## mse             500    -none-     numeric  
## rsq             500    -none-     numeric  
## oob.times       364    -none-     numeric  
## importance       14    -none-     numeric  
## importanceSD      0    -none-     NULL     
## localImportance   0    -none-     NULL     
## proximity         0    -none-     NULL     
## ntree             1    -none-     numeric  
## mtry              1    -none-     numeric  
## forest           11    -none-     list     
## coefs             0    -none-     NULL     
## y               364    -none-     numeric  
## test              0    -none-     NULL     
## inbag             0    -none-     NULL     
## xNames           14    -none-     character
## problemType       1    -none-     character
## tuneValue         1    data.frame list     
## obsLevels         1    -none-     logical  
##   model_id model_method
## 1 Final.rf           rf
##                                                                           feats
## 1 RM, LON, DIS, CRIM, NOX, INDUS, PTRATIO, AGE, TAX, LAT, .rnorm, ZN, RAD, CHAS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                      5.023                 0.955
##   max.R.sq.fit min.RMSE.fit max.Rsquared.fit
## 1    0.9624498     4.309933        0.7932597
```

```r
glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="fit.data.training.all", 
    chunk_step_major=glb_script_df[nrow(glb_script_df), "chunk_step_major"], 
    chunk_step_minor=glb_script_df[nrow(glb_script_df), "chunk_step_minor"]+1,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                     chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed9  fit.data.training.all                6                0  44.222
## elapsed10 fit.data.training.all                6                1  55.433
```


```r
glb_rsp_var_out <- paste0(glb_rsp_var_out, tail(names(glb_models_lst), 1))
if (glb_is_regression) {
    glb_trnent_df[, glb_rsp_var_out] <- predict(glb_fin_mdl, newdata=glb_trnent_df, type="raw")
    print(myplot_scatter(glb_trnent_df, glb_rsp_var, glb_rsp_var_out, 
                         smooth=TRUE))
    glb_trnent_df[, paste0(glb_rsp_var_out, ".err")] <- 
        abs(glb_trnent_df[, glb_rsp_var_out] - glb_trnent_df[, glb_rsp_var])
    print(head(orderBy(reformulate(c("-", paste0(glb_rsp_var_out, ".err"))), 
                       glb_trnent_df)))                             
}    
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

![](Boston_Housing_files/figure-html/fit.data.training.all_1-1.png) 

```
##                   TOWN TRACT      LON     LAT MEDV     CRIM ZN INDUS CHAS
## 369    Boston Back Bay   107 -71.0480 42.2105 50.0  4.89822  0 18.10    0
## 373 Boston Beacon Hill   203 -71.0397 42.2182 50.0  8.26725  0 18.10    1
## 372 Boston Beacon Hill   202 -71.0417 42.2160 50.0  9.23230  0 18.10    0
## 407    Boston Downtown   702 -71.0390 42.2198 11.9 20.71620  0 18.10    0
## 375   Boston North End   302 -71.0346 42.2187 13.8 18.49820  0 18.10    0
## 158          Cambridge  3536 -71.0690 42.2285 41.3  1.22358  0 19.58    0
##       NOX    RM   AGE    DIS RAD TAX PTRATIO          TOWN.fctr     .rnorm
## 369 0.631 4.970 100.0 1.3325  24 666    20.2    Boston Back Bay -0.4580181
## 373 0.668 5.875  89.6 1.1296  24 666    20.2 Boston Beacon Hill -1.1368931
## 372 0.631 6.216 100.0 1.1691  24 666    20.2 Boston Beacon Hill  0.9356037
## 407 0.659 4.138 100.0 1.1781  24 666    20.2    Boston Downtown  0.7163316
## 375 0.668 4.138 100.0 1.1370  24 666    20.2   Boston North End  0.2669183
## 158 0.605 6.943  97.4 1.8773   5 403    14.7          Cambridge  1.9507210
##     MEDV.predict.Final.rf MEDV.predict.Final.rf.err
## 369              35.62560                 14.374403
## 373              38.94893                 11.051070
## 372              39.02441                 10.975593
## 407              20.60228                  8.702283
## 375              21.94827                  8.148270
## 158              33.66870                  7.631297
```

```r
if (glb_is_classification && glb_is_binomial) {
    stop("not implemented")
            if (any(class(glb_fin_mdl) %in% c("train"))) {
        glb_trnent_df[, paste0(glb_rsp_var_out, ".proba")] <- 
            predict(glb_fin_mdl, newdata=glb_trnent_df, type="prob")[, 2]
    } else  if (any(class(glb_fin_mdl) %in% c("rpart", "randomForest"))) {
        glb_trnent_df[, paste0(glb_rsp_var_out, ".proba")] <- 
            predict(glb_fin_mdl, newdata=glb_trnent_df, type="prob")[, 2]
    } else  if (class(glb_fin_mdl) == "glm") {
        stop("not implemented yet")
        glb_trnent_df[, paste0(glb_rsp_var_out, ".proba")] <- 
            predict(glb_fin_mdl, newdata=glb_trnent_df, type="response")
    } else  stop("not implemented yet")   

    require(ROCR)
    ROCRpred <- prediction(glb_trnent_df[, paste0(glb_rsp_var_out, ".proba")],
                           glb_trnent_df[, glb_rsp_var])
    ROCRperf <- performance(ROCRpred, "tpr", "fpr")
    plot(ROCRperf, colorize=TRUE, print.cutoffs.at=seq(0, 1, 0.1), text.adj=c(-0.2,1.7))
    
    thresholds_df <- data.frame(threshold=seq(0.0, 1.0, 0.1))
    thresholds_df$f.score <- sapply(1:nrow(thresholds_df), function(row_ix) 
        mycompute_classifier_f.score(mdl=glb_fin_mdl, obs_df=glb_trnent_df, 
                                     proba_threshold=thresholds_df[row_ix, "threshold"], 
                                      rsp_var=glb_rsp_var, 
                                      rsp_var_out=glb_rsp_var_out))
    print(thresholds_df)
    print(myplot_line(thresholds_df, "threshold", "f.score"))
    
    proba_threshold <- thresholds_df[which.max(thresholds_df$f.score), 
                                             "threshold"]
    # This should change to maximize f.score.OOB ???
    print(sprintf("Classifier Probability Threshold: %0.4f to maximize f.score.fit",
                  proba_threshold))
    if (is.null(glb_clf_proba_threshold)) 
        glb_clf_proba_threshold <- proba_threshold else {
        print(sprintf("Classifier Probability Threshold: %0.4f per user specs",
                      glb_clf_proba_threshold))
    }

    if ((class(glb_trnent_df[, glb_rsp_var]) != "factor") | 
    	(length(levels(glb_trnent_df[, glb_rsp_var])) != 2))
		stop("expecting a factor with two levels:", glb_rsp_var)
	glb_trnent_df[, glb_rsp_var_out] <- 
		factor(levels(glb_trnent_df[, glb_rsp_var])[
			(glb_trnent_df[, paste0(glb_rsp_var_out, ".proba")] >= 
				glb_clf_proba_threshold) * 1 + 1])
             
    print(mycreate_xtab(glb_trnent_df, c(glb_rsp_var, glb_rsp_var_out)))
    print(sprintf("f.score=%0.4f", 
        mycompute_classifier_f.score(glb_fin_mdl, glb_trnent_df, 
                                     glb_clf_proba_threshold, 
                                     glb_rsp_var, glb_rsp_var_out)))    
}    

if (glb_is_classification && !glb_is_binomial) {
    glb_trnent_df[, glb_rsp_var_out] <- predict(glb_fin_mdl, newdata=glb_trnent_df, type="raw")
}    

print(glb_feats_df <- mymerge_feats_importance(feats_df=glb_feats_df, sel_mdl=glb_fin_mdl, 
                                               entity_df=glb_trnent_df))
```

```
##           id       cor.y exclude.as.feat  cor.y.abs cor.low  importance
## 12        RM  0.70872979               0 0.70872979       1 100.0000000
## 5        DIS  0.25834984               0 0.25834984       0  15.7636765
## 8        LON -0.33341021               0 0.33341021       1  15.4028402
## 4       CRIM -0.38466671               0 0.38466671       1  14.5478310
## 9        NOX -0.43202955               0 0.43202955       0  13.2532280
## 6      INDUS -0.49397282               0 0.49397282       1  11.4200253
## 10   PTRATIO -0.50539619               0 0.50539619       1   8.7681059
## 2        AGE -0.37857924               0 0.37857924       1   6.5712488
## 13       TAX -0.47940636               0 0.47940636       0   3.5652129
## 7        LAT  0.01854813               0 0.01854813       1   2.6626179
## 1     .rnorm  0.02567817               0 0.02567817       1   2.3754815
## 11       RAD -0.40830448               0 0.40830448       0   0.7947197
## 3       CHAS  0.16496009               0 0.16496009       1   0.4565189
## 16        ZN  0.36446804               0 0.36446804       1   0.4217223
## 14 TOWN.fctr -0.28659819               1 0.28659819       0          NA
## 15     TRACT  0.44111242               1 0.44111242       0          NA
```

```r
# Most of this code is used again in predict.data.new chunk
glb_analytics_diag_plots <- function(obs_df) {
    for (var in subset(glb_feats_df, !is.na(importance))$id) {
        plot_df <- melt(obs_df, id.vars=var, 
                        measure.vars=c(glb_rsp_var, glb_rsp_var_out))
#         if (var == "<feat_name>") print(myplot_scatter(plot_df, var, "value", 
#                                              facet_colcol_name="variable") + 
#                       geom_vline(xintercept=<divider_val>, linetype="dotted")) else     
            print(myplot_scatter(plot_df, var, "value", colorcol_name="variable",
                                 facet_colcol_name="variable", jitter=TRUE) + 
                      guides(color=FALSE))
    }
    
    if (glb_is_regression) {
        plot_vars_df <- subset(glb_feats_df, importance > glb_feats_df[glb_feats_df$id == ".rnorm", "importance"])
        print(myplot_prediction_regression(df=obs_df, 
                    feat_x=ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"), 
                                           feat_y=plot_vars_df$id[1],
                    rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                    id_vars=glb_id_vars)
#               + facet_wrap(reformulate(plot_vars_df$id[2])) # if [1 or 2] is a factor                                                         
#               + geom_point(aes_string(color="<col_name>.fctr")) #  to color the plot
              )
    }    
    
    if (glb_is_classification) {
        if (nrow(plot_vars_df <- subset(glb_feats_df, !is.na(importance))) == 0)
            warning("No features in selected model are statistically important")
        else print(myplot_prediction_classification(df=obs_df, 
                feat_x=ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], 
                              ".rownames"),
                                               feat_y=plot_vars_df$id[1],
                     rsp_var=glb_rsp_var, 
                     rsp_var_out=glb_rsp_var_out, 
                     id_vars=glb_id_vars)
#               + geom_hline(yintercept=<divider_val>, linetype = "dotted")
                )
    }    
}
glb_analytics_diag_plots(obs_df=glb_trnent_df)
```

![](Boston_Housing_files/figure-html/fit.data.training.all_1-2.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-3.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-4.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-5.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-6.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-7.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-8.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-9.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-10.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-11.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-12.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-13.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-14.png) ![](Boston_Housing_files/figure-html/fit.data.training.all_1-15.png) 

```
##                   TOWN TRACT      LON     LAT MEDV     CRIM ZN INDUS CHAS
## 369    Boston Back Bay   107 -71.0480 42.2105 50.0  4.89822  0  18.1    0
## 373 Boston Beacon Hill   203 -71.0397 42.2182 50.0  8.26725  0  18.1    1
## 372 Boston Beacon Hill   202 -71.0417 42.2160 50.0  9.23230  0  18.1    0
## 407    Boston Downtown   702 -71.0390 42.2198 11.9 20.71620  0  18.1    0
## 375   Boston North End   302 -71.0346 42.2187 13.8 18.49820  0  18.1    0
##       NOX    RM   AGE    DIS RAD TAX PTRATIO          TOWN.fctr     .rnorm
## 369 0.631 4.970 100.0 1.3325  24 666    20.2    Boston Back Bay -0.4580181
## 373 0.668 5.875  89.6 1.1296  24 666    20.2 Boston Beacon Hill -1.1368931
## 372 0.631 6.216 100.0 1.1691  24 666    20.2 Boston Beacon Hill  0.9356037
## 407 0.659 4.138 100.0 1.1781  24 666    20.2    Boston Downtown  0.7163316
## 375 0.668 4.138 100.0 1.1370  24 666    20.2   Boston North End  0.2669183
##     MEDV.predict.Final.rf MEDV.predict.Final.rf.err .label
## 369              35.62560                 14.374403    107
## 373              38.94893                 11.051070    203
## 372              39.02441                 10.975593    202
## 407              20.60228                  8.702283    702
## 375              21.94827                  8.148270    302
```

![](Boston_Housing_files/figure-html/fit.data.training.all_1-16.png) 

```r
replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "data.training.all.prediction","model.final")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0 
## 2.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction data.new.prediction 	firing:  model.selected 
## 3.0000 	 3 	 0 2 1 0 
## 3.0000 	multiple enabled transitions:  model.final data.training.all.prediction data.new.prediction 	firing:  data.training.all.prediction 
## 4.0000 	 5 	 0 1 1 1 
## 4.0000 	multiple enabled transitions:  model.final data.training.all.prediction data.new.prediction 	firing:  model.final 
## 5.0000 	 4 	 0 0 2 1
```

![](Boston_Housing_files/figure-html/fit.data.training.all_1-17.png) 

```r
glb_script_df <- rbind(glb_script_df, 
                   data.frame(chunk_label="predict.data.new", 
                              chunk_step_major=max(glb_script_df$chunk_step_major)+1, 
                              chunk_step_minor=0,
                              elapsed=(proc.time() - glb_script_tm)["elapsed"]))
print(tail(glb_script_df, 2))
```

```
##                     chunk_label chunk_step_major chunk_step_minor elapsed
## elapsed10 fit.data.training.all                6                1  55.433
## elapsed11      predict.data.new                7                0  63.173
```

## Step `7`: predict data.new

```r
if (glb_is_regression)
    glb_newent_df[, glb_rsp_var_out] <- 
        mypredict_mdl(glb_fin_mdl, glb_newent_df, glb_rsp_var, glb_rsp_var_out, 
                      "Final", "Final",
                				glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="raw")

if (glb_is_classification && glb_is_binomial) {
    # Compute selected model predictions
            if (any(class(glb_fin_mdl) %in% c("train"))) {
        glb_newent_df[, paste0(glb_rsp_var_out, ".proba")] <- 
            predict(glb_fin_mdl, newdata=glb_newent_df, type="prob")[, 2]
    } else  if (any(class(glb_fin_mdl) %in% c("rpart", "randomForest"))) {
        glb_newent_df[, paste0(glb_rsp_var_out, ".proba")] <- 
            predict(glb_fin_mdl, newdata=glb_newent_df, type="prob")[, 2]
    } else  if (class(glb_fin_mdl) == "glm") {
        stop("not implemented yet")
        glb_newent_df[, paste0(glb_rsp_var_out, ".proba")] <- 
            predict(glb_fin_mdl, newdata=glb_newent_df, type="response")
    } else  stop("not implemented yet")   

    if ((class(glb_newent_df[, glb_rsp_var]) != "factor") | 
		(length(levels(glb_newent_df[, glb_rsp_var])) != 2))
		stop("expecting a factor with two levels:", glb_rsp_var)
	glb_newent_df[, glb_rsp_var_out] <- 
		factor(levels(glb_newent_df[, glb_rsp_var])[
			(glb_newent_df[, paste0(glb_rsp_var_out, ".proba")] >= 
				glb_clf_proba_threshold) * 1 + 1])

    # Compute dummy model predictions
    glb_newent_df[, paste0(glb_rsp_var, ".predictdmy.proba")] <- 
        predict(glb_dmy_mdl, newdata=glb_newent_df, type="prob")[, 2]
    if ((class(glb_newent_df[, glb_rsp_var]) != "factor") | 
    	(length(levels(glb_newent_df[, glb_rsp_var])) != 2))
		stop("expecting a factor with two levels:", glb_rsp_var)
	glb_newent_df[, paste0(glb_rsp_var, ".predictdmy")] <- 
		factor(levels(glb_newent_df[, glb_rsp_var])[
			(glb_newent_df[, paste0(glb_rsp_var, ".predictdmy.proba")] >= 
				glb_clf_proba_threshold) * 1 + 1])
}

if (glb_is_classification && !glb_is_binomial) {
    # Compute final model predictions
    # glb_rsp_var_out <- paste0(glb_rsp_var_out, "Final") Already done in earlier chunk ???
	glb_newent_df[, glb_rsp_var_out] <- 
        mypredict_mdl(glb_fin_mdl, glb_newent_df, glb_rsp_var, glb_rsp_var_out, 
                      "Final", "Final",
            					glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="raw")
}
    
myprint_df(glb_newent_df[, c(glb_id_vars, glb_rsp_var, glb_rsp_var_out)])
```

```
##    TRACT MEDV MEDV.predict.Final.rf
## 5   2032 36.2              33.82711
## 8   2042 22.1              18.09004
## 15  2052 18.2              17.22280
## 16  2053 19.9              19.78306
## 18  2055 17.5              17.37657
## 19  2056 20.2              19.11338
##     TRACT MEDV MEDV.predict.Final.rf
## 185  3576 26.4              26.61178
## 344  5011 23.9              27.50174
## 364     8 16.8              20.44164
## 382   407 10.9              11.06500
## 474  1201 29.8              26.38025
## 499  1706 21.2              20.99997
##     TRACT MEDV MEDV.predict.Final.rf
## 492  1605 13.6              16.03618
## 494  1701 21.8              20.71332
## 497  1704 19.7              18.60815
## 498  1705 18.3              19.69428
## 499  1706 21.2              20.99997
## 505  1804 22.0              22.99826
```

```r
if (glb_is_regression) {
    print(mypredict_mdl(glb_fin_mdl, glb_newent_df, glb_rsp_var, glb_rsp_var_out, 
                      "Final", "Final",
                    			glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="stats"))        
}                         
```

```
##   model_id max.R.sq.Final min.RMSE.Final
## 1    Final      0.7995176       3.747351
```

```r
if (glb_is_classification && glb_is_binomial) {
    ROCRpred <- prediction(glb_newent_df[, paste0(glb_rsp_var_out, ".proba")],
                           glb_newent_df[, glb_rsp_var])
    print(sprintf("auc=%0.4f", auc <- as.numeric(performance(ROCRpred, "auc")@y.values)))   
    
    print(sprintf("probability threshold=%0.4f", glb_clf_proba_threshold))
    print(newent_conf_df <- mycreate_xtab(glb_newent_df, 
                                        c(glb_rsp_var, glb_rsp_var_out)))
    print(sprintf("f.score.sel=%0.4f", 
        mycompute_classifier_f.score(mdl=glb_fin_mdl, obs_df=glb_newent_df, 
                                     proba_threshold=glb_clf_proba_threshold, 
                                      rsp_var=glb_rsp_var, 
                                      rsp_var_out=glb_rsp_var_out)))
    print(sprintf("sensitivity=%0.4f", newent_conf_df[2, 3] / 
                      (newent_conf_df[2, 3] + newent_conf_df[2, 2])))
    print(sprintf("specificity=%0.4f", newent_conf_df[1, 2] / 
                      (newent_conf_df[1, 2] + newent_conf_df[1, 3])))
    print(sprintf("accuracy=%0.4f", (newent_conf_df[1, 2] + newent_conf_df[2, 3]) / 
                      (newent_conf_df[1, 2] + newent_conf_df[2, 3] + 
                       newent_conf_df[1, 3] + newent_conf_df[2, 2])))
    
    print(mycreate_xtab(glb_newent_df, c(glb_rsp_var, paste0(glb_rsp_var, ".predictdmy"))))
    print(sprintf("f.score.dmy=%0.4f", 
        mycompute_classifier_f.score(mdl=glb_dmy_mdl, obs_df=glb_newent_df, 
                                     proba_threshold=glb_clf_proba_threshold, 
                                      rsp_var=glb_rsp_var, 
                                      rsp_var_out=paste0(glb_rsp_var, ".predictdmy"))))
}    
    
if (glb_is_classification && !glb_is_binomial) {
    print(mypredict_mdl(glb_fin_mdl, glb_newent_df, glb_rsp_var, glb_rsp_var_out, 
                      "Final", "Final",
                				glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="stats"))    
}    
    
glb_analytics_diag_plots(obs_df=glb_newent_df)
```

![](Boston_Housing_files/figure-html/predict.data.new-1.png) ![](Boston_Housing_files/figure-html/predict.data.new-2.png) ![](Boston_Housing_files/figure-html/predict.data.new-3.png) ![](Boston_Housing_files/figure-html/predict.data.new-4.png) ![](Boston_Housing_files/figure-html/predict.data.new-5.png) ![](Boston_Housing_files/figure-html/predict.data.new-6.png) ![](Boston_Housing_files/figure-html/predict.data.new-7.png) ![](Boston_Housing_files/figure-html/predict.data.new-8.png) ![](Boston_Housing_files/figure-html/predict.data.new-9.png) ![](Boston_Housing_files/figure-html/predict.data.new-10.png) ![](Boston_Housing_files/figure-html/predict.data.new-11.png) ![](Boston_Housing_files/figure-html/predict.data.new-12.png) ![](Boston_Housing_files/figure-html/predict.data.new-13.png) ![](Boston_Housing_files/figure-html/predict.data.new-14.png) 

```
##                   TOWN TRACT      LON     LAT MEDV    CRIM ZN INDUS CHAS
## 370    Boston Back Bay   108 -71.0497 42.2125 50.0 5.66998  0 18.10    1
## 371 Boston Beacon Hill   201 -71.0422 42.2144 50.0 6.53876  0 18.10    1
## 365    Boston Back Bay   101 -71.0590 42.2098 21.9 3.47428  0 18.10    1
## 143          Cambridge  3521 -71.0480 42.2222 13.4 3.32105  0 19.58    1
## 89             Melrose  3361 -71.0420 42.2796 23.6 0.05660  0  3.41    0
##       NOX    RM   AGE    DIS RAD TAX PTRATIO          TOWN.fctr     .rnorm
## 370 0.631 6.683  96.8 1.3567  24 666    20.2    Boston Back Bay  0.3623522
## 371 0.631 7.016  97.5 1.2024  24 666    20.2 Boston Beacon Hill -0.8525257
## 365 0.718 8.780  82.9 1.9047  24 666    20.2    Boston Back Bay  0.3983627
## 143 0.871 5.403 100.0 1.3216   5 403    14.7          Cambridge -0.9571705
## 89  0.489 7.007  86.3 3.4217   2 270    17.8            Melrose -0.3315458
##     MEDV.predict.Final.rf MEDV.predict.Final.rf.err .label
## 370              28.37394                 21.626057    108
## 371              30.83295                 19.167047    201
## 365              33.09059                 11.190593    101
## 143              23.81395                 10.413953   3521
## 89               32.46719                  8.867193   3361
```

![](Boston_Housing_files/figure-html/predict.data.new-15.png) 

```r
tmp_replay_lst <- replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "data.new.prediction")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0 
## 2.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction data.new.prediction 	firing:  model.selected 
## 3.0000 	 3 	 0 2 1 0 
## 3.0000 	multiple enabled transitions:  model.final data.training.all.prediction data.new.prediction 	firing:  data.training.all.prediction 
## 4.0000 	 5 	 0 1 1 1 
## 4.0000 	multiple enabled transitions:  model.final data.training.all.prediction data.new.prediction 	firing:  model.final 
## 5.0000 	 4 	 0 0 2 1 
## 6.0000 	 6 	 0 0 1 2
```

![](Boston_Housing_files/figure-html/predict.data.new-16.png) 

```r
print(ggplot.petrinet(tmp_replay_lst[["pn"]]) + coord_flip())
```

![](Boston_Housing_files/figure-html/predict.data.new-17.png) 

Null Hypothesis ($\sf{H_{0}}$): mpg is not impacted by am_fctr.  
The variance by am_fctr appears to be independent. 
#```{r q1, cache=FALSE}
# print(t.test(subset(cars_df, am_fctr == "automatic")$mpg, 
#              subset(cars_df, am_fctr == "manual")$mpg, 
#              var.equal=FALSE)$conf)
#```
We reject the null hypothesis i.e. we have evidence to conclude that am_fctr impacts mpg (95% confidence). Manual transmission is better for miles per gallon versus automatic transmission.


```
##                   chunk_label chunk_step_major chunk_step_minor elapsed
## 10      fit.data.training.all                6                0  44.222
## 11      fit.data.training.all                6                1  55.433
## 12           predict.data.new                7                0  63.173
## 9                  fit.models                5                0   7.029
## 4         manage_missing_data                2                2   2.415
## 7             select_features                4                0   3.799
## 5          encode_retype_data                2                3   2.949
## 2                cleanse_data                2                0   0.514
## 8  remove_correlated_features                4                1   3.983
## 6            extract_features                3                0   3.007
## 3       inspectORexplore.data                2                1   0.548
## 1                 import_data                1                0   0.002
##    elapsed_diff
## 10       37.193
## 11       11.211
## 12        7.740
## 9         3.046
## 4         1.867
## 7         0.792
## 5         0.534
## 2         0.512
## 8         0.184
## 6         0.058
## 3         0.034
## 1         0.000
```

```
## [1] "Total Elapsed Time: 63.173 secs"
```

![](Boston_Housing_files/figure-html/print_sessionInfo-1.png) 

```
## R version 3.1.3 (2015-03-09)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## Running under: OS X 10.10.3 (Yosemite)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] grid      stats     graphics  grDevices utils     datasets  methods  
## [8] base     
## 
## other attached packages:
##  [1] randomForest_4.6-10 rpart.plot_1.5.2    rpart_4.1-9        
##  [4] caret_6.0-41        lattice_0.20-31     reshape2_1.4.1     
##  [7] plyr_1.8.1          caTools_1.17.1      doBy_4.5-13        
## [10] survival_2.38-1     ggplot2_1.0.1      
## 
## loaded via a namespace (and not attached):
##  [1] bitops_1.0-6        BradleyTerry2_1.0-6 brglm_0.5-9        
##  [4] car_2.0-25          codetools_0.2-11    colorspace_1.2-6   
##  [7] compiler_3.1.3      digest_0.6.8        evaluate_0.5.5     
## [10] foreach_1.4.2       formatR_1.1         gtable_0.1.2       
## [13] gtools_3.4.1        htmltools_0.2.6     iterators_1.0.7    
## [16] knitr_1.9           labeling_0.3        lme4_1.1-7         
## [19] MASS_7.3-40         Matrix_1.2-0        mgcv_1.8-6         
## [22] minqa_1.2.4         munsell_0.4.2       nlme_3.1-120       
## [25] nloptr_1.0.4        nnet_7.3-9          parallel_3.1.3     
## [28] pbkrtest_0.4-2      proto_0.3-10        quantreg_5.11      
## [31] RColorBrewer_1.1-2  Rcpp_0.11.5         rmarkdown_0.5.1    
## [34] scales_0.2.4        SparseM_1.6         splines_3.1.3      
## [37] stringr_0.6.2       tools_3.1.3         yaml_2.1.13
```
