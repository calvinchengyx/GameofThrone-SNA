# 从社交网络分析看冰火的故事套路

## 目录
* 前言
* 逻辑思路及代码实现
* Gephi生成工具的说明
* 附录

## 前言

对《冰与火之歌》做社交网络分析算是我自己的一个兴趣吧。本身我是一个冰火剧迷，后来由于对电视剧“滥砍滥伐”的不满转战小说分析党。所以一直想以冰火为话题做点什么。社交网络分析是最近几年大热的交叉学科研究方法，恰好我也在上学的时候接触了一些皮毛，感觉用来分析“冰火”，是再合适不过了。</p>

__[社交网络分析（Social Network Analysis）](https://en.wikipedia.org/wiki/Social_network_analysis)__是从网络和图论角度入手，分析社会结构的一种方法，是__[计算社会学（Computational Sociology）](https://en.wikipedia.org/wiki/Computational_sociology)__的一种代表性研究方法。因为研究过程基于数据和量化分析，但是研究对象一般是社会问题，所以也是交叉学科的代表。我个人对这个领域很感兴趣。</p>

在我的原文中，我用社交网络分析了冰火的故事套路，其实还只是非常粗浅的水平，仅供娱乐。这个方法包罗万象，值得大家深入研究。感兴趣的朋友可以深入学习一下，推荐J Scott的_[Social Network Analysis](https://www.amazon.com/Social-Network-Analysis-John-Scott/dp/1446209040)_。

## 逻辑思路及代码实现
**注**：关系定义的思路来自玛卡莱斯特学院（Macalester College）数学系的Andrew.J.Beveridge教授，具体可以参考他的论文_[Throne Of Network](https://www.maa.org/sites/default/files/pdf/Mathhorizons/NetworkofThrones%20%281%29.pdf)_。

### Step 1 整理源文件
你需要先拿到GOT五本的英文原版文件，支持购买英文正版。但是如果你和我一样比较穷且只用于非营利性目的的学术研究的话，你可以直接Google之，网络上有很多免费版资源。我就不放在GitHub里了。</p>

然后你得到一个 _GOT_txt_ 的文本文件，里面包含了全部已出版五本书的内容。默认这个txt文件中五本书是按顺序排列的。</p> 

然后手动删去：__序言、前言、附录__。得到_GoTpure.txt_</p>

开始我们的表演。

### Step 2 确定主要人物，统一姓名
因为我们要分析的是人物关系，所以按图索骥的“图”就至关重要，这就是人物姓名。没办法，这一步要__手动操作__🤷‍♀️。嗯，简单来说，就是把人物的外号和人物的姓名统一起来。例如：Edd和Eddard都是奈德史塔克的称呼，我们统一为Eddard这样。</p>

然后我们要再手动添加一个“冰火专有名词”词典，提高分词的准确性。调用jiebaR直接对文本进行分词，统计一下词频。</p>

`got_seg <- segment(got,seg) `

去掉奇怪的符号

`got_name <- read.csv("got_name.csv") `
	`got <- readLines("got_text_en.txt")`
	`got_clean_1 <- gsub('[[:punct:]]+','', got)`

统计一下词频

`word_freq <- freq(got_seg_clean)`</p>

然后根据出现频率选择最重要的人物部分。至于如何选人物部分，请移步[冰火百科](http://gameofthrones.wikia.com/wiki/Game_of_Thrones_Wiki)。如果你不想做这一步，直接用现成的就行，我帮你选了106个好汉。已经存 got_nodes.csv 文档。他们就是我们进行社交网络分析的 Nodes，是我们的 node list。

### Step 3 确定人物关系Edges

有了 node，就要有 edges。我的方法是，如果两个人在14个单词的范围中出现了同时出现一次，那么他们就被记录为有一次关系。逻辑就是对 _gotpure.txt_ 进行以14个词为单位的遍历，找到所有106个人物中两两之间的关系次数。

具体如下：

找到所有人名字的位置

```R
for(i in 1:length(got_name_list)){
  for (j in 1:(length(got_clean_string)-13)){
    if (got_name_list[i] %in% got_clean_string[j:(j+13)]) 
      x <- rbind(x, c(got_name_list[i],j))
    print(c(i,j))
  }
}
```
然后，看那些名字出现在同一位置。（这一部分是用python做的，因为我暂时不会用R并行处理这个位置查询合并的操作，代码写得比较散。有R大神会的，求代码更新！）

```python
import csv
import pymongo


client = pymongo.MongoClient('localhost',27017)
got_project = client['got_project']
role = got_project['role']
role_spot = got_project['role_spot']
role_connection = got_project['role_connection']
role_count = got_project['role_count']


# num=1
# csv_reader = csv.reader(open('/Users/Weiwen/Desktop/x.csv', encoding='utf-8'))
# for row in csv_reader:
#     print(row)
#     # print(row[0].split(';'))
#     role_spot.insert_one({'id':row[0],'name':row[1],'spot':row[2]})
#     # num+=1

print(role.count())
for i in role.find():
    # print(i)
    role = i['name']
    lines=[]
    for j in role_spot.find({'name':role}):
        # print(j)
        lines.append(j['spot'])
        # print(role,lines)

    for line in lines:
        for k in role_spot.find({'spot':line}):
            count=1
            # print(k)
            if role != k['name']:
                role_b = k['name']
                print(role,role_b,line)
                role_connection.insert_one({'role_a':role,'role_b':role_b,'line':line})

for i in role.find():
    for j in role.find():
        if i['name'] != j['name']:
            count=1
            for k in role_connection.find({'role_a':i['name']}):
                if j['name'] == k['role_b']:
                    count+=1

            role_count.insert_one({'role_a':i['name'],'role_b':j['name'],'count':count})

```
然后我们合并结果，就得到了一个有65024条人物关系表，是我们的 edge list，存储为 got_edges.csv。

**got_nodes.csv**和**got_edges.csv**两个文件，是我们接下来进行SNA分析的关键核心。

### Step 4 社交网络分析
主要使用是R中的`igaph` Package.

``` R
library(igraph)
#导入上文中两个原始文档
got_edges <- read.csv("~/Desktop/Project-GameOfThrone/2.0/got_edges.csv",header = T)
got_nodes <- read.csv("~/Desktop/Project-GameOfThrone/2.0/got_nodes.csv",header = T)
# 对数据进行一步标准化处理（为了方便之后做图）
got_edges$weight_st <- (got_edges$weight-1)/1130 
got_nodes$freq_st <- (got_nodes$freq-1)/67
# 将数据转化为图数据
net_got <- graph.data.frame(d = got_edges,vertices = got_nodes, directed = F )
# net_got


###### 全局网络分析 ######
edge_density(net_got, loops=F)
transitivity(net_got, type="global")
diameter(net_got, directed=F,unconnected=F, weights = NA)
diam_got <- get_diameter(net_got, directed = F, weights = NA) # get the first found shortest path
diam_got

mean_distance(net_got, directed = F) # average path length
distances(net_got, weights = NA)
all_shortest_paths(net_got,from = "Jon", Tyrion = "Daenerys", mode = "all",)

###### 个人节点网络分析 ######

# degree centrality
deg_got <- degree(net_got, mode = "all")  # the same as freq
got_nodes$degree <- deg_got
# closeness centrality
clo_got <- closeness(net_got, mode = "all", weights = NA, normalized = T)
got_nodes$closeness <- clo_got # too small
# betweenness centrality
bet_got <- betweenness(net_got, directed=T, weights=NA)
got_nodes$betweenness <- bet_got
# pagerank
page_rank_got <- page_rank(net_got, algo = c("prpack", "arpack", "power"), 
                           vids = V(net_got),
                           directed = TRUE, damping = 0.85,
                           personalized = NULL, weights = NULL,
                           options = NULL)
got_nodes$page_rank <- page_rank_got[[1]]


###### 社区节点网络分析（聚类） ######
clique_got <- cliques(net_got) # list of cliques
sapply(cliques(net_got), length) # clique size
largest.cliques(net_got)

#community detection based on walktrap
wt_got <- cluster_walktrap(net_got) # mod = 0.38, groups = 5, better

got_nodes$community <- wt_got$membership
```
### Step 5 画图
```R
# # Generate colors based on categories
colrs <- adjustcolor(c("#8dd3c7","#ffffb3","#bebada","#fb8072","#80b1d3","#fdb462","#b3de69","#fccde5"), 
                     alpha = 0.8)
V(net_got)$color <- colrs[V(net_got)$community]

# Set node size based on freq

# V(net_got)$size <- 10
  
# # Setting nodes variables:
V(net_got)$frame.color <- adjustcolor("grey50",alpha = 0.8)
V(net_got)$label.color <- "black"
V(net_got)$label <- V(net_got)$name
V(net_got)$label.family = "Helvetica"
V(net_got)$label.cex = 0.5
V(net_got)$label.dist = 0
V(net_got)$label.font = 2
V(net_got)$size <- V(net_got)$freq_st*20

# Set edge width based on weight:
E(net_got)$width <- E(net_got)$weight_st*15
edge_start_got <- ends(net_got, es=E(net_got), names = F)[,1] # set1 edge color to it starts nodes
  E(net_got)$color <- V(net_got)$color[edge_start_got] # set edge color
  
l8 <- layout_with_lgl(net_got) 
# so far, this is the best layout, although still overlapping

plot(net_got, 
     edge.curved=.5,
     layout = l8
     )  
```

但是由于 nodes 数量过多，在R中画图，始终避免不了 overlapping 的情况。看一下成品图</p>
![igraph](https://github.com/calvinlovescode/GameofThrone-SNA/blob/master/igraph.pdf)

为了解决这个问题，可以把数据导入 Gephi 中进行分析，美化我们的社交网络关系图。</p>

把数据导出gephi的格式。

` got_gexf <- igraph.to.gexf(net_got)`

## Gephi

这一步我就不多介绍了，大家可以自行下载[gephi](https://gephi.org/)软件玩一下，这是一个非常容易上手的图形界面分析软件，人人都可以用。
来看一下成品图
![gephi](https://github.com/calvinlovescode/GameofThrone-SNA/blob/master/P1.png)

## Appendix

更多关于R igraph的操作，可以移步[这里](http://kateto.net/networks-r-igraph)，看完就是专家啦！





