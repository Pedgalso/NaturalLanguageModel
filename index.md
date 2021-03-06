---
title       : Natural Language Model
subtitle    : Next word predictor
author      : Pedro Solera
date        : "31 May 2020"
job         : Data Science Capstone project - Coursera
framework   : io2012        # {io2012, html5slides, shower, dzslides, ...}
highlighter : highlight.js  # {highlight.js, prettify, highlight}
hitheme     : tomorrow      # 
widgets     : []            # {mathjax, quiz, bootstrap}
mode        : selfcontained # {standalone, draft}
knit        : slidify::knit2slides
github      :
  user: Pedgalso
  repo: NaturalLanguageModel
---

<style type="text/css">

strong {
  font-weight: bold;
}

code.r{
  font-size: 10px;
}
</style>

## Model information

### Model description

1. Prediction model based upon **Ngrams**
2. Observerd Ngrams are generated from **twitter**, **blogs** and **news**
3. Predicted Words are selected according to **Maximum Likelihood Estimate**
4. **Katz Back-Off Model** is used to generate words not observed in the Ngrams

### Model characteristics 

Model improvement have been focused on:

1. Increased performance time (response <3s )
2. Size of model (~11MB)  
3. Improved word prediction 

--- #DataGeneration &twocol

## Data generation

*** =left


```r
# twitter, news and blog are the provided raw data
data.sample <- c(sample(twitter, length(twitter) * 0.01),
                 sample(news, length(news) * 0.02),
                 sample(blog, length(blog) * 0.02))

# Required functions to process raw data

NLPBigramTokenizer <- function(x) {
  unlist(lapply(ngrams(words(x), 2), paste, collapse = " "),
         use.names = FALSE)
}

NLPTrigramTokenizer <- function(x) {
  unlist(lapply(ngrams(words(x), 3), paste, collapse = " "),
         use.names = FALSE)
}

Freq <- function(x){
  a<-data.frame(x$i,x$j,x$v)
  a<-aggregate(x$v, by=list(x$i), sum)
  colnames(a)<-c("wordIndex", "frequency")
  a<-a[order(a$frequency, decreasing = TRUE),]
  a$words<-x$dimnames$Terms[a$wordIndex]
  a<-a[,2:3]
  return(a)}
```

*** =right


```r

library(tm)

corpus <- VCorpus(VectorSource(data.sample))
corpus <- tm_map(corpus, removePunctuation)              
corpus <- tm_map(corpus, content_transformer(removeMostPunctuation))
corpus <- tm_map(corpus, removeNumbers)                     
corpus <- tm_map(corpus, tolower)                          
corpus <- tm_map(corpus, stripWhitespace) 
# corpus <- tm_map(corpus, removeWords, stopwords("en"))
corpus <- tm_map(corpus, PlainTextDocument)

library(ngram)

# Onegram<-TermDocumentMatrix(corpus)
Bigram<-TermDocumentMatrix(corpus, control=list(tokenize=NLPBigramTokenizer))
Trigram<-TermDocumentMatrix(corpus, control = list(tokenize=NLPTrigramTokenizer))

# OnegramData<-Freq(Onegram)
BigramData <-Freq(Bigram)
TrigramData<-Freq(Trigram)

rownames(BigramData)<-NULL
rownames(TrigramData)<-NULL
```

--- #modelCode &twocol
  
## Model code
  
*** =left
  

```r

library(tm)
library(ngram)
  
## Pre processing the input text for model function
prepro<- function (x){
    
# perform removepunctuation, removenumbers, tolower, stripWhitespace
    a<-sapply(x, function (x) preprocess(x ,case="lower",
                                         remove.punct = TRUE,
                                         remove.numbers = TRUE,
                                         fix.spacing = TRUE))
    attr(a, "names") <- NULL
# perform bigram or onegram 
    Ngram<- ngram_asweka(a, min=2, max=2)
    
    if (length(Ngram)==0) Ngram<- ngram_asweka(a, min=1, max=1)
return(Ngram)}
  
# Function required in the model, it takes the last word of a Ngram
lastword <- function(x1){
  a <- ngram_asweka(x1, min=1, max=1)
  a <- a[length(a)] 
return(a)}
```
  
*** =right
  

```r
  
# Model function
model <- function (x, TrigramData, BigramData, gamma2= 0.5, gamma3=0.5) {
  
# Last Ngram of the user input data
  df<-x[length(x)]
# look up in trigram database for matches
  s3<- data.frame()
  if (length(ngram_asweka(df, min=1, max=1))>1){
    s3<-TrigramData[grep(paste0("^", df, " "), TrigramData$words),]
# keep only top10 matches
    if (dim(s3)[1] >10) s3<-s3[1:10, ]
    p3 <- (s3$frequency/ sum(s3$frequency))*gamma3
    s3$probability <- p3}
# look up in bigram database for matches
  lw<-lastword(df)
  s2<-BigramData[grep(paste0("^", lw, " "), BigramData$words),]
# keep only top10 
  if (dim(s2)[1] >10) s2<-s2[1:10, ]
    p2 <- (s2$frequency/ sum(s2$frequency))*gamma2
    s2$probability <- p2
# Join together bigram and trigram matches
  if (nrow(s3)>0) s<-rbind(s3,s2)
  else  s<-s2
# take last word from each bigram and trigram, these are the predicted words 
  s$words<-sapply(s$words, lastword)
# drop frequency column 
  s$frequency <- NULL
# join together predicted words from trigrams&bigrams
  s<-aggregate(s$probability, by=list(s$word), sum)
  s<-s[order(s$x, decreasing=TRUE),]
  names(s)<-c("Words", "Probability")
  
return(s)}
```

--- #wayforward

## Way forward

To improve the model, the following activities are proposed:

1. Model requires **further validation** (tunning up gamma2 and gamma3 variables) 
2. **Sentence context** could added to the model by accounting for:
 - Markov Chains model, that can be used to set new probabilities distributions to the Ngrams
 - Using Ngrams without [stopwords](https://en.wikipedia.org/wiki/Stop_words)
3. **Alternative models** could be developed and embedded to the present one, such as Bert or GPT-2. 

