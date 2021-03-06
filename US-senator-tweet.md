---
title: "U.S. Senators on Twitter"
author: "Hanyu Zhang"
output:
  html_document:
    toc: true
    self_contained: true
    keep_md: true
    code_folding: hide
---




```r
library(tidyverse)
library(igraph)
library(ggraph)
library(ggthemes)
library(readr)
```

## 1. Who follows whom?

### a) Network of Followers


```r
follow <- read_csv("senators_follow.csv")
senators <- read_csv("senators_twitter.csv")

#filter to see who follows who
fol <-  follow %>% 
  filter(following == "TRUE") %>% 
  select(source, target)

#the nodes
nodes <- senators %>% 
  select(twitter_handle, senator,party)

# create an igraph object  
g <- graph_from_data_frame(d = fol, vertices = nodes, directed = TRUE)

#calculate in and out degree
V(g)$in_degree <- degree(g, mode = "in")
V(g)$out_degree <- degree(g, mode = "out")
V(g)$size <- centralization.degree(g)$res

library(ggnetwork)
set.seed(2103)
dat <- ggnetwork(g, 
          layout=igraph::with_kk())

#check to see who has the highest in and out degree
# dat %>% 
#   arrange(desc(out_degree)) %>% 
#   distinct(name, .keep_all = TRUE)


library(ggrepel)
# filter for the senators with highest in and out degree
dat_top <- dat %>% filter(in_degree >= 91 | out_degree >= 76 ) %>% distinct(name, .keep_all = TRUE)
ggplot() +
  geom_edges(data=dat, 
             aes(x=x, y=y, xend=xend, yend=yend),size=0.03,color="grey50", curvature=0.1) +
  geom_nodes(data=dat,
             aes(x=x, y=y, xend=xend, yend=yend, size = size, color = party),alpha=2/3) + scale_color_manual(name = "Party", values = c("blue","green","red"), labels=c("D" = "Democrat", "I" = "Independent", "R"="Republican"))+
  scale_size_continuous(range = c(0.01, 5))+ geom_label_repel(data = dat_top, max.overlaps=100, aes(x=x, y=y, label=senator),size=2, color="#8856a7",segment.size=0.5,position=position_jitter(), force=100) + labs(size="Centrality") + theme_blank() 
```

<img src="US-senator-tweet_files/figure-html/unnamed-chunk-2-1.png" width="70%" height="70%" />

  From the network we could see that it's clearly Democrat and Republican are two clusters, which suggests that senators tend to follow and followed by their own party colleagues??? more. 
  Senator Johnson,Senator Romney,and Senator Lankford has the highest in degree, which means they have the highest number of followers and they are all republican; Senator Collins, Senator Murkowski, Senator	Manchin and Senator Grassley has the highest out degree, only one of them is democrat. So Republican senators seem to be more active on twitter. Those 7 senators also seem to be in the middle of the cluster between two parties, which may suggest they have a lot connections with senators from both of the party and they also seems to be large in size(high total number of following and follower).


## 2. What are they tweeting about?

### a) Most Common Topics over Time


```r
senator_tweets <- readRDS("senator_tweets.RDS")
# get the hashtags
hashtag <- senator_tweets  %>% 
  filter(is_retweet == "FALSE") %>% 
  select(hashtags) 
tag = unlist(hashtag$hashtags)
tagdf <- data.frame(tag = tag)
tagdf <- na.omit(tagdf)
library(tidytext)
# identify the most common one by counting their times of appearances
tags <- tagdf %>%
  group_by(tag) %>% 
  count(tag) %>% 
  arrange(desc(n)) 
library(wordcloud)
# select 100 most common hashtag
purple_orange <- brewer.pal(10,"RdBu")
wordcloud(tags$tag, tags$n,
         max.words = 100, colors = purple_orange)
```

<img src="US-senator-tweet_files/figure-html/unnamed-chunk-3-1.png" width="70%" height="70%" />

### b) Election Fraud 2020 - Dems vs. Reps

One topic that did receive substantial attention in the recent past the issue whether the [2020 presidential election involved fraud] and should be overturned. The resulting far-right and conservative campaign to Stop the Steal promoted the conspiracy theory that falsely posited that widespread electoral fraud occurred during the 2020 presidential election to deny incumbent President Donald Trump victory over former vice president Joe Biden.



```r
subhash <- senator_tweets  %>% 
  select(screen_name, hashtags, text)
library(tidyr)
subhash <- unnest(subhash, hashtags)
subhash$hashtags <- tolower(subhash$hashtags)
su <- subhash %>% 
  filter(hashtags %in% c("holdtheLine", "trumpwon",  "trump2020", "maga","standfirm","trumplost", "trumpwin", "backtheblue" , "tcot", "americafirst")) %>% 
  group_by(screen_name, hashtags) %>% 
  count(screen_name, hashtags) %>% 
  arrange(desc(n)) 

sen <- left_join(su,senators, by = c("screen_name" = "twitter_handle"))
sub <- sen %>% filter(n >=3 | party == "D")

t <- theme(axis.title = element_text(color = "grey40"),axis.text = element_text(color = "grey40"),plot.caption = element_text(color = "grey60", size = 5),plot.title = element_text(hjust = 0.5))

ggplot(sub) + geom_col(aes(x = 
n, y = reorder(screen_name, n), fill = party), width = 0.7) + scale_fill_manual(values = c("darkblue", "red"))+ scale_x_continuous(breaks = seq(0,60,5)) + labs(title = "Senators with Their Hashtags", x = "Number of Hashtags used", y = "")+
  geom_label(data = filter(sub,screen_name != "SteveDaines"),aes(x = n + 6, y =screen_name, label = hashtags)) + geom_label_repel(data = filter(sub,screen_name == "SteveDaines"), aes(x = n + 6, y =screen_name, label = hashtags))+theme_minimal() + t
```

<img src="US-senator-tweet_files/figure-html/unnamed-chunk-4-1.png" width="70%" height="70%" />

I filtered for hashtags "holdtheLine", "trumpwon",  "trump2020", "maga","standfirm","trumplost", "trumpwin", "backtheblue" , "tcot", "americafirst", which all support the stop the steal the movement. I select top 10 senators in the Republican party who talks most on those hashtags and all the democrat senators. It looks like senators from the Republican party talk a lot on this issue, especially senator Mike Lee, democrat party seems to talk about this less.

## 3. Are you talking to me?

Often tweets are simply public statements without addressing a specific audience. However, it is possible to interact with a specific person by adding them as a friend, becoming their follower, re-tweeting their messages, and/or mentioning them in a tweet using the @ symbol.

### a) Identifying Re-Tweets


```r
# filter for retweet 
retweet <- senator_tweets %>% 
  filter(is_retweet == "TRUE") %>% 
  filter(screen_name %in% nodes$twitter_handle & retweet_screen_name %in% nodes$twitter_handle) %>% #filter for senators 
  select(screen_name, retweet_screen_name) %>% 
  rename(retweeter = screen_name, origin = retweet_screen_name)
#another possible approach
# retweetp <- left_join(retweet, nodes, by = c("retweeter" = "twitter_handle")) # match party information for senators(retweeters)
# retweetp <- retweetp %>% 
#   rename(party_of_ret = party)
# retparty <- left_join(retweetp, nodes, by = c("origin" = "twitter_handle")) # match party information for senators(origins)
# retparty <- dplyr :: rename(retparty, originparty = party)
# 
# retparty %>% 
#   group_by(retweeter, originparty) %>% 
#   count(originparty)
# 
# retparty %>% 
#   group_by(origin, party_of_ret) %>% 
#   count(party_of_ret)

# count the number of retweeted to later assign to the edge's weight
re <- retweet %>% 
  filter(retweeter != origin) %>% 
  group_by(retweeter, origin) %>% 
  count(origin)

gr <- graph_from_data_frame(d = re, vertices = nodes, directed = TRUE)


set.seed(2103)
dat <- ggnetwork(gr, 
          layout=igraph::with_kk())


ggplot() +
  geom_edges(data=dat, 
             aes(x=x, y=y, xend=xend, yend=yend, size=n),color="grey50", curvature=0.1) +
  geom_nodes(data=dat,
             aes(x=x, y=y, xend=xend, yend=yend, color = party),alpha=2/3) + scale_color_manual(name = "Party", values = c("blue","green","red"),labels=c("D" = "Democrat", "I" = "Independent", "R"="Republican"))+
  scale_size_continuous(name = "Number of Retweets",range = c(0.05, 1))+ theme_blank() 
```

<img src="US-senator-tweet_files/figure-html/unnamed-chunk-5-1.png" width="70%" height="70%" />

We could see that the clustering is still very obvious, so I think most of the senators seem to largely re-tweet their own party colleagues??? messages. One of the independent senators is in the middle of the two clusters, so I guess he or she re-tweeted on both sides of the aisle.

### b) Identifying Mentions


```r
mention <- senator_tweets %>% 
  filter(is_retweet == "FALSE") %>% 
  filter(screen_name %in% nodes$twitter_handle & mentions_screen_name %in% nodes$twitter_handle) #filter for senators 
m <- mention %>% 
  mutate(mname =unlist(mention$mentions_screen_name) ) %>% 
  select(screen_name, mname)

gra <- graph_from_data_frame(d = m, vertices = nodes, directed = FALSE)

# calculate the number of mentions as weight of the edges
E(gra)$weight <- 1
g <- igraph::simplify(gra, edge.attr.comb="sum")
# thinner the width of the edge so the graph wont look too messy
E(g)$weight <- E(g)$weight/20
# the centrality of the nodes
V(g)$centrality <- centralization.degree(g)$res

set.seed(2103)
dat <- ggnetwork(g, 
          layout=igraph::with_kk())


ggplot() +
  geom_edges(data=dat, 
             aes(x=x, y=y, xend=xend, yend=yend, size = weight), color="grey50", curvature=0.1) +
  geom_nodes(data=dat,
             aes(x=x, y=y, xend=xend, yend=yend, color = party, size = centrality),alpha=2/3) + scale_color_manual(name = "Party", values = c("blue","green","red"),labels=c("D" = "Democrat", "I" = "Independent", "R"="Republican"))+scale_size_continuous(name = "Centrality", range = c(0.01, 2))+
   theme_blank() #+ theme(legend.position = "top")
```

<img src="US-senator-tweet_files/figure-html/unnamed-chunk-6-1.png" width="70%" height="70%" />

In contrast to following and followers network and retweet network, this mention network shows that many senators actually mention their colleagues from other parties frequently.

