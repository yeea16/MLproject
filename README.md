# 
---
title: "Prediction Assignment Writeup"
author: "YEEA"
date: "11/23/2020"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, warning=FALSE, message=FALSE)
```

## Overview
Quantification of human movement and its use for predicting the activity that the subject is actively engage in is become more and more important. Here, we will use some data, collected by [http://groupware.les.inf.puc-rio.br/har](http://groupware.les.inf.puc-rio.br/har), which contains different accelerometer data from six subjects, which were requested to lift Barbel weights in the correct way and in five
incorrect fashions.

We split the available working data into a training and a testing dataset, accounting for 60% and 40% of the total number of experimental observations. We preprocess the data to remove variables with high quantities of missing data, variables accounting for temporal evolution, and then apply principal component analysis to extract the most variable variables to apply to them a machine learning algorithm.

We train different models of machine learning and compare their performance using the testing dataset. The highest accuracy is obtained using random forests. We finally use that model to predict the type of lift for each of the twenty observations.

## Preprocessing and exploratory data analysis
We load the required libraries
```{r}
library(caret)
library(Rmisc)
library(dplyr)
library(corrplot)
library(randomForest)
```

We start reading the data. We will use the testing data for final prediction and the training data as our working dataset, both for the training and testing of the model.
```{r, comment=""}
trainingfile="pml-training.csv"
if(!file.exists(trainingfile)){
        trainingurl="https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
        download.file(trainingurl,trainingfile)
}
testingfile="pml-testing.csv"
if(!file.exists(testingfile)){
        testingurl="https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
        download.file(testingurl,testingfile)
}
working=read.csv(trainingfile)
prediction=read.csv(testingfile)
dim(working)
```
There are 19622 observables across 160 variables, one of which is the *classe* variable that we are trying to predict. There is a high number of `NA` values in some of the variables, which can reduce de prediction efficiency. We remove those variables for which the amount of NA values is above one quarter of the total number of observables. There are also some temporal variables, which we remove too due to the difficulty in using them for the prediction. We remove the same variables from the prediction dataset
```{r}
maxnaratio=0.25
nmaxna=nrow(working)*maxnaratio
indcolremove=c(grep("user|window|timestamp",names(working)),which(colSums(is.na(working)|working=="")>nmaxna))
working=working[,-indcolremove]
prediction=prediction[,-indcolremove]
```

We convert the *classe* variable to a factor variable and notice that *classe* is the 54th variable in the preprocessed dataset.
```{r, comment=""}
working$classe=as.factor(working$classe)
which(names(working)=="classe")
```

We use the caret package to partition the working dataset into a training dataset, cointaining 60% of the data, and a testing dataset, containing the remaining 40%.
```{r}
indtraining=createDataPartition(working$classe,p=0.6,list=FALSE)
training=working[indtraining,]
testing=working[-indtraining,]
```

We try to find any correlation between *classe* and the rest of the variables. We select only the other four variables which show at least a 25% correlation with *classe*
```{r, comment=""}
correlations=cor(training[,-c(1,54)],as.numeric(training$classe))
bestcorrelations = subset(as.data.frame(as.table(correlations)), abs(Freq)>0.25)
bestcorrelations=arrange(bestcorrelations,desc(abs(bestcorrelations$Freq)))[,c(1,3)]
bestcorrelations
```

And create a plot showing the distribution of those four variables across the 5 different classes. We do not see a clear pattern that would allow to identify the *classe* variable depending on the value of those four variables.
```{r,cache=TRUE}
p1=ggplot(training,aes(classe,pitch_forearm))+geom_boxplot(aes(fill=classe))
p2=ggplot(training,aes(classe,magnet_arm_x))+geom_boxplot(aes(fill=classe))
p3=ggplot(training,aes(classe,magnet_belt_y))+geom_boxplot(aes(fill=classe))
p4=ggplot(training,aes(classe,magnet_arm_y))+geom_boxplot(aes(fill=classe))
multiplot(p1,p2,p3,p4,cols=2)
```

## Machine Learning Models

First, we prepocess the data using Principal Component Analysis to extract the principal components of the data, *i.e.*, we construct variable with decreased variability in the data.
```{r}
pcapp_all=preProcess(training[,-54],method="pca")
```
We apply the PCA to all the three training, testing and prediction datasets. We have reduced the number of variables from 53 to 25.
```{r, comment=""}
training_pca_all=predict(pcapp_all,training[,-54])
testing_pca_all=predict(pcapp_all,testing[,-54])
prediction_pca_all=predict(pcapp_all,prediction)
dim(training_pca_all)[2]
```

We create a new dataset for the training data, containing the PCA-preprocessed training data together with the outcome variable, *classe*:
```{r}
trainingdata_pca_all=cbind(training_pca_all,classe=training$classe)
```

We now use that dataset to train three different models: one with trees, using *rpart*; another one with random forest, using *rf*; and the last one a gradient method, *gbm*.
```{r, cache=TRUE}
modelrpartpca=train(classe~.,method="rpart",data=trainingdata_pca_all)
modelrfpca=train(classe~.,method="rf",data=trainingdata_pca_all)
modelgbmpca=train(classe~.,method="gbm",data=trainingdata_pca_all)
```

Sometimes, there may be variables that are highly correlated with each other. We can try to simplify the model by removing some of that variables. We create a correlation matrix and calculate the indices of the seven variables to be removed in order to remove pairwise correlation from the dataset at a 90% correlation threshold.
```{r, comment=""}
correlationmatrix=cor(training[,-54])
indstrongcorrelation=findCorrelation(correlationmatrix)
length(indstrongcorrelation)
```

We apply the same Principal Component Analysis as before, but removing those seven variables. We create a new dataset containing the PCA-preprocessed training data and train the same three models as before.
```{r, cache=TRUE}
pcapp_exccorr=preProcess(training[,-c(indstrongcorrelation,54)],method="pca")
training_pca_exccorr=predict(pcapp_exccorr,training[,-c(indstrongcorrelation,54)])
testing_pca_exccorr=predict(pcapp_exccorr,testing[,-c(indstrongcorrelation,54)])
prediction_pca_exccorr=predict(pcapp_exccorr,prediction[-indstrongcorrelation])
trainingdata_pca_exccorr=cbind(training_pca_exccorr,classe=training$classe)
modelrpartpcaexc=train(classe~.,method="rpart",data=trainingdata_pca_exccorr)
modelrfpcaexc=train(classe~.,method="rf",data=trainingdata_pca_exccorr)
modelgbmpcaexc=train(classe~.,method="gbm",data=trainingdata_pca_exccorr)
```

We know apply the six trained models to predict the *classe* values using the corresponding PCA-preprocessed testing data and calculate the percentage of agreement between our predictions and the actual values for *classe*
```{r}
ntest=length(testing$classe)
sum(predict(modelrpartpca,newdata=testing_pca_all)==testing$classe)/ntest*100
sum(predict(modelrfpca,newdata=testing_pca_all)==testing$classe)/ntest*100
sum(predict(modelgbmpca,newdata=testing_pca_all)==testing$classe)/ntest*100
sum(predict(modelrpartpcaexc,newdata=testing_pca_exccorr)==testing$classe)/ntest*100
sum(predict(modelrfpcaexc,newdata=testing_pca_exccorr)==testing$classe)/ntest*100
sum(predict(modelgbmpcaexc,newdata=testing_pca_exccorr)==testing$classe)/ntest*100
```

We observe that the highest accuracy (99.03%) is obtained for the random forest model without removing highly inter-correlated variables. We now use that model to predict the *classe* variable fo the prediction data.
```{r}
predict(modelrfpca,newdata=prediction_pca_all)
```---
title: "Prediction Assignment Writeup"
author: "YEEA"
date: "11/23/2020"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, warning=FALSE, message=FALSE)
```

## Overview
Quantification of human movement and its use for predicting the activity that the subject is actively engage in is become more and more important. Here, we will use some data, collected by [http://groupware.les.inf.puc-rio.br/har](http://groupware.les.inf.puc-rio.br/har), which contains different accelerometer data from six subjects, which were requested to lift Barbel weights in the correct way and in five
incorrect fashions.

We split the available working data into a training and a testing dataset, accounting for 60% and 40% of the total number of experimental observations. We preprocess the data to remove variables with high quantities of missing data, variables accounting for temporal evolution, and then apply principal component analysis to extract the most variable variables to apply to them a machine learning algorithm.

We train different models of machine learning and compare their performance using the testing dataset. The highest accuracy is obtained using random forests. We finally use that model to predict the type of lift for each of the twenty observations.

## Preprocessing and exploratory data analysis
We load the required libraries
```{r}
library(caret)
library(Rmisc)
library(dplyr)
library(corrplot)
library(randomForest)
```

We start reading the data. We will use the testing data for final prediction and the training data as our working dataset, both for the training and testing of the model.
```{r, comment=""}
trainingfile="pml-training.csv"
if(!file.exists(trainingfile)){
        trainingurl="https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
        download.file(trainingurl,trainingfile)
}
testingfile="pml-testing.csv"
if(!file.exists(testingfile)){
        testingurl="https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
        download.file(testingurl,testingfile)
}
working=read.csv(trainingfile)
prediction=read.csv(testingfile)
dim(working)
```
There are 19622 observables across 160 variables, one of which is the *classe* variable that we are trying to predict. There is a high number of `NA` values in some of the variables, which can reduce de prediction efficiency. We remove those variables for which the amount of NA values is above one quarter of the total number of observables. There are also some temporal variables, which we remove too due to the difficulty in using them for the prediction. We remove the same variables from the prediction dataset
```{r}
maxnaratio=0.25
nmaxna=nrow(working)*maxnaratio
indcolremove=c(grep("user|window|timestamp",names(working)),which(colSums(is.na(working)|working=="")>nmaxna))
working=working[,-indcolremove]
prediction=prediction[,-indcolremove]
```

We convert the *classe* variable to a factor variable and notice that *classe* is the 54th variable in the preprocessed dataset.
```{r, comment=""}
working$classe=as.factor(working$classe)
which(names(working)=="classe")
```

We use the caret package to partition the working dataset into a training dataset, cointaining 60% of the data, and a testing dataset, containing the remaining 40%.
```{r}
indtraining=createDataPartition(working$classe,p=0.6,list=FALSE)
training=working[indtraining,]
testing=working[-indtraining,]
```

We try to find any correlation between *classe* and the rest of the variables. We select only the other four variables which show at least a 25% correlation with *classe*
```{r, comment=""}
correlations=cor(training[,-c(1,54)],as.numeric(training$classe))
bestcorrelations = subset(as.data.frame(as.table(correlations)), abs(Freq)>0.25)
bestcorrelations=arrange(bestcorrelations,desc(abs(bestcorrelations$Freq)))[,c(1,3)]
bestcorrelations
```

And create a plot showing the distribution of those four variables across the 5 different classes. We do not see a clear pattern that would allow to identify the *classe* variable depending on the value of those four variables.
```{r,cache=TRUE}
p1=ggplot(training,aes(classe,pitch_forearm))+geom_boxplot(aes(fill=classe))
p2=ggplot(training,aes(classe,magnet_arm_x))+geom_boxplot(aes(fill=classe))
p3=ggplot(training,aes(classe,magnet_belt_y))+geom_boxplot(aes(fill=classe))
p4=ggplot(training,aes(classe,magnet_arm_y))+geom_boxplot(aes(fill=classe))
multiplot(p1,p2,p3,p4,cols=2)
```

## Machine Learning Models

First, we prepocess the data using Principal Component Analysis to extract the principal components of the data, *i.e.*, we construct variable with decreased variability in the data.
```{r}
pcapp_all=preProcess(training[,-54],method="pca")
```
We apply the PCA to all the three training, testing and prediction datasets. We have reduced the number of variables from 53 to 25.
```{r, comment=""}
training_pca_all=predict(pcapp_all,training[,-54])
testing_pca_all=predict(pcapp_all,testing[,-54])
prediction_pca_all=predict(pcapp_all,prediction)
dim(training_pca_all)[2]
```

We create a new dataset for the training data, containing the PCA-preprocessed training data together with the outcome variable, *classe*:
```{r}
trainingdata_pca_all=cbind(training_pca_all,classe=training$classe)
```

We now use that dataset to train three different models: one with trees, using *rpart*; another one with random forest, using *rf*; and the last one a gradient method, *gbm*.
```{r, cache=TRUE}
modelrpartpca=train(classe~.,method="rpart",data=trainingdata_pca_all)
modelrfpca=train(classe~.,method="rf",data=trainingdata_pca_all)
modelgbmpca=train(classe~.,method="gbm",data=trainingdata_pca_all)
```

Sometimes, there may be variables that are highly correlated with each other. We can try to simplify the model by removing some of that variables. We create a correlation matrix and calculate the indices of the seven variables to be removed in order to remove pairwise correlation from the dataset at a 90% correlation threshold.
```{r, comment=""}
correlationmatrix=cor(training[,-54])
indstrongcorrelation=findCorrelation(correlationmatrix)
length(indstrongcorrelation)
```

We apply the same Principal Component Analysis as before, but removing those seven variables. We create a new dataset containing the PCA-preprocessed training data and train the same three models as before.
```{r, cache=TRUE}
pcapp_exccorr=preProcess(training[,-c(indstrongcorrelation,54)],method="pca")
training_pca_exccorr=predict(pcapp_exccorr,training[,-c(indstrongcorrelation,54)])
testing_pca_exccorr=predict(pcapp_exccorr,testing[,-c(indstrongcorrelation,54)])
prediction_pca_exccorr=predict(pcapp_exccorr,prediction[-indstrongcorrelation])
trainingdata_pca_exccorr=cbind(training_pca_exccorr,classe=training$classe)
modelrpartpcaexc=train(classe~.,method="rpart",data=trainingdata_pca_exccorr)
modelrfpcaexc=train(classe~.,method="rf",data=trainingdata_pca_exccorr)
modelgbmpcaexc=train(classe~.,method="gbm",data=trainingdata_pca_exccorr)
```

We know apply the six trained models to predict the *classe* values using the corresponding PCA-preprocessed testing data and calculate the percentage of agreement between our predictions and the actual values for *classe*
```{r}
ntest=length(testing$classe)
sum(predict(modelrpartpca,newdata=testing_pca_all)==testing$classe)/ntest*100
sum(predict(modelrfpca,newdata=testing_pca_all)==testing$classe)/ntest*100
sum(predict(modelgbmpca,newdata=testing_pca_all)==testing$classe)/ntest*100
sum(predict(modelrpartpcaexc,newdata=testing_pca_exccorr)==testing$classe)/ntest*100
sum(predict(modelrfpcaexc,newdata=testing_pca_exccorr)==testing$classe)/ntest*100
sum(predict(modelgbmpcaexc,newdata=testing_pca_exccorr)==testing$classe)/ntest*100
```

We observe that the highest accuracy (99.03%) is obtained for the random forest model without removing highly inter-correlated variables. We now use that model to predict the *classe* variable fo the prediction data.
```{r}
predict(modelrfpca,newdata=prediction_pca_all)
```
