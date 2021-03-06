# TUGAS
Tugas visualisasi (Perpustakaan Seaborn)
---
title: "**Clustering wines with k-means**"
author: "Xavier Vivancos García"
date: '`r Sys.Date()`'
output: 
  html_document:
    number_sections: yes
    toc: yes
    theme: cosmo
    highlight: tango
---

# **Introduction**

k-means is an unsupervised machine learning algorithm used to find groups of observations (clusters) that share similar characteristics. 
What is the meaning of unsupervised learning? It means that the observations given in the data set are unlabeled, there is no outcome to be predicted. 
We are going to use a [Wine data set](https://archive.ics.uci.edu/ml/datasets/wine) to cluster different types of wines. 
This data set contains the results of a chemical analysis of wines grown in a specific area of Italy. 

# **Loading data** {.tabset .tabset-fade .tabset-pills}

First we need to load some libraries and read the data set.

```{r message=FALSE, warning=FALSE}
# Load libraries
library(tidyverse)
library(corrplot)
library(gridExtra)
library(GGally)
library(knitr)

# Read the stats
wines <- read.csv("../input/Wine.csv")
```

We don't need the `Customer_Segment` column. As we have said before, k-means is an unsupervised machine learning algorithm and works with unlabeled data. 

```{r}
# Remove the Type column
wines <- wines[, -14]
```

Let's get an idea of what we're working with.

## First rows 
```{r}
# First rows 
kable(head(wines))
```

## Last rows 
```{r}
# Last rows 
kable(tail(wines))
```

## Summary 
```{r}
# Summary
kable(summary(wines))
```

## Structure
```{r}
# Structure 
str(wines)
```

# **Data analysis** 

First we have to explore and visualize the data.

```{r message=FALSE, fig.align='center'}
# Histogram for each Attribute
wines %>%
  gather(Attributes, value, 1:13) %>%
  ggplot(aes(x=value, fill=Attributes)) +
  geom_histogram(colour="black", show.legend=FALSE) +
  facet_wrap(~Attributes, scales="free_x") +
  labs(x="Values", y="Frequency",
       title="Wines Attributes - Histograms") +
  theme_bw()
```

```{r message=FALSE, fig.align='center'}
# Density plot for each Attribute
wines %>%
  gather(Attributes, value, 1:13) %>%
  ggplot(aes(x=value, fill=Attributes)) +
  geom_density(colour="black", alpha=0.5, show.legend=FALSE) +
  facet_wrap(~Attributes, scales="free_x") +
  labs(x="Values", y="Density",
       title="Wines Attributes - Density plots") +
  theme_bw()
```

```{r message=FALSE, warning=FALSE, fig.align='center'}
# Boxplot for each Attribute  
wines %>%
  gather(Attributes, values, c(1:4, 6:12)) %>%
  ggplot(aes(x=reorder(Attributes, values, FUN=median), y=values, fill=Attributes)) +
  geom_boxplot(show.legend=FALSE) +
  labs(title="Wines Attributes - Boxplots") +
  theme_bw() +
  theme(axis.title.y=element_blank(),
        axis.title.x=element_blank()) +
  ylim(0, 35) +
  coord_flip()
```

We haven't included magnesium and proline, since their values are very high and worsen the visualization.

What is the relationship between the different attributes? We can use the `corrplot()` function to create a graphical display of a correlation matrix. 

```{r fig.align='center'}
# Correlation matrix 
corrplot(cor(wines), type="upper", method="ellipse", tl.cex=0.9)
```

There is a strong linear correlation between `Total_Phenols` and `Flavanoids`. We can model the relationship between these two variables by fitting a linear equation.

```{r fig.align='center'}
# Relationship between Phenols and Flavanoids
ggplot(wines, aes(x=Total_Phenols, y=Flavanoids)) +
  geom_point() +
  geom_smooth(method="lm", se=FALSE) +
  labs(title="Wines Attributes",
       subtitle="Relationship between Phenols and Flavanoids") +
  theme_bw()
```

Now that we have done a exploratory data analysis, we can prepare the data in order to execute the k-means algorithm. 

# **Data preparation** 

We have to normalize the variables to express them in the same range of values. In other words, normalization means adjusting values measured on different scales to a common scale.

```{r fig.align='center'}
# Normalization
winesNorm <- as.data.frame(scale(wines))

# Original data
p1 <- ggplot(wines, aes(x=Alcohol, y=Malic_Acid)) +
  geom_point() +
  labs(title="Original data") +
  theme_bw()

# Normalized data 
p2 <- ggplot(winesNorm, aes(x=Alcohol, y=Malic_Acid)) +
  geom_point() +
  labs(title="Normalized data") +
  theme_bw()

# Subplot
grid.arrange(p1, p2, ncol=2)
```

The points in the normalized data are the same as the original one. The only thing that changes is the scale of the axis.

# **k-means execution** 

In this section we are going to execute the k-means algorithm and analyze the main components that the function returns. 

```{r}
# Execution of k-means with k=2
set.seed(1234)
wines_k2 <- kmeans(winesNorm, centers=2)
```

The `kmeans()` function returns an object of class "`kmeans`" with information about the partition: 

* `cluster`. A vector of integers indicating the cluster to which each point is allocated.

* `centers`. A matrix of cluster centers.

* `size`. The number of points in each cluster.

```{r}
# Cluster to which each point is allocated
wines_k2$cluster

# Cluster centers
wines_k2$centers

# Cluster size
wines_k2$size
```

Additionally, the `kmeans()` function returns some ratios that let us know how compact is a cluster and how different are several clusters among themselves. 

* `betweenss`. The between-cluster sum of squares. In an optimal segmentation, one expects this ratio to be as higher as possible, since we would like to have heterogeneous clusters.

* `withinss`. Vector of within-cluster sum of squares, one component per cluster. In an optimal segmentation, one expects this ratio to be as lower as possible for each cluster, 
* since we would like to have homogeneity within the clusters.

* `tot.withinss`. Total within-cluster sum of squares. 

* `totss`. The total sum of squares.

```{r}
# Between-cluster sum of squares
wines_k2$betweenss

# Within-cluster sum of squares
wines_k2$withinss

# Total within-cluster sum of squares 
wines_k2$tot.withinss

# Total sum of squares
wines_k2$totss
```

# **How many clusters?**

To study graphically which value of `k` gives us the best partition, we can plot `betweenss` and `tot.withinss` vs Choice of `k`. 

```{r fig.align='center'}
bss <- numeric()
wss <- numeric()

# Run the algorithm for different values of k 
set.seed(1234)

for(i in 1:10){

  # For each k, calculate betweenss and tot.withinss
  bss[i] <- kmeans(winesNorm, centers=i)$betweenss
  wss[i] <- kmeans(winesNorm, centers=i)$tot.withinss

}

# Between-cluster sum of squares vs Choice of k
p3 <- qplot(1:10, bss, geom=c("point", "line"), 
            xlab="Number of clusters", ylab="Between-cluster sum of squares") +
  scale_x_continuous(breaks=seq(0, 10, 1)) +
  theme_bw()

# Total within-cluster sum of squares vs Choice of k
p4 <- qplot(1:10, wss, geom=c("point", "line"),
            xlab="Number of clusters", ylab="Total within-cluster sum of squares") +
  scale_x_continuous(breaks=seq(0, 10, 1)) +
  theme_bw()

# Subplot
grid.arrange(p3, p4, ncol=2)
```

Which is the optimal value for `k`? One should choose a number of clusters so that adding another cluster doesn't give much better partition of the data. At some point the gain will drop, giving an angle in the graph (elbow criterion). 
The number of clusters is chosen at this point. In our case, it is clear that 3 is the appropriate value for `k`. 

# **Results**

```{r fig.align='center'}
# Execution of k-means with k=3
set.seed(1234)

wines_k3 <- kmeans(winesNorm, centers=3)

# Mean values of each cluster
aggregate(wines, by=list(wines_k3$cluster), mean)

# Clustering 
ggpairs(cbind(wines, Cluster=as.factor(wines_k3$cluster)),
        columns=1:6, aes(colour=Cluster, alpha=0.5),
        lower=list(continuous="points"),
        upper=list(continuous="blank"),
        axisLabels="none", switch="both") +
        theme_bw()
```

# **Summary** 

In this entry we have learned about the k-means algorithm, including the data normalization before we execute it, the choice of the optimal number of clusters (elbow criterion) and the visualization of the clustering.

It has been a pleasure to make this post, I have learned a lot! Thank you for reading and if you like it, please upvote it.

# **Citations for used packages**

Dua, D. and Karra Taniskidou, E. (2017). UCI Machine Learning Repository [http://archive.ics.uci.edu/ml]. Irvine, CA: University of California, School of Information and Computer Science.

Hadley Wickham (2017). tidyverse: Easily Install and Load the 'Tidyverse'. R package version 1.2.1. https://CRAN.R-project.org/package=tidyverse

Taiyun Wei and Viliam Simko (2017). R package "corrplot": Visualization of a Correlation Matrix (Version 0.84). Available from https://github.com/taiyun/corrplot

Baptiste Auguie (2017). gridExtra: Miscellaneous Functions for "Grid" Graphics. R package version 2.3. https://CRAN.R-project.org/package=gridExtra

Barret Schloerke, Jason Crowley, Di Cook, Francois Briatte, Moritz Marbach, Edwin Thoen, Amos Elberg and Joseph Larmarange (2017). GGally: Extension to 'ggplot2'. R package version 1.3.2. https://CRAN.R-project.org/package=GGally

Yihui Xie (2018). knitr: A General-Purpose Package for Dynamic Report Generation in R. R package version 1.20.

Yihui Xie (2015) Dynamic Documents with R and knitr. 2nd edition. Chapman and Hall/CRC. ISBN 978-1498716963

Yihui Xie (2014) knitr: A Comprehensive Tool for Reproducible Research in R. In Victoria Stodden, Friedrich Leisch and Roger D. Peng, editors, Implementing Reproducible Computational Research. Chapman and Hall/CRC. ISBN 978-1466561595
