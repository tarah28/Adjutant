[![Build Status](https://travis-ci.com/amcrisan/Adjutant.svg?token=Vvo9EBpsrenpSQP9qszF&branch=master)](https://travis-ci.com/amcrisan/Adjutant)

1. [Overview](#overview)
    * [Citation](#citation)
2. [Download](#download)
3. [Demo](#demo)
    * [Using the Adjuntant Shiny App](#using-the-adjuntant-shiny-app)
    * [Using Adjutant within a R script](#using-adjutant-within-a-R-script)

## Overview 
Adjutant is an open-source, interactive, and R-based application to support mining PubMed for a systematic or a literature review. Given a PubMed-compatible search query, Adjutant  downloads the relevant articles and allows the user to perform an unsupervised clustering analysis to identify data-driven topic clusters. Users can also sample documents using different strategies to obtain a more manageable dataset for further analysis. Adjutant makes explicit trade-offs between speed and accuracy, which are modifiable by the user, such that a complete analysis of several thousand documents can take a few minutes. All analytic datasets generated by Adjutant are saved, allowing users to easily conduct other downstream analyses that Adjutant does not explicitly support.

### Citation
If you use Adjutant, please cite:

> Anamaria Crisan, Tamara Munzner, and Jennifer Gardy (2017)
> Adjutant: an R-based tool to support topic discovery for systematic and literature review
> url: https://github.com/amcrisan/Adjutant/

## Download

Download the latest development code of Adjutant from GitHub using [devtools](https://cran.r-project.org/package=devtools) with

```R
devtools::install_github("amcrisan/Adjutant")
```

### Download: Super new to R version 

**Video coming soon**

#### Getting R and Install Adjutant
Maybe you want to use Adjutant (you read about it, you saw it somewhere, you're my mom), but you don't really know what R is, so you're not sure where to start. 

First, you need to [download R](https://cloud.r-project.org/) onto your computer. You may run into trouble if you work in a place that prevents you from downloading and installing applications to your computer - this means you might need to us a home computer (but also talk to your IT team about why R is the best thing ever). You can also [download RStudio](https://www.rstudio.com/products/rstudio/download/), a useful tool for running R code and working on your own R code. You don't need RStudio to run Adjutant, but it can make things easier.

Second, you need to install the [devtools package](https://cran.r-project.org/web/packages/devtools/index.html). You can do this by opening R (can be an icon on your desktop) or by opening RStudio.

Within R or the RStudio **console**, type the following:

```R
install.packages("devtools")
```

Now we can install Adjutant! So type the following:

```R
devtools::install_github("amcrisan/Adjutant")
```

## Demo

Adjutant can be used through its attendant Shiny Application or within your own R script, although **the design of Adjutant is driven more towards the Shiny App usage**. Adjutant can be used just for looking up and downloading articles from PubMed to your computer, or performing a topic clustering analysis and/or a sophisticated document sampling. 

### Using the Adjuntant Shiny App

**Video coming soon**

With Adjutant installed, you just need to type two commands to get it going.

```R
library(adjutant) #this gets R ready to run Adjutant
runAdjutant() #this will launch Adjutant user interface
```

That's it! Have a lot of fun exploring!

### Using Adjutant within a R script

It is also possible to use Adjutant within your own code, bypassing the Shiny application all together.

**Step 0: Load Adjutant**

To run this demo the first thing you need to do is load Adjutant (once you've downloaded it), and some additional packages for analysis.

```R
library(adjutant)
library(dplyr)
library(ggplot2)

#also set a seed - there is some randomness in the analysis.
set.seed(416)
```

**Step 1: Downloading data from PubMed to your computer**

**processSearch** is Adjutant's PubMed search function and is effectively a wrapper for [RISmed](https://cran.r-project.org/web/packages/RISmed/RISmed.pdf) that formats RISmed's output into a clean data frame, with additional PubMed metadata (PubMed central citation count, language, article type etc). You can pass RISmed's EUtilsSummary parameters to the Adjutant's **processSearch** function.

Please note, that Adjutant's downstream methods *expect* and dataframe with the column names that are produced by the **processSearch** method.

*Depending upon the size of the query, it can take a few minutes to download all of the data.*

```R
df<-processSearch("(outbreak OR epidemic OR pandemic) AND genom*",retmax=2000)
```
**Step 2: Generating a tidy text corpus**

Adjutant next constructs a per-article ["bag-of-words"](https://en.wikipedia.org/wiki/Bag-of-words_model) arranged in a [tidy text format](https://www.tidytextmining.com/tidytext.html). This step is necessary for further analysis in Adjutant, but can also be an interesting data set for other kinds of analysis not supported by Adjutant.

```R
tidy_df<-tidyCorpus(corpus = df)
```
**Step 3: Performing a dimensionality reduction using t-SNE**

To learn more about t-SNE, please consult this [excellent article in Distill Pub](https://distill.pub/2016/misread-tsne/). Generally, Adjutant's topic clustering works best when there are a large number of diverse articles. It is still possible to get reasonable results with a smaller number of articles (fewer than 1,000 for example), and with a very homogenous dataset (for example just recent zika virus articles), but more general searches with large number of documents work best.

```R
tsneObj<-runTSNE(tidy_df,check_duplicates=FALSE)

#add t-SNE co-ordinates to df object
df<-inner_join(df,tsneObj$Y,by="PMID")

# plot the t-SNE results
ggplot(df,aes(x=tsneComp1,y=tsneComp2))+
  geom_point(alpha=0.2)+
  theme_bw()
```

Below is an initial plot of our documents! Each point is one article, darker points means many overlapping articles, and you'll notice that some articles already form visible groups (clusters), in the next step we'll suss out which clusters are reasonably well defined (according to the algorithm).

![](https://user-images.githubusercontent.com/5395870/36921077-d84ac778-1e17-11e8-8e57-a1d8f0c19808.png)

**Step 4: Perform an unsupervised clustering using hdbscan**

If you use hdbscan through Adjutant, the result will be the Adjuntant optimized hdbscan minPts parameter. You can also just use hdbscan directly on the result t-SNE dimensionally reduced data and sort out what the best hbdscan minPts parameter is for yourself. 

```R
#run HDBSCAN and select the optimal cluster parameters automaticallu
optClusters <- optimalParam(df)

#add the new cluster ID's the running dataset
df<-inner_join(df,optClusters,by="PMID") %>%
    mutate(tsneClusterStatus = ifelse(tsneCluster == 0, "not-clustered","clustered"))

# plot the HDBSCAN clusters (no names yet)
clusterNames <- df %>%
  dplyr::group_by(tsneCluster) %>%
  dplyr::summarise(medX = median(tsneComp1),
                   medY = median(tsneComp2)) %>%
  dplyr::filter(tsneCluster != 0)

ggplot(df,aes(x=tsneComp1,y=tsneComp2,group=tsneCluster))+
  geom_point(aes(colour = tsneClusterStatus),alpha=0.2)+
  geom_label(data=clusterNames,aes(x=medX,y=medY,label=tsneCluster),size=2,colour="red")+
  stat_ellipse(aes(alpha=tsneClusterStatus))+
  scale_colour_manual(values=c("black","blue"),name="cluster status")+
  scale_alpha_manual(values=c(1,0),name="cluster status")+ #remove the cluster for noise
  theme_bw()
```

Below is the same plot as before, but now we've got some clusters that Adjutant thinks are reasonable.

![](https://user-images.githubusercontent.com/5395870/36921074-d6be519a-1e17-11e8-98d5-a8b63e87623c.png)

**Step 5: Naming the clusters**

Adjutant has a function called **getTopTerms** that will automatically name a cluster according to its the top two most commonly occuring terms. If there are ties, it will return all 

```R
clustNames<-df %>%
          group_by(tsneCluster)%>%
          mutate(tsneClusterNames = getTopTerms(clustPMID = PMID,clustValue=tsneCluster,topNVal = 2,tidyCorpus=tidy_df)) %>%
          select(PMID,tsneClusterNames) %>%
          ungroup()
        
#update document corpus with cluster names
df<-inner_join(df,clustNames,by=c("PMID","tsneCluster"))

#re-plot the clusters

clusterNames <- df %>%
  dplyr::group_by(tsneClusterNames) %>%
  dplyr::summarise(medX = median(tsneComp1),
                   medY = median(tsneComp2)) %>%
  dplyr::filter(tsneClusterNames != "Noise")

ggplot(df,aes(x=tsneComp1,y=tsneComp2,group=tsneClusterNames))+
  geom_point(aes(colour = tsneClusterStatus),alpha=0.2)+
  stat_ellipse(aes(alpha=tsneClusterStatus))+
  geom_label(data=clusterNames,aes(x=medX,y=medY,label=tsneClusterNames),size=3,colour="red")+
  scale_colour_manual(values=c("black","blue"),name="cluster status")+
  scale_alpha_manual(values=c(1,0),name="cluster status")+ #remove the cluster for noise
  theme_bw()
```

Finally, here's the same plot again, but now with the names of the clusters. You'll notice that some clusters have very specific names (zika-viru; ebola viru) and some have more generic names (sequenc-isol-outbreak). My hypothesis for what is happening is a classic signal issue - some pathogens with only a few articles seem to cluster together into these more generic clusters, whereas pathogens with a lot of publications (and hence strong signal in the data) tend to form their own clusters.

![](https://user-images.githubusercontent.com/5395870/36920256-fe85e1d2-1e14-11e8-9c84-6bb7c8b13662.png)

**Step 6: Go forth and analyze some more!**

You can use df, tidy_df, or any other object produced by Adjutant in your own analysis!
