===
Prediction Assignment: Writeup
===
## Synopsis  

Using devices such as *Jawbone Up*, *Nike FuelBand*, and *Fitbit* is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement �V a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify *how well they do it.*  

In this project, we use data from 4 accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the **groupware** website: [http://groupware.les.inf.puc-rio.br/har](http://groupware.les.inf.puc-rio.br/har) (see the section on the Weight Lifting Exercise Dataset).  

In this report, you can see we transforming the training set (downloaded from the [course website](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv)) by reducing the predictors to 12 components and taking Gradient Boosting machine (GBM) methodology to predict the manner in which participants did the exercise. The result shows that the reduced 12 principal components could predict human actions. The accuracy rate is 79%.  

At the end, we take the prediction model to predict the test set which is also downloaded from the [course website](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv).  

## Data Processing
### Data Preparation


```r
train <- read.csv("pml-training.csv"); test <- read.csv("pml-testing.csv")
dim(train); summary(train[,160])
```

```
## [1] 19622   160
```

```
##    A    B    C    D    E 
## 5580 3797 3422 3216 3607
```

There are 160 variables in total. We don't need all of the variables, espicially for most of the statistical numbers having missing values. We only take the raw measured data to predict whether the exercises are exactly according to the specification (Class A), throwing the elbows to the front (Class B), lifting the dumbbell only halfway (Class C), lowering the dumbbell only halfway (Class D) or throwing the hips to the front (Class E).  

In particular, the input 52 predictors from the 4 sensors (belt, arm, dumbell and forearm) are:  
    - The sliding window records from 3 Euler angles (roll, pitch and yaw)  
    - The raw accelerometer, gyroscope and magnetometer readings  


```r
train <- train[,c("classe", "roll_belt", "pitch_belt", "yaw_belt", "total_accel_belt",
                  "gyros_belt_x", "gyros_belt_y", "gyros_belt_z",
                  "accel_belt_x", "accel_belt_y", "accel_belt_z",
                  "magnet_belt_x", "magnet_belt_y", "magnet_belt_z",
                  "roll_arm", "pitch_arm", "yaw_arm", "total_accel_arm",
                  "gyros_arm_x", "gyros_arm_y", "gyros_arm_z",
                  "accel_arm_x", "accel_arm_y", "accel_arm_z",
                  "magnet_arm_x", "magnet_arm_y", "magnet_arm_z",
                  "roll_dumbbell", "pitch_dumbbell", "yaw_dumbbell", "total_accel_dumbbell",
                  "gyros_dumbbell_x", "gyros_dumbbell_y", "gyros_dumbbell_z",
                  "accel_dumbbell_x", "accel_dumbbell_y", "accel_dumbbell_z",
                  "magnet_dumbbell_x", "magnet_dumbbell_y", "magnet_dumbbell_z",
                  "roll_forearm", "pitch_forearm", "yaw_forearm", "total_accel_forearm",
                  "gyros_forearm_x", "gyros_forearm_y", "gyros_forearm_z",
                  "accel_forearm_x", "accel_forearm_y", "accel_forearm_z",
                  "magnet_forearm_x", "magnet_forearm_y", "magnet_forearm_z")]
dim(train)
```

```
## [1] 19622    53
```

### Pre-processing  

In the very begining, we'd find if there is any high correlations between predictors.  


```r
M <- abs(cor(train[,-1]))
diag(M) <- 0
length(which(M > 0.8))/2; which(M > 0.8, arr.ind = TRUE)
```

```
## [1] 19
```

```
##                  row col
## yaw_belt           3   1
## total_accel_belt   4   1
## accel_belt_y       9   1
## accel_belt_z      10   1
## accel_belt_x       8   2
## magnet_belt_x     11   2
## roll_belt          1   3
## roll_belt          1   4
## accel_belt_y       9   4
## accel_belt_z      10   4
## pitch_belt         2   8
## magnet_belt_x     11   8
## roll_belt          1   9
## total_accel_belt   4   9
## accel_belt_z      10   9
## roll_belt          1  10
## total_accel_belt   4  10
## accel_belt_y       9  10
## pitch_belt         2  11
## accel_belt_x       8  11
## gyros_arm_y       19  18
## gyros_arm_x       18  19
## magnet_arm_x      24  21
## accel_arm_x       21  24
## magnet_arm_z      26  25
## magnet_arm_y      25  26
## accel_dumbbell_x  34  28
## accel_dumbbell_z  36  29
## gyros_dumbbell_z  33  31
## gyros_forearm_z   46  31
## gyros_dumbbell_x  31  33
## gyros_forearm_z   46  33
## pitch_dumbbell    28  34
## yaw_dumbbell      29  36
## gyros_forearm_z   46  45
## gyros_dumbbell_x  31  46
## gyros_dumbbell_z  33  46
## gyros_forearm_y   45  46
```

19 variables have high correlation with some others.  

For there are too many predictors and more than one third of them are correlated with others. We need to reduce the number of predictors statistically.  


```r
library(caret)
preproc <- preProcess(train[, -1], method = "pca", thresh = .8)
preproc
```

```
## 
## Call:
## preProcess.default(x = train[, -1], method = "pca", thresh = 0.8)
## 
## Created from 19622 samples and 52 variables
## Pre-processing: principal component signal extraction, scaled, centered 
## 
## PCA needed 12 components to capture 80 percent of the variance
```

```r
round(preproc$rotation, 4)
```

```
##                          PC1     PC2     PC3     PC4     PC5     PC6
## roll_belt            -0.3077  0.1301 -0.0756  0.0228  0.0109 -0.0168
## pitch_belt           -0.0239 -0.2920 -0.0748  0.0354 -0.1022  0.1718
## yaw_belt             -0.2011  0.2528 -0.0205 -0.0020  0.0540 -0.1144
## total_accel_belt     -0.3042  0.1105 -0.0970  0.0255 -0.0118 -0.0166
## gyros_belt_x          0.0943  0.1918  0.1968 -0.0365  0.0995  0.1745
## gyros_belt_y         -0.1033  0.2075  0.0833 -0.0324  0.0621  0.1517
## gyros_belt_z          0.1796  0.0461  0.1071 -0.0387  0.0476  0.1181
## accel_belt_x          0.0089  0.2943  0.0919 -0.0372  0.1247 -0.1585
## accel_belt_y         -0.3168  0.0383 -0.1051  0.0335 -0.0224  0.0325
## accel_belt_z          0.3166 -0.1058  0.0712 -0.0234 -0.0279  0.0192
## magnet_belt_x        -0.0162  0.2899  0.0473 -0.0248  0.1185 -0.1825
## magnet_belt_y         0.1164  0.0871 -0.0787  0.0104 -0.2696  0.1540
## magnet_belt_z         0.0596  0.1203 -0.0647  0.0102 -0.2425  0.1775
## roll_arm              0.0628 -0.1781  0.0629 -0.0296  0.0909 -0.2156
## pitch_arm             0.0366  0.0686 -0.2297  0.0635  0.1993  0.0390
## yaw_arm               0.0509 -0.1183  0.0060 -0.0132  0.1120 -0.1370
## total_accel_arm       0.1112 -0.0371  0.0576  0.0057 -0.0730 -0.0215
## gyros_arm_x          -0.0113  0.0513  0.0013 -0.0036  0.0162 -0.0394
## gyros_arm_y           0.0757 -0.0803 -0.0056  0.0030  0.0037  0.0515
## gyros_arm_z          -0.1573  0.1844  0.0697 -0.0113  0.0164  0.0846
## accel_arm_x          -0.1612 -0.1081  0.1711 -0.0544 -0.2670 -0.2169
## accel_arm_y           0.2689 -0.1245 -0.1311  0.0202  0.1216 -0.0040
## accel_arm_z          -0.1265 -0.0033 -0.2755  0.0564  0.1755  0.0549
## magnet_arm_x         -0.0908 -0.0110  0.2660 -0.0771 -0.2448 -0.1054
## magnet_arm_y          0.0658  0.0271 -0.3696  0.0873  0.2011  0.1074
## magnet_arm_z          0.0326  0.0273 -0.3074  0.0741  0.2768  0.1569
## roll_dumbbell         0.0869  0.1292  0.0605 -0.0270 -0.0780 -0.0598
## pitch_dumbbell       -0.1093 -0.1489  0.0978 -0.0358  0.0827 -0.0772
## yaw_dumbbell         -0.1245 -0.2665  0.0045  0.0128  0.0219  0.0200
## total_accel_dumbbell  0.1684  0.1478 -0.1210  0.0352 -0.1544 -0.1241
## gyros_dumbbell_x     -0.0034 -0.0115 -0.1245 -0.4507 -0.0295  0.0041
## gyros_dumbbell_y     -0.0011  0.0425  0.0796  0.3519 -0.0283  0.0219
## gyros_dumbbell_z     -0.0003  0.0065  0.1062  0.4523  0.0195 -0.0177
## accel_dumbbell_x     -0.1702 -0.1389  0.1358 -0.0488  0.1659 -0.0445
## accel_dumbbell_y      0.1815  0.1816 -0.0020 -0.0053 -0.1368 -0.0900
## accel_dumbbell_z     -0.1536 -0.2475  0.0701  0.0074  0.1467 -0.0157
## magnet_dumbbell_x    -0.1688 -0.1940 -0.1435  0.0316 -0.0551 -0.2071
## magnet_dumbbell_y     0.1457  0.1705  0.2061 -0.0416  0.0465  0.2154
## magnet_dumbbell_z     0.1706 -0.0219  0.1896 -0.0638  0.2476 -0.2196
## roll_forearm          0.0648 -0.0451 -0.1504  0.0241 -0.1813 -0.1485
## pitch_forearm        -0.1453 -0.1034  0.0987 -0.0221 -0.0755  0.0971
## yaw_forearm           0.1139 -0.0360 -0.1286  0.0257 -0.0485 -0.2837
## total_accel_forearm  -0.0007  0.0980  0.0029  0.0155  0.0310 -0.1994
## gyros_forearm_x      -0.0698  0.1946 -0.0960 -0.1733  0.0369 -0.1349
## gyros_forearm_y      -0.0035  0.0210  0.0936  0.4092 -0.0181 -0.0272
## gyros_forearm_z       0.0021  0.0267  0.1098  0.4534  0.0176 -0.0703
## accel_forearm_x       0.1920 -0.0883 -0.1237  0.0127 -0.0165 -0.1816
## accel_forearm_y       0.0349  0.0944 -0.1137  0.0321  0.0145 -0.4014
## accel_forearm_z      -0.0313  0.0380  0.2147 -0.0582  0.3344 -0.0679
## magnet_forearm_x      0.1052 -0.0107  0.0089 -0.0080  0.0969  0.0128
## magnet_forearm_y      0.0246  0.0536 -0.1390  0.0458 -0.0113 -0.2236
## magnet_forearm_z     -0.0385  0.1116 -0.1982  0.0546 -0.3294  0.0027
##                          PC7     PC8     PC9    PC10    PC11    PC12
## roll_belt             0.0379 -0.0861  0.0166 -0.0061  0.0010 -0.0298
## pitch_belt           -0.1221 -0.0315 -0.0179 -0.0473  0.0258 -0.1585
## yaw_belt              0.0949 -0.0441  0.0203  0.0207 -0.0165  0.1267
## total_accel_belt      0.0384 -0.0957  0.0222  0.0018  0.0054 -0.0603
## gyros_belt_x         -0.0757  0.0477 -0.0568 -0.1250 -0.0655 -0.0688
## gyros_belt_y         -0.0752 -0.0346 -0.0744 -0.0601 -0.1888  0.0311
## gyros_belt_z         -0.1492  0.0213 -0.1470  0.0412 -0.1792  0.0294
## accel_belt_x          0.1001  0.0134  0.0006  0.0396 -0.0276  0.0977
## accel_belt_y          0.0057 -0.0871  0.0259 -0.0162  0.0126 -0.0361
## accel_belt_z         -0.0318  0.0911 -0.0077  0.0132  0.0009  0.0585
## magnet_belt_x         0.1062 -0.0253  0.0127  0.0710  0.0359  0.0529
## magnet_belt_y        -0.0055  0.1486  0.0929 -0.0248 -0.0572  0.3750
## magnet_belt_z        -0.0394  0.1989  0.0836 -0.1223 -0.0582  0.4520
## roll_arm              0.0178  0.0988 -0.0242  0.1591  0.0134  0.3095
## pitch_arm             0.0542  0.0291  0.0477  0.0005  0.2758  0.0827
## yaw_arm               0.0112  0.0454 -0.0559  0.2313 -0.0837  0.0975
## total_accel_arm      -0.0587 -0.3328  0.1624 -0.2506  0.4244  0.1085
## gyros_arm_x          -0.5177  0.0801  0.3004  0.3058  0.0662 -0.0107
## gyros_arm_y           0.4919 -0.0675 -0.3060 -0.2969 -0.0376  0.0065
## gyros_arm_z          -0.2242 -0.0327  0.1342  0.0199  0.0339 -0.0165
## accel_arm_x           0.0554  0.0179 -0.0305  0.1788 -0.0385  0.0011
## accel_arm_y           0.0300  0.1329 -0.0033  0.1335 -0.0180 -0.0648
## accel_arm_z           0.1134  0.2622 -0.0709  0.1742 -0.2115 -0.0429
## magnet_arm_x          0.0632  0.1404 -0.1083  0.2140 -0.2058 -0.0548
## magnet_arm_y          0.0010  0.0356  0.1069  0.0143  0.0582 -0.0212
## magnet_arm_z          0.0150  0.1972 -0.0295  0.0560 -0.1317 -0.0353
## roll_dumbbell         0.1565  0.2938  0.3452 -0.1677  0.0092 -0.2496
## pitch_dumbbell        0.0745  0.2693  0.3521 -0.2789 -0.0793 -0.0340
## yaw_dumbbell         -0.0651  0.0618  0.0852 -0.1579 -0.0535  0.0403
## total_accel_dumbbell  0.1408 -0.0183  0.0997  0.1664  0.1643 -0.2193
## gyros_dumbbell_x      0.0071 -0.0317 -0.0051 -0.0064  0.0451 -0.0440
## gyros_dumbbell_y      0.0001 -0.0518 -0.0064 -0.0326  0.0051 -0.1756
## gyros_dumbbell_z      0.0148  0.0300  0.0086  0.0261 -0.0451  0.0670
## accel_dumbbell_x      0.0089  0.1796  0.2180 -0.2378 -0.1302  0.1157
## accel_dumbbell_y      0.1540  0.1747  0.2279 -0.0120  0.0570 -0.2472
## accel_dumbbell_z     -0.0719  0.0527  0.0541 -0.1377 -0.0465  0.0690
## magnet_dumbbell_x     0.1173  0.0361  0.1939  0.0114  0.0981 -0.0779
## magnet_dumbbell_y    -0.0624  0.1220  0.0368 -0.1574 -0.0919 -0.1728
## magnet_dumbbell_z     0.0531  0.0177  0.0342  0.0709  0.0427 -0.0703
## roll_forearm         -0.0522  0.0015 -0.0191  0.0043 -0.1992 -0.1365
## pitch_forearm         0.0785  0.2685 -0.1319  0.1555  0.1108 -0.1176
## yaw_forearm          -0.1387  0.0704 -0.1305 -0.0678  0.0632  0.0622
## total_accel_forearm  -0.0044  0.2485 -0.0922 -0.1302  0.2548  0.2117
## gyros_forearm_x       0.0732  0.0317  0.0128 -0.0139 -0.0693  0.1560
## gyros_forearm_y       0.0320  0.0014  0.0147  0.0155  0.0075  0.0430
## gyros_forearm_z       0.0308  0.0212  0.0116  0.0304 -0.0173  0.0688
## accel_forearm_x       0.0139 -0.2810  0.1911 -0.0441 -0.3239  0.0175
## accel_forearm_y      -0.1878  0.0491 -0.1565 -0.2400 -0.1699  0.0042
## accel_forearm_z      -0.0732 -0.0541 -0.0541 -0.0246 -0.0191 -0.1323
## magnet_forearm_x      0.1523 -0.3757  0.3498  0.0745 -0.3797  0.1205
## magnet_forearm_y     -0.3508  0.0163 -0.2327 -0.3374 -0.0956 -0.1672
## magnet_forearm_z     -0.0333  0.0241  0.0111 -0.0912 -0.2052 -0.0877
```

52 variables can be reduced to 12 components to capture 80% of the variance.  


```r
trainPC <- predict(preproc, train[,-1]); trainPC$classe <- train$classe
```

## Modeling  

Since the dependent variable is discrete data we cannot take regression method. In particular here we use GBM method to classify the training data set into 5 exercise classes which should match the original class.  


```r
library(gbm)
modFit <- train(classe ~ ., method = "gbm", data = trainPC)
```

```
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0570
##      2        1.5757             nan     0.1000    0.0424
##      3        1.5500             nan     0.1000    0.0365
##      4        1.5277             nan     0.1000    0.0289
##      5        1.5095             nan     0.1000    0.0244
##      6        1.4944             nan     0.1000    0.0206
##      7        1.4813             nan     0.1000    0.0199
##      8        1.4695             nan     0.1000    0.0190
##      9        1.4576             nan     0.1000    0.0155
##     10        1.4484             nan     0.1000    0.0127
##     20        1.3794             nan     0.1000    0.0079
##     40        1.2938             nan     0.1000    0.0047
##     60        1.2373             nan     0.1000    0.0018
##     80        1.1965             nan     0.1000    0.0023
##    100        1.1634             nan     0.1000    0.0010
##    120        1.1369             nan     0.1000    0.0016
##    140        1.1138             nan     0.1000    0.0010
##    150        1.1030             nan     0.1000    0.0011
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1004
##      2        1.5509             nan     0.1000    0.0713
##      3        1.5084             nan     0.1000    0.0527
##      4        1.4770             nan     0.1000    0.0449
##      5        1.4497             nan     0.1000    0.0322
##      6        1.4301             nan     0.1000    0.0279
##      7        1.4132             nan     0.1000    0.0305
##      8        1.3946             nan     0.1000    0.0257
##      9        1.3773             nan     0.1000    0.0234
##     10        1.3626             nan     0.1000    0.0224
##     20        1.2552             nan     0.1000    0.0123
##     40        1.1357             nan     0.1000    0.0063
##     60        1.0546             nan     0.1000    0.0036
##     80        0.9958             nan     0.1000    0.0029
##    100        0.9488             nan     0.1000    0.0018
##    120        0.9030             nan     0.1000    0.0013
##    140        0.8647             nan     0.1000    0.0027
##    150        0.8464             nan     0.1000    0.0017
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1203
##      2        1.5374             nan     0.1000    0.0902
##      3        1.4837             nan     0.1000    0.0646
##      4        1.4437             nan     0.1000    0.0539
##      5        1.4114             nan     0.1000    0.0448
##      6        1.3841             nan     0.1000    0.0406
##      7        1.3595             nan     0.1000    0.0374
##      8        1.3369             nan     0.1000    0.0286
##      9        1.3195             nan     0.1000    0.0304
##     10        1.2992             nan     0.1000    0.0307
##     20        1.1705             nan     0.1000    0.0147
##     40        1.0298             nan     0.1000    0.0057
##     60        0.9391             nan     0.1000    0.0050
##     80        0.8659             nan     0.1000    0.0048
##    100        0.8064             nan     0.1000    0.0027
##    120        0.7566             nan     0.1000    0.0038
##    140        0.7129             nan     0.1000    0.0021
##    150        0.6929             nan     0.1000    0.0017
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0557
##      2        1.5764             nan     0.1000    0.0395
##      3        1.5511             nan     0.1000    0.0388
##      4        1.5282             nan     0.1000    0.0315
##      5        1.5087             nan     0.1000    0.0265
##      6        1.4929             nan     0.1000    0.0231
##      7        1.4791             nan     0.1000    0.0198
##      8        1.4663             nan     0.1000    0.0176
##      9        1.4556             nan     0.1000    0.0172
##     10        1.4449             nan     0.1000    0.0149
##     20        1.3718             nan     0.1000    0.0075
##     40        1.2847             nan     0.1000    0.0044
##     60        1.2276             nan     0.1000    0.0029
##     80        1.1873             nan     0.1000    0.0016
##    100        1.1545             nan     0.1000    0.0016
##    120        1.1276             nan     0.1000    0.0012
##    140        1.1039             nan     0.1000    0.0010
##    150        1.0934             nan     0.1000    0.0006
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0880
##      2        1.5564             nan     0.1000    0.0759
##      3        1.5108             nan     0.1000    0.0544
##      4        1.4777             nan     0.1000    0.0453
##      5        1.4513             nan     0.1000    0.0396
##      6        1.4279             nan     0.1000    0.0332
##      7        1.4078             nan     0.1000    0.0266
##      8        1.3913             nan     0.1000    0.0302
##      9        1.3714             nan     0.1000    0.0223
##     10        1.3574             nan     0.1000    0.0227
##     20        1.2486             nan     0.1000    0.0135
##     40        1.1302             nan     0.1000    0.0064
##     60        1.0518             nan     0.1000    0.0045
##     80        0.9880             nan     0.1000    0.0039
##    100        0.9347             nan     0.1000    0.0031
##    120        0.8905             nan     0.1000    0.0026
##    140        0.8526             nan     0.1000    0.0010
##    150        0.8365             nan     0.1000    0.0015
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1129
##      2        1.5422             nan     0.1000    0.0910
##      3        1.4885             nan     0.1000    0.0719
##      4        1.4463             nan     0.1000    0.0473
##      5        1.4169             nan     0.1000    0.0493
##      6        1.3870             nan     0.1000    0.0409
##      7        1.3612             nan     0.1000    0.0361
##      8        1.3386             nan     0.1000    0.0333
##      9        1.3176             nan     0.1000    0.0298
##     10        1.2986             nan     0.1000    0.0276
##     20        1.1702             nan     0.1000    0.0138
##     40        1.0289             nan     0.1000    0.0092
##     60        0.9344             nan     0.1000    0.0034
##     80        0.8627             nan     0.1000    0.0026
##    100        0.8011             nan     0.1000    0.0048
##    120        0.7496             nan     0.1000    0.0022
##    140        0.7051             nan     0.1000    0.0021
##    150        0.6829             nan     0.1000    0.0025
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0596
##      2        1.5730             nan     0.1000    0.0461
##      3        1.5449             nan     0.1000    0.0366
##      4        1.5233             nan     0.1000    0.0337
##      5        1.5034             nan     0.1000    0.0258
##      6        1.4870             nan     0.1000    0.0224
##      7        1.4739             nan     0.1000    0.0198
##      8        1.4613             nan     0.1000    0.0201
##      9        1.4495             nan     0.1000    0.0170
##     10        1.4391             nan     0.1000    0.0146
##     20        1.3670             nan     0.1000    0.0099
##     40        1.2767             nan     0.1000    0.0052
##     60        1.2189             nan     0.1000    0.0026
##     80        1.1765             nan     0.1000    0.0016
##    100        1.1436             nan     0.1000    0.0019
##    120        1.1165             nan     0.1000    0.0012
##    140        1.0930             nan     0.1000    0.0010
##    150        1.0822             nan     0.1000    0.0010
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0909
##      2        1.5542             nan     0.1000    0.0638
##      3        1.5154             nan     0.1000    0.0609
##      4        1.4795             nan     0.1000    0.0519
##      5        1.4486             nan     0.1000    0.0416
##      6        1.4236             nan     0.1000    0.0364
##      7        1.4016             nan     0.1000    0.0304
##      8        1.3826             nan     0.1000    0.0257
##      9        1.3667             nan     0.1000    0.0247
##     10        1.3511             nan     0.1000    0.0239
##     20        1.2411             nan     0.1000    0.0118
##     40        1.1196             nan     0.1000    0.0059
##     60        1.0382             nan     0.1000    0.0046
##     80        0.9762             nan     0.1000    0.0021
##    100        0.9250             nan     0.1000    0.0019
##    120        0.8812             nan     0.1000    0.0023
##    140        0.8442             nan     0.1000    0.0029
##    150        0.8270             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1248
##      2        1.5356             nan     0.1000    0.0962
##      3        1.4798             nan     0.1000    0.0652
##      4        1.4413             nan     0.1000    0.0525
##      5        1.4100             nan     0.1000    0.0474
##      6        1.3812             nan     0.1000    0.0457
##      7        1.3529             nan     0.1000    0.0390
##      8        1.3285             nan     0.1000    0.0346
##      9        1.3065             nan     0.1000    0.0251
##     10        1.2899             nan     0.1000    0.0304
##     20        1.1580             nan     0.1000    0.0139
##     40        1.0137             nan     0.1000    0.0090
##     60        0.9203             nan     0.1000    0.0060
##     80        0.8508             nan     0.1000    0.0045
##    100        0.7923             nan     0.1000    0.0022
##    120        0.7390             nan     0.1000    0.0030
##    140        0.6951             nan     0.1000    0.0019
##    150        0.6754             nan     0.1000    0.0026
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0588
##      2        1.5737             nan     0.1000    0.0461
##      3        1.5460             nan     0.1000    0.0359
##      4        1.5243             nan     0.1000    0.0320
##      5        1.5041             nan     0.1000    0.0249
##      6        1.4892             nan     0.1000    0.0218
##      7        1.4758             nan     0.1000    0.0193
##      8        1.4643             nan     0.1000    0.0195
##      9        1.4521             nan     0.1000    0.0152
##     10        1.4423             nan     0.1000    0.0142
##     20        1.3697             nan     0.1000    0.0087
##     40        1.2813             nan     0.1000    0.0052
##     60        1.2261             nan     0.1000    0.0029
##     80        1.1829             nan     0.1000    0.0023
##    100        1.1486             nan     0.1000    0.0013
##    120        1.1218             nan     0.1000    0.0012
##    140        1.0988             nan     0.1000    0.0014
##    150        1.0887             nan     0.1000    0.0005
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1003
##      2        1.5487             nan     0.1000    0.0659
##      3        1.5090             nan     0.1000    0.0491
##      4        1.4784             nan     0.1000    0.0476
##      5        1.4496             nan     0.1000    0.0383
##      6        1.4264             nan     0.1000    0.0348
##      7        1.4055             nan     0.1000    0.0281
##      8        1.3882             nan     0.1000    0.0260
##      9        1.3721             nan     0.1000    0.0273
##     10        1.3548             nan     0.1000    0.0203
##     20        1.2429             nan     0.1000    0.0116
##     40        1.1218             nan     0.1000    0.0062
##     60        1.0419             nan     0.1000    0.0033
##     80        0.9791             nan     0.1000    0.0021
##    100        0.9302             nan     0.1000    0.0014
##    120        0.8890             nan     0.1000    0.0029
##    140        0.8505             nan     0.1000    0.0018
##    150        0.8340             nan     0.1000    0.0022
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1149
##      2        1.5387             nan     0.1000    0.0789
##      3        1.4909             nan     0.1000    0.0707
##      4        1.4489             nan     0.1000    0.0491
##      5        1.4185             nan     0.1000    0.0435
##      6        1.3915             nan     0.1000    0.0396
##      7        1.3665             nan     0.1000    0.0383
##      8        1.3438             nan     0.1000    0.0388
##      9        1.3207             nan     0.1000    0.0387
##     10        1.2960             nan     0.1000    0.0286
##     20        1.1616             nan     0.1000    0.0170
##     40        1.0199             nan     0.1000    0.0062
##     60        0.9281             nan     0.1000    0.0057
##     80        0.8535             nan     0.1000    0.0032
##    100        0.7964             nan     0.1000    0.0035
##    120        0.7521             nan     0.1000    0.0022
##    140        0.7096             nan     0.1000    0.0017
##    150        0.6901             nan     0.1000    0.0016
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0588
##      2        1.5716             nan     0.1000    0.0472
##      3        1.5431             nan     0.1000    0.0333
##      4        1.5221             nan     0.1000    0.0327
##      5        1.5021             nan     0.1000    0.0282
##      6        1.4849             nan     0.1000    0.0238
##      7        1.4705             nan     0.1000    0.0191
##      8        1.4587             nan     0.1000    0.0186
##      9        1.4477             nan     0.1000    0.0163
##     10        1.4377             nan     0.1000    0.0151
##     20        1.3654             nan     0.1000    0.0088
##     40        1.2747             nan     0.1000    0.0046
##     60        1.2185             nan     0.1000    0.0026
##     80        1.1744             nan     0.1000    0.0022
##    100        1.1426             nan     0.1000    0.0016
##    120        1.1158             nan     0.1000    0.0014
##    140        1.0920             nan     0.1000    0.0010
##    150        1.0813             nan     0.1000    0.0008
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0912
##      2        1.5539             nan     0.1000    0.0637
##      3        1.5148             nan     0.1000    0.0624
##      4        1.4779             nan     0.1000    0.0466
##      5        1.4492             nan     0.1000    0.0397
##      6        1.4261             nan     0.1000    0.0330
##      7        1.4056             nan     0.1000    0.0296
##      8        1.3868             nan     0.1000    0.0251
##      9        1.3704             nan     0.1000    0.0278
##     10        1.3524             nan     0.1000    0.0213
##     20        1.2447             nan     0.1000    0.0112
##     40        1.1244             nan     0.1000    0.0041
##     60        1.0446             nan     0.1000    0.0032
##     80        0.9815             nan     0.1000    0.0040
##    100        0.9326             nan     0.1000    0.0027
##    120        0.8880             nan     0.1000    0.0035
##    140        0.8521             nan     0.1000    0.0024
##    150        0.8331             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1171
##      2        1.5398             nan     0.1000    0.0817
##      3        1.4903             nan     0.1000    0.0633
##      4        1.4515             nan     0.1000    0.0576
##      5        1.4160             nan     0.1000    0.0489
##      6        1.3860             nan     0.1000    0.0439
##      7        1.3593             nan     0.1000    0.0397
##      8        1.3353             nan     0.1000    0.0371
##      9        1.3131             nan     0.1000    0.0310
##     10        1.2933             nan     0.1000    0.0293
##     20        1.1635             nan     0.1000    0.0138
##     40        1.0212             nan     0.1000    0.0076
##     60        0.9243             nan     0.1000    0.0058
##     80        0.8536             nan     0.1000    0.0028
##    100        0.7943             nan     0.1000    0.0051
##    120        0.7430             nan     0.1000    0.0039
##    140        0.6999             nan     0.1000    0.0030
##    150        0.6798             nan     0.1000    0.0033
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0566
##      2        1.5736             nan     0.1000    0.0446
##      3        1.5471             nan     0.1000    0.0349
##      4        1.5259             nan     0.1000    0.0296
##      5        1.5079             nan     0.1000    0.0271
##      6        1.4916             nan     0.1000    0.0228
##      7        1.4781             nan     0.1000    0.0175
##      8        1.4672             nan     0.1000    0.0194
##      9        1.4554             nan     0.1000    0.0153
##     10        1.4460             nan     0.1000    0.0145
##     20        1.3743             nan     0.1000    0.0081
##     40        1.2862             nan     0.1000    0.0045
##     60        1.2279             nan     0.1000    0.0028
##     80        1.1871             nan     0.1000    0.0019
##    100        1.1550             nan     0.1000    0.0019
##    120        1.1266             nan     0.1000    0.0011
##    140        1.1036             nan     0.1000    0.0010
##    150        1.0933             nan     0.1000    0.0009
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0884
##      2        1.5568             nan     0.1000    0.0621
##      3        1.5196             nan     0.1000    0.0609
##      4        1.4826             nan     0.1000    0.0468
##      5        1.4545             nan     0.1000    0.0386
##      6        1.4316             nan     0.1000    0.0335
##      7        1.4109             nan     0.1000    0.0289
##      8        1.3933             nan     0.1000    0.0241
##      9        1.3781             nan     0.1000    0.0256
##     10        1.3612             nan     0.1000    0.0235
##     20        1.2536             nan     0.1000    0.0118
##     40        1.1312             nan     0.1000    0.0072
##     60        1.0507             nan     0.1000    0.0040
##     80        0.9918             nan     0.1000    0.0031
##    100        0.9414             nan     0.1000    0.0038
##    120        0.8990             nan     0.1000    0.0017
##    140        0.8627             nan     0.1000    0.0020
##    150        0.8435             nan     0.1000    0.0020
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1225
##      2        1.5363             nan     0.1000    0.0805
##      3        1.4883             nan     0.1000    0.0607
##      4        1.4508             nan     0.1000    0.0525
##      5        1.4195             nan     0.1000    0.0446
##      6        1.3918             nan     0.1000    0.0412
##      7        1.3661             nan     0.1000    0.0385
##      8        1.3424             nan     0.1000    0.0357
##      9        1.3208             nan     0.1000    0.0280
##     10        1.3026             nan     0.1000    0.0291
##     20        1.1674             nan     0.1000    0.0150
##     40        1.0264             nan     0.1000    0.0069
##     60        0.9383             nan     0.1000    0.0057
##     80        0.8642             nan     0.1000    0.0049
##    100        0.8033             nan     0.1000    0.0048
##    120        0.7510             nan     0.1000    0.0023
##    140        0.7060             nan     0.1000    0.0030
##    150        0.6856             nan     0.1000    0.0017
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0618
##      2        1.5714             nan     0.1000    0.0437
##      3        1.5436             nan     0.1000    0.0392
##      4        1.5200             nan     0.1000    0.0327
##      5        1.5002             nan     0.1000    0.0262
##      6        1.4842             nan     0.1000    0.0241
##      7        1.4689             nan     0.1000    0.0208
##      8        1.4561             nan     0.1000    0.0195
##      9        1.4443             nan     0.1000    0.0153
##     10        1.4349             nan     0.1000    0.0155
##     20        1.3630             nan     0.1000    0.0073
##     40        1.2768             nan     0.1000    0.0052
##     60        1.2190             nan     0.1000    0.0042
##     80        1.1776             nan     0.1000    0.0021
##    100        1.1456             nan     0.1000    0.0015
##    120        1.1177             nan     0.1000    0.0014
##    140        1.0950             nan     0.1000    0.0009
##    150        1.0849             nan     0.1000    0.0010
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0953
##      2        1.5519             nan     0.1000    0.0651
##      3        1.5126             nan     0.1000    0.0617
##      4        1.4762             nan     0.1000    0.0405
##      5        1.4505             nan     0.1000    0.0425
##      6        1.4252             nan     0.1000    0.0389
##      7        1.4015             nan     0.1000    0.0314
##      8        1.3826             nan     0.1000    0.0296
##      9        1.3634             nan     0.1000    0.0244
##     10        1.3478             nan     0.1000    0.0239
##     20        1.2396             nan     0.1000    0.0112
##     40        1.1224             nan     0.1000    0.0057
##     60        1.0434             nan     0.1000    0.0048
##     80        0.9824             nan     0.1000    0.0030
##    100        0.9314             nan     0.1000    0.0028
##    120        0.8875             nan     0.1000    0.0018
##    140        0.8497             nan     0.1000    0.0022
##    150        0.8313             nan     0.1000    0.0017
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1198
##      2        1.5375             nan     0.1000    0.0842
##      3        1.4881             nan     0.1000    0.0654
##      4        1.4486             nan     0.1000    0.0527
##      5        1.4162             nan     0.1000    0.0420
##      6        1.3901             nan     0.1000    0.0419
##      7        1.3639             nan     0.1000    0.0394
##      8        1.3399             nan     0.1000    0.0387
##      9        1.3153             nan     0.1000    0.0314
##     10        1.2952             nan     0.1000    0.0274
##     20        1.1611             nan     0.1000    0.0137
##     40        1.0204             nan     0.1000    0.0087
##     60        0.9277             nan     0.1000    0.0059
##     80        0.8537             nan     0.1000    0.0033
##    100        0.7907             nan     0.1000    0.0034
##    120        0.7430             nan     0.1000    0.0027
##    140        0.6972             nan     0.1000    0.0029
##    150        0.6777             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0582
##      2        1.5746             nan     0.1000    0.0441
##      3        1.5477             nan     0.1000    0.0368
##      4        1.5252             nan     0.1000    0.0324
##      5        1.5059             nan     0.1000    0.0270
##      6        1.4897             nan     0.1000    0.0237
##      7        1.4754             nan     0.1000    0.0208
##      8        1.4627             nan     0.1000    0.0163
##      9        1.4523             nan     0.1000    0.0154
##     10        1.4425             nan     0.1000    0.0149
##     20        1.3709             nan     0.1000    0.0086
##     40        1.2815             nan     0.1000    0.0042
##     60        1.2249             nan     0.1000    0.0027
##     80        1.1824             nan     0.1000    0.0022
##    100        1.1494             nan     0.1000    0.0014
##    120        1.1225             nan     0.1000    0.0015
##    140        1.0987             nan     0.1000    0.0010
##    150        1.0888             nan     0.1000    0.0004
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0898
##      2        1.5547             nan     0.1000    0.0746
##      3        1.5104             nan     0.1000    0.0487
##      4        1.4811             nan     0.1000    0.0475
##      5        1.4530             nan     0.1000    0.0380
##      6        1.4291             nan     0.1000    0.0285
##      7        1.4118             nan     0.1000    0.0318
##      8        1.3924             nan     0.1000    0.0259
##      9        1.3765             nan     0.1000    0.0273
##     10        1.3591             nan     0.1000    0.0226
##     20        1.2489             nan     0.1000    0.0113
##     40        1.1237             nan     0.1000    0.0056
##     60        1.0448             nan     0.1000    0.0027
##     80        0.9853             nan     0.1000    0.0032
##    100        0.9336             nan     0.1000    0.0025
##    120        0.8912             nan     0.1000    0.0028
##    140        0.8542             nan     0.1000    0.0012
##    150        0.8377             nan     0.1000    0.0024
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1129
##      2        1.5411             nan     0.1000    0.0937
##      3        1.4865             nan     0.1000    0.0611
##      4        1.4496             nan     0.1000    0.0592
##      5        1.4140             nan     0.1000    0.0450
##      6        1.3861             nan     0.1000    0.0437
##      7        1.3587             nan     0.1000    0.0374
##      8        1.3352             nan     0.1000    0.0323
##      9        1.3141             nan     0.1000    0.0292
##     10        1.2952             nan     0.1000    0.0263
##     20        1.1652             nan     0.1000    0.0130
##     40        1.0235             nan     0.1000    0.0070
##     60        0.9300             nan     0.1000    0.0051
##     80        0.8569             nan     0.1000    0.0026
##    100        0.7961             nan     0.1000    0.0036
##    120        0.7473             nan     0.1000    0.0045
##    140        0.7023             nan     0.1000    0.0015
##    150        0.6812             nan     0.1000    0.0014
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0599
##      2        1.5728             nan     0.1000    0.0443
##      3        1.5457             nan     0.1000    0.0381
##      4        1.5225             nan     0.1000    0.0297
##      5        1.5045             nan     0.1000    0.0299
##      6        1.4867             nan     0.1000    0.0244
##      7        1.4723             nan     0.1000    0.0179
##      8        1.4612             nan     0.1000    0.0173
##      9        1.4503             nan     0.1000    0.0172
##     10        1.4398             nan     0.1000    0.0149
##     20        1.3684             nan     0.1000    0.0087
##     40        1.2792             nan     0.1000    0.0057
##     60        1.2218             nan     0.1000    0.0033
##     80        1.1804             nan     0.1000    0.0020
##    100        1.1489             nan     0.1000    0.0013
##    120        1.1217             nan     0.1000    0.0011
##    140        1.0986             nan     0.1000    0.0008
##    150        1.0888             nan     0.1000    0.0007
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0886
##      2        1.5539             nan     0.1000    0.0651
##      3        1.5149             nan     0.1000    0.0591
##      4        1.4794             nan     0.1000    0.0500
##      5        1.4489             nan     0.1000    0.0407
##      6        1.4238             nan     0.1000    0.0339
##      7        1.4032             nan     0.1000    0.0312
##      8        1.3840             nan     0.1000    0.0245
##      9        1.3683             nan     0.1000    0.0240
##     10        1.3534             nan     0.1000    0.0238
##     20        1.2453             nan     0.1000    0.0115
##     40        1.1219             nan     0.1000    0.0062
##     60        1.0459             nan     0.1000    0.0051
##     80        0.9861             nan     0.1000    0.0034
##    100        0.9334             nan     0.1000    0.0034
##    120        0.8902             nan     0.1000    0.0012
##    140        0.8531             nan     0.1000    0.0023
##    150        0.8342             nan     0.1000    0.0011
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1145
##      2        1.5408             nan     0.1000    0.0816
##      3        1.4913             nan     0.1000    0.0635
##      4        1.4526             nan     0.1000    0.0486
##      5        1.4225             nan     0.1000    0.0520
##      6        1.3894             nan     0.1000    0.0445
##      7        1.3623             nan     0.1000    0.0356
##      8        1.3404             nan     0.1000    0.0399
##      9        1.3162             nan     0.1000    0.0329
##     10        1.2951             nan     0.1000    0.0257
##     20        1.1681             nan     0.1000    0.0147
##     40        1.0276             nan     0.1000    0.0088
##     60        0.9360             nan     0.1000    0.0067
##     80        0.8631             nan     0.1000    0.0053
##    100        0.8017             nan     0.1000    0.0024
##    120        0.7488             nan     0.1000    0.0020
##    140        0.7081             nan     0.1000    0.0025
##    150        0.6893             nan     0.1000    0.0029
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0585
##      2        1.5734             nan     0.1000    0.0442
##      3        1.5467             nan     0.1000    0.0354
##      4        1.5246             nan     0.1000    0.0302
##      5        1.5060             nan     0.1000    0.0265
##      6        1.4901             nan     0.1000    0.0230
##      7        1.4765             nan     0.1000    0.0205
##      8        1.4639             nan     0.1000    0.0184
##      9        1.4522             nan     0.1000    0.0145
##     10        1.4434             nan     0.1000    0.0139
##     20        1.3715             nan     0.1000    0.0075
##     40        1.2832             nan     0.1000    0.0060
##     60        1.2260             nan     0.1000    0.0037
##     80        1.1857             nan     0.1000    0.0020
##    100        1.1525             nan     0.1000    0.0017
##    120        1.1258             nan     0.1000    0.0017
##    140        1.1036             nan     0.1000    0.0009
##    150        1.0940             nan     0.1000    0.0007
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0871
##      2        1.5567             nan     0.1000    0.0731
##      3        1.5123             nan     0.1000    0.0480
##      4        1.4826             nan     0.1000    0.0504
##      5        1.4525             nan     0.1000    0.0397
##      6        1.4282             nan     0.1000    0.0330
##      7        1.4082             nan     0.1000    0.0273
##      8        1.3916             nan     0.1000    0.0271
##      9        1.3747             nan     0.1000    0.0275
##     10        1.3578             nan     0.1000    0.0224
##     20        1.2492             nan     0.1000    0.0124
##     40        1.1264             nan     0.1000    0.0057
##     60        1.0485             nan     0.1000    0.0062
##     80        0.9859             nan     0.1000    0.0035
##    100        0.9359             nan     0.1000    0.0022
##    120        0.8944             nan     0.1000    0.0040
##    140        0.8562             nan     0.1000    0.0023
##    150        0.8393             nan     0.1000    0.0021
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1122
##      2        1.5410             nan     0.1000    0.0797
##      3        1.4928             nan     0.1000    0.0620
##      4        1.4555             nan     0.1000    0.0592
##      5        1.4196             nan     0.1000    0.0480
##      6        1.3908             nan     0.1000    0.0464
##      7        1.3632             nan     0.1000    0.0386
##      8        1.3395             nan     0.1000    0.0343
##      9        1.3184             nan     0.1000    0.0291
##     10        1.2999             nan     0.1000    0.0304
##     20        1.1684             nan     0.1000    0.0158
##     40        1.0246             nan     0.1000    0.0076
##     60        0.9348             nan     0.1000    0.0045
##     80        0.8618             nan     0.1000    0.0041
##    100        0.8041             nan     0.1000    0.0034
##    120        0.7546             nan     0.1000    0.0032
##    140        0.7127             nan     0.1000    0.0022
##    150        0.6927             nan     0.1000    0.0021
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0604
##      2        1.5723             nan     0.1000    0.0477
##      3        1.5441             nan     0.1000    0.0369
##      4        1.5214             nan     0.1000    0.0326
##      5        1.5011             nan     0.1000    0.0270
##      6        1.4844             nan     0.1000    0.0209
##      7        1.4712             nan     0.1000    0.0222
##      8        1.4574             nan     0.1000    0.0187
##      9        1.4462             nan     0.1000    0.0157
##     10        1.4361             nan     0.1000    0.0133
##     20        1.3650             nan     0.1000    0.0100
##     40        1.2770             nan     0.1000    0.0037
##     60        1.2202             nan     0.1000    0.0022
##     80        1.1778             nan     0.1000    0.0016
##    100        1.1463             nan     0.1000    0.0019
##    120        1.1197             nan     0.1000    0.0011
##    140        1.0970             nan     0.1000    0.0011
##    150        1.0868             nan     0.1000    0.0007
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0924
##      2        1.5530             nan     0.1000    0.0637
##      3        1.5143             nan     0.1000    0.0643
##      4        1.4764             nan     0.1000    0.0478
##      5        1.4469             nan     0.1000    0.0396
##      6        1.4235             nan     0.1000    0.0348
##      7        1.4028             nan     0.1000    0.0282
##      8        1.3853             nan     0.1000    0.0246
##      9        1.3702             nan     0.1000    0.0274
##     10        1.3530             nan     0.1000    0.0248
##     20        1.2420             nan     0.1000    0.0105
##     40        1.1160             nan     0.1000    0.0070
##     60        1.0392             nan     0.1000    0.0054
##     80        0.9796             nan     0.1000    0.0037
##    100        0.9305             nan     0.1000    0.0022
##    120        0.8860             nan     0.1000    0.0028
##    140        0.8504             nan     0.1000    0.0014
##    150        0.8341             nan     0.1000    0.0011
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1275
##      2        1.5327             nan     0.1000    0.0840
##      3        1.4832             nan     0.1000    0.0710
##      4        1.4398             nan     0.1000    0.0492
##      5        1.4097             nan     0.1000    0.0467
##      6        1.3818             nan     0.1000    0.0434
##      7        1.3554             nan     0.1000    0.0365
##      8        1.3329             nan     0.1000    0.0331
##      9        1.3119             nan     0.1000    0.0308
##     10        1.2919             nan     0.1000    0.0273
##     20        1.1577             nan     0.1000    0.0156
##     40        1.0167             nan     0.1000    0.0068
##     60        0.9245             nan     0.1000    0.0048
##     80        0.8567             nan     0.1000    0.0047
##    100        0.7951             nan     0.1000    0.0024
##    120        0.7462             nan     0.1000    0.0032
##    140        0.7045             nan     0.1000    0.0021
##    150        0.6844             nan     0.1000    0.0014
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0575
##      2        1.5725             nan     0.1000    0.0461
##      3        1.5443             nan     0.1000    0.0362
##      4        1.5220             nan     0.1000    0.0317
##      5        1.5024             nan     0.1000    0.0247
##      6        1.4874             nan     0.1000    0.0227
##      7        1.4735             nan     0.1000    0.0188
##      8        1.4621             nan     0.1000    0.0204
##      9        1.4496             nan     0.1000    0.0155
##     10        1.4403             nan     0.1000    0.0173
##     20        1.3696             nan     0.1000    0.0086
##     40        1.2805             nan     0.1000    0.0052
##     60        1.2233             nan     0.1000    0.0028
##     80        1.1814             nan     0.1000    0.0019
##    100        1.1481             nan     0.1000    0.0016
##    120        1.1224             nan     0.1000    0.0007
##    140        1.0991             nan     0.1000    0.0007
##    150        1.0887             nan     0.1000    0.0007
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0887
##      2        1.5545             nan     0.1000    0.0743
##      3        1.5095             nan     0.1000    0.0566
##      4        1.4750             nan     0.1000    0.0427
##      5        1.4492             nan     0.1000    0.0388
##      6        1.4257             nan     0.1000    0.0336
##      7        1.4052             nan     0.1000    0.0266
##      8        1.3885             nan     0.1000    0.0287
##      9        1.3701             nan     0.1000    0.0204
##     10        1.3565             nan     0.1000    0.0243
##     20        1.2432             nan     0.1000    0.0131
##     40        1.1224             nan     0.1000    0.0074
##     60        1.0440             nan     0.1000    0.0042
##     80        0.9856             nan     0.1000    0.0046
##    100        0.9342             nan     0.1000    0.0026
##    120        0.8910             nan     0.1000    0.0022
##    140        0.8543             nan     0.1000    0.0019
##    150        0.8373             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1135
##      2        1.5413             nan     0.1000    0.0805
##      3        1.4936             nan     0.1000    0.0637
##      4        1.4550             nan     0.1000    0.0598
##      5        1.4192             nan     0.1000    0.0458
##      6        1.3909             nan     0.1000    0.0448
##      7        1.3639             nan     0.1000    0.0343
##      8        1.3429             nan     0.1000    0.0339
##      9        1.3220             nan     0.1000    0.0346
##     10        1.3000             nan     0.1000    0.0296
##     20        1.1632             nan     0.1000    0.0124
##     40        1.0211             nan     0.1000    0.0053
##     60        0.9285             nan     0.1000    0.0044
##     80        0.8539             nan     0.1000    0.0029
##    100        0.7971             nan     0.1000    0.0037
##    120        0.7485             nan     0.1000    0.0042
##    140        0.7035             nan     0.1000    0.0023
##    150        0.6833             nan     0.1000    0.0025
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0570
##      2        1.5733             nan     0.1000    0.0465
##      3        1.5461             nan     0.1000    0.0329
##      4        1.5259             nan     0.1000    0.0308
##      5        1.5071             nan     0.1000    0.0285
##      6        1.4903             nan     0.1000    0.0213
##      7        1.4770             nan     0.1000    0.0208
##      8        1.4641             nan     0.1000    0.0176
##      9        1.4534             nan     0.1000    0.0159
##     10        1.4437             nan     0.1000    0.0139
##     20        1.3705             nan     0.1000    0.0084
##     40        1.2790             nan     0.1000    0.0045
##     60        1.2215             nan     0.1000    0.0035
##     80        1.1811             nan     0.1000    0.0020
##    100        1.1480             nan     0.1000    0.0014
##    120        1.1220             nan     0.1000    0.0013
##    140        1.0999             nan     0.1000    0.0011
##    150        1.0897             nan     0.1000    0.0011
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0992
##      2        1.5489             nan     0.1000    0.0647
##      3        1.5102             nan     0.1000    0.0470
##      4        1.4815             nan     0.1000    0.0487
##      5        1.4519             nan     0.1000    0.0407
##      6        1.4270             nan     0.1000    0.0354
##      7        1.4058             nan     0.1000    0.0257
##      8        1.3889             nan     0.1000    0.0274
##      9        1.3718             nan     0.1000    0.0251
##     10        1.3563             nan     0.1000    0.0242
##     20        1.2480             nan     0.1000    0.0113
##     40        1.1259             nan     0.1000    0.0055
##     60        1.0480             nan     0.1000    0.0040
##     80        0.9879             nan     0.1000    0.0031
##    100        0.9373             nan     0.1000    0.0033
##    120        0.8937             nan     0.1000    0.0013
##    140        0.8575             nan     0.1000    0.0023
##    150        0.8394             nan     0.1000    0.0017
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1161
##      2        1.5414             nan     0.1000    0.0816
##      3        1.4933             nan     0.1000    0.0622
##      4        1.4556             nan     0.1000    0.0561
##      5        1.4223             nan     0.1000    0.0493
##      6        1.3924             nan     0.1000    0.0415
##      7        1.3665             nan     0.1000    0.0365
##      8        1.3433             nan     0.1000    0.0362
##      9        1.3208             nan     0.1000    0.0349
##     10        1.2992             nan     0.1000    0.0259
##     20        1.1673             nan     0.1000    0.0174
##     40        1.0270             nan     0.1000    0.0063
##     60        0.9356             nan     0.1000    0.0042
##     80        0.8632             nan     0.1000    0.0062
##    100        0.8034             nan     0.1000    0.0045
##    120        0.7516             nan     0.1000    0.0026
##    140        0.7056             nan     0.1000    0.0015
##    150        0.6868             nan     0.1000    0.0028
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0567
##      2        1.5750             nan     0.1000    0.0441
##      3        1.5481             nan     0.1000    0.0360
##      4        1.5262             nan     0.1000    0.0320
##      5        1.5067             nan     0.1000    0.0244
##      6        1.4921             nan     0.1000    0.0232
##      7        1.4784             nan     0.1000    0.0204
##      8        1.4668             nan     0.1000    0.0188
##      9        1.4553             nan     0.1000    0.0153
##     10        1.4457             nan     0.1000    0.0164
##     20        1.3741             nan     0.1000    0.0086
##     40        1.2859             nan     0.1000    0.0050
##     60        1.2293             nan     0.1000    0.0031
##     80        1.1889             nan     0.1000    0.0021
##    100        1.1555             nan     0.1000    0.0017
##    120        1.1283             nan     0.1000    0.0015
##    140        1.1054             nan     0.1000    0.0009
##    150        1.0959             nan     0.1000    0.0012
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0882
##      2        1.5557             nan     0.1000    0.0733
##      3        1.5125             nan     0.1000    0.0489
##      4        1.4827             nan     0.1000    0.0484
##      5        1.4544             nan     0.1000    0.0391
##      6        1.4305             nan     0.1000    0.0339
##      7        1.4098             nan     0.1000    0.0293
##      8        1.3917             nan     0.1000    0.0257
##      9        1.3759             nan     0.1000    0.0268
##     10        1.3590             nan     0.1000    0.0229
##     20        1.2537             nan     0.1000    0.0136
##     40        1.1259             nan     0.1000    0.0066
##     60        1.0448             nan     0.1000    0.0051
##     80        0.9848             nan     0.1000    0.0025
##    100        0.9391             nan     0.1000    0.0022
##    120        0.8979             nan     0.1000    0.0036
##    140        0.8546             nan     0.1000    0.0018
##    150        0.8379             nan     0.1000    0.0022
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1119
##      2        1.5408             nan     0.1000    0.0904
##      3        1.4878             nan     0.1000    0.0680
##      4        1.4477             nan     0.1000    0.0578
##      5        1.4131             nan     0.1000    0.0454
##      6        1.3858             nan     0.1000    0.0377
##      7        1.3625             nan     0.1000    0.0365
##      8        1.3403             nan     0.1000    0.0353
##      9        1.3188             nan     0.1000    0.0288
##     10        1.3008             nan     0.1000    0.0308
##     20        1.1652             nan     0.1000    0.0132
##     40        1.0274             nan     0.1000    0.0060
##     60        0.9312             nan     0.1000    0.0053
##     80        0.8604             nan     0.1000    0.0048
##    100        0.7999             nan     0.1000    0.0049
##    120        0.7515             nan     0.1000    0.0034
##    140        0.7063             nan     0.1000    0.0030
##    150        0.6849             nan     0.1000    0.0014
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0569
##      2        1.5742             nan     0.1000    0.0446
##      3        1.5469             nan     0.1000    0.0346
##      4        1.5249             nan     0.1000    0.0337
##      5        1.5052             nan     0.1000    0.0284
##      6        1.4887             nan     0.1000    0.0233
##      7        1.4749             nan     0.1000    0.0211
##      8        1.4622             nan     0.1000    0.0184
##      9        1.4510             nan     0.1000    0.0161
##     10        1.4411             nan     0.1000    0.0142
##     20        1.3713             nan     0.1000    0.0091
##     40        1.2819             nan     0.1000    0.0054
##     60        1.2259             nan     0.1000    0.0033
##     80        1.1844             nan     0.1000    0.0021
##    100        1.1516             nan     0.1000    0.0019
##    120        1.1242             nan     0.1000    0.0011
##    140        1.1003             nan     0.1000    0.0010
##    150        1.0898             nan     0.1000    0.0010
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0899
##      2        1.5554             nan     0.1000    0.0729
##      3        1.5112             nan     0.1000    0.0474
##      4        1.4819             nan     0.1000    0.0493
##      5        1.4522             nan     0.1000    0.0398
##      6        1.4278             nan     0.1000    0.0307
##      7        1.4090             nan     0.1000    0.0259
##      8        1.3925             nan     0.1000    0.0293
##      9        1.3744             nan     0.1000    0.0270
##     10        1.3573             nan     0.1000    0.0235
##     20        1.2484             nan     0.1000    0.0109
##     40        1.1305             nan     0.1000    0.0059
##     60        1.0502             nan     0.1000    0.0043
##     80        0.9881             nan     0.1000    0.0032
##    100        0.9379             nan     0.1000    0.0036
##    120        0.8933             nan     0.1000    0.0021
##    140        0.8584             nan     0.1000    0.0025
##    150        0.8403             nan     0.1000    0.0022
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1135
##      2        1.5410             nan     0.1000    0.0821
##      3        1.4924             nan     0.1000    0.0611
##      4        1.4551             nan     0.1000    0.0589
##      5        1.4201             nan     0.1000    0.0516
##      6        1.3891             nan     0.1000    0.0412
##      7        1.3636             nan     0.1000    0.0363
##      8        1.3414             nan     0.1000    0.0362
##      9        1.3179             nan     0.1000    0.0293
##     10        1.2990             nan     0.1000    0.0275
##     20        1.1679             nan     0.1000    0.0156
##     40        1.0268             nan     0.1000    0.0061
##     60        0.9357             nan     0.1000    0.0054
##     80        0.8626             nan     0.1000    0.0054
##    100        0.8003             nan     0.1000    0.0040
##    120        0.7530             nan     0.1000    0.0027
##    140        0.7074             nan     0.1000    0.0042
##    150        0.6860             nan     0.1000    0.0023
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0571
##      2        1.5747             nan     0.1000    0.0452
##      3        1.5473             nan     0.1000    0.0359
##      4        1.5253             nan     0.1000    0.0332
##      5        1.5060             nan     0.1000    0.0276
##      6        1.4893             nan     0.1000    0.0212
##      7        1.4761             nan     0.1000    0.0193
##      8        1.4642             nan     0.1000    0.0198
##      9        1.4516             nan     0.1000    0.0152
##     10        1.4416             nan     0.1000    0.0147
##     20        1.3673             nan     0.1000    0.0081
##     40        1.2771             nan     0.1000    0.0051
##     60        1.2184             nan     0.1000    0.0037
##     80        1.1775             nan     0.1000    0.0025
##    100        1.1444             nan     0.1000    0.0015
##    120        1.1174             nan     0.1000    0.0010
##    140        1.0940             nan     0.1000    0.0008
##    150        1.0841             nan     0.1000    0.0009
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0905
##      2        1.5544             nan     0.1000    0.0752
##      3        1.5095             nan     0.1000    0.0580
##      4        1.4742             nan     0.1000    0.0456
##      5        1.4468             nan     0.1000    0.0349
##      6        1.4258             nan     0.1000    0.0359
##      7        1.4044             nan     0.1000    0.0282
##      8        1.3864             nan     0.1000    0.0238
##      9        1.3722             nan     0.1000    0.0253
##     10        1.3557             nan     0.1000    0.0264
##     20        1.2449             nan     0.1000    0.0103
##     40        1.1234             nan     0.1000    0.0061
##     60        1.0427             nan     0.1000    0.0045
##     80        0.9830             nan     0.1000    0.0033
##    100        0.9351             nan     0.1000    0.0021
##    120        0.8920             nan     0.1000    0.0025
##    140        0.8566             nan     0.1000    0.0024
##    150        0.8389             nan     0.1000    0.0023
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1151
##      2        1.5399             nan     0.1000    0.0819
##      3        1.4912             nan     0.1000    0.0590
##      4        1.4549             nan     0.1000    0.0588
##      5        1.4209             nan     0.1000    0.0498
##      6        1.3898             nan     0.1000    0.0448
##      7        1.3628             nan     0.1000    0.0420
##      8        1.3372             nan     0.1000    0.0348
##      9        1.3157             nan     0.1000    0.0297
##     10        1.2978             nan     0.1000    0.0273
##     20        1.1602             nan     0.1000    0.0144
##     40        1.0208             nan     0.1000    0.0073
##     60        0.9292             nan     0.1000    0.0050
##     80        0.8525             nan     0.1000    0.0042
##    100        0.7964             nan     0.1000    0.0043
##    120        0.7438             nan     0.1000    0.0025
##    140        0.7037             nan     0.1000    0.0029
##    150        0.6853             nan     0.1000    0.0020
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0600
##      2        1.5729             nan     0.1000    0.0477
##      3        1.5453             nan     0.1000    0.0381
##      4        1.5224             nan     0.1000    0.0318
##      5        1.5029             nan     0.1000    0.0269
##      6        1.4867             nan     0.1000    0.0226
##      7        1.4729             nan     0.1000    0.0210
##      8        1.4598             nan     0.1000    0.0178
##      9        1.4493             nan     0.1000    0.0168
##     10        1.4387             nan     0.1000    0.0160
##     20        1.3654             nan     0.1000    0.0084
##     40        1.2734             nan     0.1000    0.0048
##     60        1.2178             nan     0.1000    0.0035
##     80        1.1760             nan     0.1000    0.0021
##    100        1.1433             nan     0.1000    0.0014
##    120        1.1157             nan     0.1000    0.0010
##    140        1.0937             nan     0.1000    0.0011
##    150        1.0838             nan     0.1000    0.0009
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0922
##      2        1.5529             nan     0.1000    0.0661
##      3        1.5136             nan     0.1000    0.0594
##      4        1.4778             nan     0.1000    0.0503
##      5        1.4471             nan     0.1000    0.0392
##      6        1.4236             nan     0.1000    0.0345
##      7        1.4027             nan     0.1000    0.0302
##      8        1.3842             nan     0.1000    0.0275
##      9        1.3662             nan     0.1000    0.0243
##     10        1.3505             nan     0.1000    0.0215
##     20        1.2373             nan     0.1000    0.0113
##     40        1.1184             nan     0.1000    0.0052
##     60        1.0397             nan     0.1000    0.0050
##     80        0.9790             nan     0.1000    0.0033
##    100        0.9314             nan     0.1000    0.0026
##    120        0.8876             nan     0.1000    0.0025
##    140        0.8490             nan     0.1000    0.0017
##    150        0.8321             nan     0.1000    0.0028
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1153
##      2        1.5403             nan     0.1000    0.0839
##      3        1.4898             nan     0.1000    0.0617
##      4        1.4520             nan     0.1000    0.0484
##      5        1.4216             nan     0.1000    0.0533
##      6        1.3893             nan     0.1000    0.0492
##      7        1.3602             nan     0.1000    0.0383
##      8        1.3362             nan     0.1000    0.0398
##      9        1.3106             nan     0.1000    0.0317
##     10        1.2902             nan     0.1000    0.0299
##     20        1.1579             nan     0.1000    0.0126
##     40        1.0170             nan     0.1000    0.0073
##     60        0.9239             nan     0.1000    0.0048
##     80        0.8480             nan     0.1000    0.0030
##    100        0.7911             nan     0.1000    0.0048
##    120        0.7424             nan     0.1000    0.0021
##    140        0.6999             nan     0.1000    0.0031
##    150        0.6799             nan     0.1000    0.0024
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0589
##      2        1.5744             nan     0.1000    0.0450
##      3        1.5471             nan     0.1000    0.0350
##      4        1.5259             nan     0.1000    0.0326
##      5        1.5058             nan     0.1000    0.0238
##      6        1.4911             nan     0.1000    0.0228
##      7        1.4776             nan     0.1000    0.0207
##      8        1.4649             nan     0.1000    0.0156
##      9        1.4551             nan     0.1000    0.0160
##     10        1.4450             nan     0.1000    0.0150
##     20        1.3734             nan     0.1000    0.0079
##     40        1.2840             nan     0.1000    0.0047
##     60        1.2279             nan     0.1000    0.0030
##     80        1.1850             nan     0.1000    0.0015
##    100        1.1523             nan     0.1000    0.0013
##    120        1.1269             nan     0.1000    0.0011
##    140        1.1034             nan     0.1000    0.0011
##    150        1.0935             nan     0.1000    0.0007
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0876
##      2        1.5577             nan     0.1000    0.0632
##      3        1.5198             nan     0.1000    0.0586
##      4        1.4850             nan     0.1000    0.0456
##      5        1.4566             nan     0.1000    0.0406
##      6        1.4324             nan     0.1000    0.0344
##      7        1.4115             nan     0.1000    0.0288
##      8        1.3928             nan     0.1000    0.0278
##      9        1.3758             nan     0.1000    0.0222
##     10        1.3620             nan     0.1000    0.0245
##     20        1.2508             nan     0.1000    0.0108
##     40        1.1269             nan     0.1000    0.0053
##     60        1.0455             nan     0.1000    0.0043
##     80        0.9866             nan     0.1000    0.0042
##    100        0.9400             nan     0.1000    0.0027
##    120        0.8982             nan     0.1000    0.0021
##    140        0.8620             nan     0.1000    0.0019
##    150        0.8447             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1093
##      2        1.5426             nan     0.1000    0.0793
##      3        1.4946             nan     0.1000    0.0630
##      4        1.4550             nan     0.1000    0.0466
##      5        1.4261             nan     0.1000    0.0506
##      6        1.3953             nan     0.1000    0.0481
##      7        1.3655             nan     0.1000    0.0362
##      8        1.3428             nan     0.1000    0.0348
##      9        1.3212             nan     0.1000    0.0271
##     10        1.3040             nan     0.1000    0.0279
##     20        1.1671             nan     0.1000    0.0154
##     40        1.0290             nan     0.1000    0.0080
##     60        0.9369             nan     0.1000    0.0042
##     80        0.8661             nan     0.1000    0.0054
##    100        0.8055             nan     0.1000    0.0019
##    120        0.7585             nan     0.1000    0.0028
##    140        0.7145             nan     0.1000    0.0021
##    150        0.6934             nan     0.1000    0.0020
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0585
##      2        1.5742             nan     0.1000    0.0427
##      3        1.5475             nan     0.1000    0.0376
##      4        1.5249             nan     0.1000    0.0329
##      5        1.5056             nan     0.1000    0.0283
##      6        1.4888             nan     0.1000    0.0230
##      7        1.4750             nan     0.1000    0.0202
##      8        1.4630             nan     0.1000    0.0186
##      9        1.4514             nan     0.1000    0.0162
##     10        1.4412             nan     0.1000    0.0145
##     20        1.3697             nan     0.1000    0.0091
##     40        1.2791             nan     0.1000    0.0047
##     60        1.2212             nan     0.1000    0.0026
##     80        1.1801             nan     0.1000    0.0022
##    100        1.1465             nan     0.1000    0.0014
##    120        1.1189             nan     0.1000    0.0008
##    140        1.0957             nan     0.1000    0.0010
##    150        1.0850             nan     0.1000    0.0009
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0876
##      2        1.5550             nan     0.1000    0.0637
##      3        1.5160             nan     0.1000    0.0599
##      4        1.4795             nan     0.1000    0.0474
##      5        1.4512             nan     0.1000    0.0409
##      6        1.4268             nan     0.1000    0.0324
##      7        1.4076             nan     0.1000    0.0320
##      8        1.3878             nan     0.1000    0.0248
##      9        1.3731             nan     0.1000    0.0253
##     10        1.3570             nan     0.1000    0.0218
##     20        1.2461             nan     0.1000    0.0137
##     40        1.1180             nan     0.1000    0.0066
##     60        1.0373             nan     0.1000    0.0053
##     80        0.9762             nan     0.1000    0.0027
##    100        0.9253             nan     0.1000    0.0025
##    120        0.8846             nan     0.1000    0.0032
##    140        0.8482             nan     0.1000    0.0019
##    150        0.8292             nan     0.1000    0.0033
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1158
##      2        1.5405             nan     0.1000    0.0813
##      3        1.4912             nan     0.1000    0.0694
##      4        1.4493             nan     0.1000    0.0586
##      5        1.4141             nan     0.1000    0.0526
##      6        1.3812             nan     0.1000    0.0440
##      7        1.3541             nan     0.1000    0.0358
##      8        1.3315             nan     0.1000    0.0317
##      9        1.3121             nan     0.1000    0.0306
##     10        1.2921             nan     0.1000    0.0266
##     20        1.1577             nan     0.1000    0.0167
##     40        1.0145             nan     0.1000    0.0068
##     60        0.9234             nan     0.1000    0.0050
##     80        0.8518             nan     0.1000    0.0031
##    100        0.7939             nan     0.1000    0.0034
##    120        0.7446             nan     0.1000    0.0031
##    140        0.7032             nan     0.1000    0.0023
##    150        0.6834             nan     0.1000    0.0013
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0585
##      2        1.5740             nan     0.1000    0.0467
##      3        1.5455             nan     0.1000    0.0373
##      4        1.5227             nan     0.1000    0.0325
##      5        1.5030             nan     0.1000    0.0247
##      6        1.4878             nan     0.1000    0.0240
##      7        1.4735             nan     0.1000    0.0208
##      8        1.4607             nan     0.1000    0.0184
##      9        1.4489             nan     0.1000    0.0175
##     10        1.4384             nan     0.1000    0.0138
##     20        1.3660             nan     0.1000    0.0095
##     40        1.2775             nan     0.1000    0.0048
##     60        1.2190             nan     0.1000    0.0032
##     80        1.1782             nan     0.1000    0.0020
##    100        1.1449             nan     0.1000    0.0017
##    120        1.1176             nan     0.1000    0.0011
##    140        1.0948             nan     0.1000    0.0013
##    150        1.0840             nan     0.1000    0.0011
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0913
##      2        1.5545             nan     0.1000    0.0740
##      3        1.5100             nan     0.1000    0.0508
##      4        1.4786             nan     0.1000    0.0476
##      5        1.4494             nan     0.1000    0.0399
##      6        1.4259             nan     0.1000    0.0340
##      7        1.4051             nan     0.1000    0.0311
##      8        1.3858             nan     0.1000    0.0275
##      9        1.3689             nan     0.1000    0.0230
##     10        1.3542             nan     0.1000    0.0230
##     20        1.2442             nan     0.1000    0.0119
##     40        1.1234             nan     0.1000    0.0066
##     60        1.0427             nan     0.1000    0.0058
##     80        0.9834             nan     0.1000    0.0028
##    100        0.9318             nan     0.1000    0.0024
##    120        0.8885             nan     0.1000    0.0026
##    140        0.8516             nan     0.1000    0.0012
##    150        0.8359             nan     0.1000    0.0033
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1158
##      2        1.5387             nan     0.1000    0.0820
##      3        1.4895             nan     0.1000    0.0702
##      4        1.4469             nan     0.1000    0.0499
##      5        1.4164             nan     0.1000    0.0532
##      6        1.3843             nan     0.1000    0.0433
##      7        1.3569             nan     0.1000    0.0363
##      8        1.3351             nan     0.1000    0.0379
##      9        1.3105             nan     0.1000    0.0326
##     10        1.2911             nan     0.1000    0.0287
##     20        1.1596             nan     0.1000    0.0147
##     40        1.0177             nan     0.1000    0.0065
##     60        0.9227             nan     0.1000    0.0058
##     80        0.8520             nan     0.1000    0.0047
##    100        0.7942             nan     0.1000    0.0024
##    120        0.7433             nan     0.1000    0.0032
##    140        0.6971             nan     0.1000    0.0033
##    150        0.6744             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0593
##      2        1.5741             nan     0.1000    0.0462
##      3        1.5465             nan     0.1000    0.0349
##      4        1.5251             nan     0.1000    0.0307
##      5        1.5068             nan     0.1000    0.0277
##      6        1.4908             nan     0.1000    0.0210
##      7        1.4776             nan     0.1000    0.0197
##      8        1.4660             nan     0.1000    0.0184
##      9        1.4548             nan     0.1000    0.0153
##     10        1.4452             nan     0.1000    0.0147
##     20        1.3729             nan     0.1000    0.0089
##     40        1.2842             nan     0.1000    0.0053
##     60        1.2272             nan     0.1000    0.0034
##     80        1.1855             nan     0.1000    0.0023
##    100        1.1530             nan     0.1000    0.0015
##    120        1.1257             nan     0.1000    0.0013
##    140        1.1028             nan     0.1000    0.0010
##    150        1.0926             nan     0.1000    0.0012
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0913
##      2        1.5550             nan     0.1000    0.0657
##      3        1.5155             nan     0.1000    0.0479
##      4        1.4871             nan     0.1000    0.0490
##      5        1.4578             nan     0.1000    0.0440
##      6        1.4315             nan     0.1000    0.0348
##      7        1.4107             nan     0.1000    0.0313
##      8        1.3921             nan     0.1000    0.0265
##      9        1.3759             nan     0.1000    0.0210
##     10        1.3624             nan     0.1000    0.0245
##     20        1.2511             nan     0.1000    0.0109
##     40        1.1321             nan     0.1000    0.0053
##     60        1.0537             nan     0.1000    0.0051
##     80        0.9896             nan     0.1000    0.0021
##    100        0.9383             nan     0.1000    0.0034
##    120        0.8942             nan     0.1000    0.0025
##    140        0.8571             nan     0.1000    0.0029
##    150        0.8392             nan     0.1000    0.0021
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1156
##      2        1.5400             nan     0.1000    0.0809
##      3        1.4910             nan     0.1000    0.0608
##      4        1.4538             nan     0.1000    0.0500
##      5        1.4232             nan     0.1000    0.0402
##      6        1.3989             nan     0.1000    0.0432
##      7        1.3724             nan     0.1000    0.0353
##      8        1.3508             nan     0.1000    0.0372
##      9        1.3277             nan     0.1000    0.0288
##     10        1.3097             nan     0.1000    0.0314
##     20        1.1703             nan     0.1000    0.0123
##     40        1.0241             nan     0.1000    0.0067
##     60        0.9319             nan     0.1000    0.0061
##     80        0.8593             nan     0.1000    0.0052
##    100        0.8023             nan     0.1000    0.0034
##    120        0.7494             nan     0.1000    0.0044
##    140        0.7053             nan     0.1000    0.0025
##    150        0.6838             nan     0.1000    0.0019
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0550
##      2        1.5756             nan     0.1000    0.0413
##      3        1.5491             nan     0.1000    0.0369
##      4        1.5271             nan     0.1000    0.0302
##      5        1.5090             nan     0.1000    0.0270
##      6        1.4925             nan     0.1000    0.0217
##      7        1.4795             nan     0.1000    0.0215
##      8        1.4660             nan     0.1000    0.0165
##      9        1.4557             nan     0.1000    0.0162
##     10        1.4458             nan     0.1000    0.0151
##     20        1.3736             nan     0.1000    0.0076
##     40        1.2832             nan     0.1000    0.0039
##     60        1.2268             nan     0.1000    0.0038
##     80        1.1832             nan     0.1000    0.0024
##    100        1.1502             nan     0.1000    0.0015
##    120        1.1226             nan     0.1000    0.0011
##    140        1.0991             nan     0.1000    0.0011
##    150        1.0886             nan     0.1000    0.0010
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0988
##      2        1.5482             nan     0.1000    0.0641
##      3        1.5107             nan     0.1000    0.0557
##      4        1.4764             nan     0.1000    0.0451
##      5        1.4485             nan     0.1000    0.0374
##      6        1.4265             nan     0.1000    0.0352
##      7        1.4042             nan     0.1000    0.0287
##      8        1.3865             nan     0.1000    0.0269
##      9        1.3693             nan     0.1000    0.0260
##     10        1.3527             nan     0.1000    0.0213
##     20        1.2446             nan     0.1000    0.0111
##     40        1.1260             nan     0.1000    0.0072
##     60        1.0476             nan     0.1000    0.0052
##     80        0.9835             nan     0.1000    0.0037
##    100        0.9323             nan     0.1000    0.0035
##    120        0.8920             nan     0.1000    0.0031
##    140        0.8540             nan     0.1000    0.0022
##    150        0.8366             nan     0.1000    0.0025
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1229
##      2        1.5356             nan     0.1000    0.0798
##      3        1.4880             nan     0.1000    0.0714
##      4        1.4461             nan     0.1000    0.0587
##      5        1.4109             nan     0.1000    0.0497
##      6        1.3812             nan     0.1000    0.0442
##      7        1.3545             nan     0.1000    0.0327
##      8        1.3344             nan     0.1000    0.0337
##      9        1.3135             nan     0.1000    0.0315
##     10        1.2936             nan     0.1000    0.0309
##     20        1.1615             nan     0.1000    0.0152
##     40        1.0227             nan     0.1000    0.0074
##     60        0.9279             nan     0.1000    0.0062
##     80        0.8578             nan     0.1000    0.0052
##    100        0.7998             nan     0.1000    0.0032
##    120        0.7497             nan     0.1000    0.0027
##    140        0.7065             nan     0.1000    0.0028
##    150        0.6874             nan     0.1000    0.0020
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0595
##      2        1.5731             nan     0.1000    0.0454
##      3        1.5453             nan     0.1000    0.0379
##      4        1.5218             nan     0.1000    0.0303
##      5        1.5034             nan     0.1000    0.0266
##      6        1.4873             nan     0.1000    0.0209
##      7        1.4742             nan     0.1000    0.0217
##      8        1.4618             nan     0.1000    0.0168
##      9        1.4519             nan     0.1000    0.0177
##     10        1.4407             nan     0.1000    0.0146
##     20        1.3693             nan     0.1000    0.0096
##     40        1.2767             nan     0.1000    0.0047
##     60        1.2181             nan     0.1000    0.0028
##     80        1.1773             nan     0.1000    0.0024
##    100        1.1452             nan     0.1000    0.0021
##    120        1.1172             nan     0.1000    0.0016
##    140        1.0947             nan     0.1000    0.0008
##    150        1.0842             nan     0.1000    0.0008
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0896
##      2        1.5540             nan     0.1000    0.0638
##      3        1.5150             nan     0.1000    0.0617
##      4        1.4786             nan     0.1000    0.0398
##      5        1.4551             nan     0.1000    0.0440
##      6        1.4298             nan     0.1000    0.0357
##      7        1.4089             nan     0.1000    0.0319
##      8        1.3893             nan     0.1000    0.0281
##      9        1.3721             nan     0.1000    0.0270
##     10        1.3545             nan     0.1000    0.0244
##     20        1.2425             nan     0.1000    0.0116
##     40        1.1169             nan     0.1000    0.0074
##     60        1.0396             nan     0.1000    0.0035
##     80        0.9775             nan     0.1000    0.0035
##    100        0.9246             nan     0.1000    0.0019
##    120        0.8811             nan     0.1000    0.0022
##    140        0.8448             nan     0.1000    0.0016
##    150        0.8290             nan     0.1000    0.0016
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1150
##      2        1.5381             nan     0.1000    0.0803
##      3        1.4892             nan     0.1000    0.0674
##      4        1.4486             nan     0.1000    0.0569
##      5        1.4136             nan     0.1000    0.0500
##      6        1.3832             nan     0.1000    0.0478
##      7        1.3548             nan     0.1000    0.0373
##      8        1.3315             nan     0.1000    0.0295
##      9        1.3128             nan     0.1000    0.0318
##     10        1.2927             nan     0.1000    0.0288
##     20        1.1580             nan     0.1000    0.0162
##     40        1.0155             nan     0.1000    0.0078
##     60        0.9267             nan     0.1000    0.0054
##     80        0.8535             nan     0.1000    0.0042
##    100        0.7928             nan     0.1000    0.0043
##    120        0.7457             nan     0.1000    0.0029
##    140        0.7000             nan     0.1000    0.0019
##    150        0.6809             nan     0.1000    0.0013
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0581
##      2        1.5741             nan     0.1000    0.0462
##      3        1.5464             nan     0.1000    0.0384
##      4        1.5235             nan     0.1000    0.0300
##      5        1.5047             nan     0.1000    0.0269
##      6        1.4885             nan     0.1000    0.0233
##      7        1.4741             nan     0.1000    0.0206
##      8        1.4618             nan     0.1000    0.0166
##      9        1.4515             nan     0.1000    0.0170
##     10        1.4411             nan     0.1000    0.0145
##     20        1.3711             nan     0.1000    0.0079
##     40        1.2853             nan     0.1000    0.0044
##     60        1.2292             nan     0.1000    0.0035
##     80        1.1887             nan     0.1000    0.0012
##    100        1.1552             nan     0.1000    0.0015
##    120        1.1288             nan     0.1000    0.0013
##    140        1.1056             nan     0.1000    0.0015
##    150        1.0954             nan     0.1000    0.0020
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0924
##      2        1.5550             nan     0.1000    0.0746
##      3        1.5106             nan     0.1000    0.0572
##      4        1.4765             nan     0.1000    0.0482
##      5        1.4486             nan     0.1000    0.0398
##      6        1.4247             nan     0.1000    0.0330
##      7        1.4047             nan     0.1000    0.0286
##      8        1.3863             nan     0.1000    0.0272
##      9        1.3697             nan     0.1000    0.0226
##     10        1.3547             nan     0.1000    0.0194
##     20        1.2472             nan     0.1000    0.0128
##     40        1.1281             nan     0.1000    0.0066
##     60        1.0486             nan     0.1000    0.0041
##     80        0.9872             nan     0.1000    0.0030
##    100        0.9381             nan     0.1000    0.0033
##    120        0.8956             nan     0.1000    0.0018
##    140        0.8571             nan     0.1000    0.0018
##    150        0.8394             nan     0.1000    0.0022
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1227
##      2        1.5358             nan     0.1000    0.0911
##      3        1.4815             nan     0.1000    0.0620
##      4        1.4442             nan     0.1000    0.0527
##      5        1.4134             nan     0.1000    0.0493
##      6        1.3836             nan     0.1000    0.0421
##      7        1.3562             nan     0.1000    0.0354
##      8        1.3347             nan     0.1000    0.0320
##      9        1.3152             nan     0.1000    0.0295
##     10        1.2966             nan     0.1000    0.0288
##     20        1.1637             nan     0.1000    0.0159
##     40        1.0224             nan     0.1000    0.0061
##     60        0.9328             nan     0.1000    0.0049
##     80        0.8604             nan     0.1000    0.0046
##    100        0.8018             nan     0.1000    0.0025
##    120        0.7491             nan     0.1000    0.0033
##    140        0.7053             nan     0.1000    0.0026
##    150        0.6846             nan     0.1000    0.0029
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0589
##      2        1.5730             nan     0.1000    0.0478
##      3        1.5440             nan     0.1000    0.0348
##      4        1.5229             nan     0.1000    0.0339
##      5        1.5030             nan     0.1000    0.0275
##      6        1.4861             nan     0.1000    0.0235
##      7        1.4720             nan     0.1000    0.0202
##      8        1.4598             nan     0.1000    0.0190
##      9        1.4480             nan     0.1000    0.0161
##     10        1.4384             nan     0.1000    0.0138
##     20        1.3667             nan     0.1000    0.0094
##     40        1.2766             nan     0.1000    0.0050
##     60        1.2189             nan     0.1000    0.0030
##     80        1.1756             nan     0.1000    0.0021
##    100        1.1419             nan     0.1000    0.0018
##    120        1.1145             nan     0.1000    0.0012
##    140        1.0912             nan     0.1000    0.0012
##    150        1.0807             nan     0.1000    0.0009
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.0940
##      2        1.5540             nan     0.1000    0.0737
##      3        1.5102             nan     0.1000    0.0616
##      4        1.4752             nan     0.1000    0.0485
##      5        1.4453             nan     0.1000    0.0343
##      6        1.4246             nan     0.1000    0.0341
##      7        1.4043             nan     0.1000    0.0305
##      8        1.3860             nan     0.1000    0.0277
##      9        1.3676             nan     0.1000    0.0263
##     10        1.3512             nan     0.1000    0.0222
##     20        1.2407             nan     0.1000    0.0136
##     40        1.1145             nan     0.1000    0.0059
##     60        1.0361             nan     0.1000    0.0049
##     80        0.9790             nan     0.1000    0.0031
##    100        0.9285             nan     0.1000    0.0032
##    120        0.8855             nan     0.1000    0.0023
##    140        0.8491             nan     0.1000    0.0034
##    150        0.8300             nan     0.1000    0.0026
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1180
##      2        1.5400             nan     0.1000    0.0816
##      3        1.4897             nan     0.1000    0.0635
##      4        1.4515             nan     0.1000    0.0597
##      5        1.4162             nan     0.1000    0.0483
##      6        1.3856             nan     0.1000    0.0474
##      7        1.3571             nan     0.1000    0.0374
##      8        1.3347             nan     0.1000    0.0379
##      9        1.3115             nan     0.1000    0.0337
##     10        1.2895             nan     0.1000    0.0283
##     20        1.1546             nan     0.1000    0.0135
##     40        1.0116             nan     0.1000    0.0075
##     60        0.9189             nan     0.1000    0.0034
##     80        0.8484             nan     0.1000    0.0051
##    100        0.7926             nan     0.1000    0.0040
##    120        0.7419             nan     0.1000    0.0048
##    140        0.6961             nan     0.1000    0.0023
##    150        0.6760             nan     0.1000    0.0028
## 
## Iter   TrainDeviance   ValidDeviance   StepSize   Improve
##      1        1.6094             nan     0.1000    0.1190
##      2        1.5361             nan     0.1000    0.0794
##      3        1.4879             nan     0.1000    0.0600
##      4        1.4512             nan     0.1000    0.0460
##      5        1.4222             nan     0.1000    0.0455
##      6        1.3934             nan     0.1000    0.0480
##      7        1.3651             nan     0.1000    0.0396
##      8        1.3410             nan     0.1000    0.0328
##      9        1.3206             nan     0.1000    0.0334
##     10        1.3006             nan     0.1000    0.0275
##     20        1.1709             nan     0.1000    0.0132
##     40        1.0381             nan     0.1000    0.0060
##     60        0.9477             nan     0.1000    0.0048
##     80        0.8774             nan     0.1000    0.0027
##    100        0.8201             nan     0.1000    0.0036
##    120        0.7736             nan     0.1000    0.0024
##    140        0.7311             nan     0.1000    0.0012
##    150        0.7136             nan     0.1000    0.0016
```

```r
print(modFit$finalModel); print(modFit$results)
```

```
## A gradient boosted model with multinomial loss function.
## 150 iterations were performed.
## There were 12 predictors of which 12 had non-zero influence.
```

```
##   shrinkage interaction.depth n.minobsinnode n.trees  Accuracy     Kappa
## 1       0.1                 1             10      50 0.5056580 0.3647573
## 4       0.1                 2             10      50 0.5971195 0.4882277
## 7       0.1                 3             10      50 0.6496023 0.5556807
## 2       0.1                 1             10     100 0.5528246 0.4296121
## 5       0.1                 2             10     100 0.6563703 0.5642161
## 8       0.1                 3             10     100 0.7123485 0.6354410
## 3       0.1                 1             10     150 0.5764124 0.4608738
## 6       0.1                 2             10     150 0.6925386 0.6101210
## 9       0.1                 3             10     150 0.7483685 0.6811817
##    AccuracySD     KappaSD
## 1 0.006678787 0.008722804
## 4 0.006655618 0.008466309
## 7 0.006250145 0.008152597
## 2 0.004840211 0.006163796
## 5 0.005410903 0.006883575
## 8 0.005668186 0.007233797
## 3 0.005077862 0.006485974
## 6 0.005062459 0.006350517
## 9 0.004268253 0.005487587
```

```r
pgbm <- predict(modFit, trainPC[,-13])
confusionMatrix(trainPC$classe, pgbm)
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction    A    B    C    D    E
##          A 4757  181  261  324   57
##          B  419 2748  382  156   92
##          C  325  205 2737   84   71
##          D  177  108  348 2505   78
##          E  198  251  287  157 2714
## 
## Overall Statistics
##                                           
##                Accuracy : 0.7879          
##                  95% CI : (0.7822, 0.7936)
##     No Information Rate : 0.2995          
##     P-Value [Acc > NIR] : < 2.2e-16       
##                                           
##                   Kappa : 0.7314          
##  Mcnemar's Test P-Value : < 2.2e-16       
## 
## Statistics by Class:
## 
##                      Class: A Class: B Class: C Class: D Class: E
## Sensitivity            0.8096   0.7867   0.6817   0.7765   0.9011
## Specificity            0.9401   0.9350   0.9561   0.9566   0.9462
## Pos Pred Value         0.8525   0.7237   0.7998   0.7789   0.7524
## Neg Pred Value         0.9203   0.9529   0.9211   0.9561   0.9814
## Prevalence             0.2995   0.1780   0.2046   0.1644   0.1535
## Detection Rate         0.2424   0.1400   0.1395   0.1277   0.1383
## Detection Prevalence   0.2844   0.1935   0.1744   0.1639   0.1838
## Balanced Accuracy      0.8748   0.8608   0.8189   0.8666   0.9236
```

As the results show, the accuracy rate is around 79%.  

## Prediction  

Finally, we'd apply the fitted model to predict the test data set.  


```r
test <- test[,c("problem_id", "roll_belt", "pitch_belt", "yaw_belt", "total_accel_belt",
                "gyros_belt_x", "gyros_belt_y", "gyros_belt_z",
                "accel_belt_x", "accel_belt_y", "accel_belt_z",
                "magnet_belt_x", "magnet_belt_y", "magnet_belt_z",
                "roll_arm", "pitch_arm", "yaw_arm", "total_accel_arm",
                "gyros_arm_x", "gyros_arm_y", "gyros_arm_z",
                "accel_arm_x", "accel_arm_y", "accel_arm_z",
                "magnet_arm_x", "magnet_arm_y", "magnet_arm_z",
                "roll_dumbbell", "pitch_dumbbell", "yaw_dumbbell", "total_accel_dumbbell",
                "gyros_dumbbell_x", "gyros_dumbbell_y", "gyros_dumbbell_z",
                "accel_dumbbell_x", "accel_dumbbell_y", "accel_dumbbell_z",
                "magnet_dumbbell_x", "magnet_dumbbell_y", "magnet_dumbbell_z",
                "roll_forearm", "pitch_forearm", "yaw_forearm", "total_accel_forearm",
                "gyros_forearm_x", "gyros_forearm_y", "gyros_forearm_z",
                "accel_forearm_x", "accel_forearm_y", "accel_forearm_z",
                "magnet_forearm_x", "magnet_forearm_y", "magnet_forearm_z")]
testPC <- predict(preproc, test[,-1])
predict(modFit, testPC[,-13])
```

```
##  [1] C C A A A E D B A A A C B A E E A B D B
## Levels: A B C D E
```
