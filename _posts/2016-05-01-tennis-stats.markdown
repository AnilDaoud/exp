---
layout: post
title:  "Tennis stats"
date:   2016-05-01 10:38:27
tags: r,tennis,statistics
categories: experiments
---

I came across [Tennis-data](http://www.tennis-data.co.uk/) while random-browsing and I thought it would make the perfect data set to experiment with [R](https://www.r-project.org/).

##Step 1: data

Download all spreadsheets. Thankfully they all have the same basic structure, one line per match, same columns etc. Most of them have one sheet named the same as the file.

{% highlight R %}
library(gdata)
library(data.table)
years = list.files(pattern='[.]xls')

# this is supposedly the least painful way to do this but WRank got corrupted when i did this. this did not happen with data.table
# library(plyr)
# df=ldply(years,read.xls)

dflist = lapply(years,read.xls)
df = rbindlist(dflist, use.names = TRUE, fill = TRUE)
df = as.data.frame(df)

saveRDS(df,file="tennis.data")
{% endhighlight %}

Run this with _RScript_ or simply in a _R_ prompt and it will go through all the excel files and generate a nice R data frame saved in a file named tennis.data.

If you import this in R

{% highlight R %}
df <- readRDS(file="tennis.data")
head(df)
{% endhighlight %}

It will look something like this:

| Location | Tournament         | Date       | Series        | Surface | Round     | Winner     | Loser       | WRank | WRank | W1 | L1 | W2 | L2 | W3 | L3 |
|----------|--------------------|------------|---------------|---------|-----------|------------|-------------|-------|-------|----|----|----|----|----|----|
| Adelaide | AAPT Championships | 2001-01-01 | International | Hard    | 1st Round | Clement A. | Gaudenzi A. | 18    | 101   | 6  | 7  | 6  | 0  | 6  | 3  |
| ...      |                    |            |               |         |           |            |             |       |       |    |    |    |    |    |    |
| ...      |                    |            |               |         |           |            |             |       |       |    |    |    |    |    |    |

##Step 2: play

Right so now we can load the data in R from the data file, instead of looping through the spreadsheets every time.

{% highlight R %}
sink(file="/dev/null")
suppressPackageStartupMessages(library(reshape2))
suppressPackageStartupMessages(library(gdata))
df <- readRDS(file="data/tennis.data")
{% endhighlight %}

Now that that's out of the way, this will reformat the few columns we want to pla with here:

{% highlight R %}
df$LRank <- suppressWarnings(as.integer(as.character(df$LRank)))
df$WRank <- suppressWarnings(as.integer(as.character(df$WRank)))
df$Date <- as.POSIXct(df$Date, format='%Y-%m-%d')
df$Year <- format(df$Date, format='%Y')
{% endhighlight %}

Right so now we can finally play with the data!
{% highlight R %}
# If the first set win means match win, set to 1, otherwise 0
df$Win1 <- ifelse(df["W1"] >= df["L1"],1,0)
# if the winner outranks the loser set to 1
df$Win2 <- ifelse(df["WRank"] <= df["LRank"],1,0)
# if both previous conditions are true, set to 1
df$Win3 <- ifelse(df["Win1"]==1 & df["Win2"]==1,1,0)
{% endhighlight %}

Now we just need to write a small function to compute stats, and then one to display them. You'll see the name of the functions is very creative:
{% highlight R %}
computeStats <- function(data, column) {
    data.subseted = data[,colnames(data) %in% c("Year",column)]
    result = dcast(as.data.frame(table(data.subseted)), as.formula(paste("Year",column, sep ="~")), value.var="Freq")
    result$total = result$"0"+result$"1"
    result <- data.frame(Losses = result$"0",Wins=result$"1",Total=result$total, row.names=result$Year, stringsAsFactors=F)
    result["Total",] = colSums(result)
    result$Pct <- result$Wins/(result$Total)*100
    return(result)
}

displayStats <- function(table) {
    message("======================================================")
    write.fwf(format(table, big.mark=",",zero.print=FALSE,trim=TRUE, digits=4),rownames=TRUE)
    message("======================================================")
    message(paste("It took",format(proc.time()["elapsed"]*1000,digits=5,big.mark=","), "milli-seconds to run this"))
}
{% endhighlight %}

Right, so we can display everything now:

{% highlight R %}
message("First set win means match win?")
displayStats(computeStats(df,"Win1"))

message ("  ")
message("Winner outranks loser?")
displayStats(computeStats(df,"Win2"))

message ("  ")
message("First set win means match win, and winner outranks loser?")
displayStats(computeStats(df,"Win3"))
{% endhighlight %}

##Step 3: Now what

It looks like first set win means match win in ~80% of cases!

	First set win means match win?
	======================================================
	Losses Wins Total Pct
	2001  586   2,465  3,051  80.79
	2002  583   2,216  2,799  79.17
	2003  532   2,268  2,800  81.00
	2004  525   2,345  2,870  81.71
	2005  519   2,380  2,899  82.10
	2006  564   2,332  2,896  80.52
	2007  530   2,284  2,814  81.17
	2008  504   2,166  2,670  81.12
	2009  551   2,164  2,715  79.71
	2010  480   2,183  2,663  81.98
	2011  491   2,170  2,661  81.55
	2012  483   2,183  2,666  81.88
	2013  492   2,086  2,578  80.92
	2014  501   2,034  2,535  80.24
	2015  512   2,109  2,621  80.47
	2016  168   735    903    81.40
	Total 8,021 34,120 42,141 80.97
	======================================================
	It took 1,333 milli-seconds to run this

Is the ATP rank a better predictor? Looks like not. This looks like more of a 2 out of 3 type of stat.

	Winner outranks loser?
	======================================================
	Losses Wins Total Pct
	2001  1,140  1,906  3,046  62.57
	2002  1,026  1,774  2,800  63.36
	2003  1,007  1,805  2,812  64.19
	2004  1,042  1,829  2,871  63.71
	2005  991    1,912  2,903  65.86
	2006  1,010  1,889  2,899  65.16
	2007  982    1,835  2,817  65.14
	2008  895    1,785  2,680  66.60
	2009  885    1,840  2,725  67.52
	2010  899    1,775  2,674  66.38
	2011  874    1,800  2,674  67.31
	2012  862    1,814  2,676  67.79
	2013  895    1,692  2,587  65.40
	2014  823    1,736  2,559  67.84
	2015  842    1,779  2,621  67.87
	2016  286    620    906    68.43
	Total 14,459 27,791 42,250 65.78
	======================================================
	It took 1,397 milli-seconds to run this

##Step 4: Well that was underwhelming

Who cares.

Il n'y a de vraiment beau que ce qui ne peut servir à rien ; tout ce qui est utile est laid, car c'est l'expression de quelque besoin, et ceux de l'homme sont ignobles et dégoûtants, comme sa pauvre et infirme nature. - L'endroit le plus utile d'une maison, ce sont les latrines.
 -- Théophile Gauthier, Mademoiselle de Maupin, préface.

