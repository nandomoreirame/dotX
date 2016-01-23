---
layout: post
title:  "Starting-up within Microsoft: Maguro"
date:   2015-10-6 15:20:22 +0700
author: Henry Tan
editor: Argi Karunia
categories: blog
profile: >
  Henry Tan Setiawan focuses on research and development of Deep Learning infrastructure and services (a.k.a Project Adam). He was leading the data classification tech team at BlueKai, the leader in AdTech Big Data company prior getting acquired by Oracle in 2013. He holds a PhD in Computer Science from University of Technology, Sydney. He joined Microsoft, Redmond, USA in 2006 and helped developed the Bing BigIndex search backend infrastructures and services (a.k.a Maguro) and contributed to the R&D of other large scale distributed services at Microsoft including, but not limited to, Azure Machine Learning, Azure Cloud Storage, and Messenger Server backend services.
source:
  title: Tech a break #5
  date: Sep 14, 2015
  author: Henry Tan
refs:
  - https://en.wikipedia.org/wiki/Power_law
---
<iframe src="https://docs.google.com/presentation/d/14x8kWwWA7_bdQBaE0IfBv9OpbiYfMAAkQdVXDcEfurg/embed?start=false&amp;loop=false&amp;delayms=3000" width="640" height="389" frameborder="0" allowfullscreen="allowfullscreen"></iframe><br>

#### You might be thinking, what does “*Starting-up within Microsoft*” mean?

Though it is not very common, at [Microsoft][microsoft] we occasionally incubate products or technologies from a ground-up. Typically at any group at Microsoft we do have *short-term*, *mid-term* and *long-term* focus. Incubation of new technologies can happen for different focuses and needs and mostly they are considered as *long-term bets*.

### Maguro ~ The Big Tuna

![Big Tuna][img_tuna]
<span class="img-caption">Source: [nationalgeographic.com][natgeo]</span>

Maguro means tuna in Japanese language. But, why we incubated Maguro? We were not [Google][google], our index was only a fraction of what Google had in 2010, and we did not have as big of a budget as Google.

We realized that one way to stay alive is to increase our index size, so our goal was to scale our index size from low to high *tens of Billion Documents* with technology that can support up to *1 Trillion documents*.

Using the current architecture, then, was not financially feasible as it will be too expensive, besides it will hit a perf bottleneck at that scale. Hence we incubated Maguro, a system for efficiently searching very large collections of text content of up to 1 trillion documents at low cost.

### Search Engine

![Search engine (single box)][img_maguro_1]
<span class="img-caption">Search engine (single box)</span>

Imagine a single server Search Engine, if each document (forward index unit) and reverse index can be compressed to about says 1kb – with 32GBytes of memory roughly you can hold 32Millions of documents. There are trillions of web documents on the Internet, mathematically roughly 35,000 servers for one copy of index, for about 500-1000 QPS, for 20-30K QPS, means for the capacity you will need 30-60 x 35,000 = 1M – 2M servers. $1-4Bn investment!

With Maguro we aim to at least build the system with the investment requirement of 1/10th of conventional architecture. We aim to put up to 100M or more docs per node ~ 4Bn docs per segment ~ and let’s say hypothetically the segment size is around 40 nodes; it requires 250 segments for 1Trillion web documents. With 10K machines per capacity unit (serving row) and assume the traffics requires us to build 10 capacity rows; it requires roughly 100K machines roughly 1/10th of the capacity required without Maguro technology.

### Head vs Tail

![Head vs long tail keywords][img_maguro_2]
<span class="img-caption">Source: [neilpatel.com][neilpatel]</span>

We make money from head contents, but we retain customers from tail contents. If customers search for something that is not in index, since switching cost is virtually zero, they can jump right away. The irony is that the cost to hold up tail contents is way too high.

The web content distribution follows the Power law distribution, where a relative change in one quantity results in a proportional relative change in the other quantity, independent of the initial size of those quantities: one quantity varies as a power of another. [[1]](#ref-1)

When resource is limited, it means you need strategy. In statistics, a long tail of some distributions is the portion of the distribution having a large number of occurrences far from the “head” or central part of the distribution. This insight gives you some hints the needs for a different treatment of the web contents.

With such insight that the web content is mostly skewed towards the tail content, we can devise a strategy *where the existing system can be used to serve the head content and focus on the solution targeting the long tail content*. Hence we incubated Maguro for the long tail of content with less dynamics and less metadata, but very good cost efficiency.

### Doc Shard Architecture

![Search engine architecture (doc shard)][img_maguro_3]
<span class="img-caption">Search engine architecture (doc shard)</span>

Typical search engine architecture is a grid like architecture.

![Doc shard (complete answer)][img_maguro_4]
<span class="img-caption">Doc shard (complete answer)</span>

To obtain a complete answer, you send query to each of the node of each column and load-balance them to form a complete row.

![Doc shard (full row)][img_maguro_5]
<span class="img-caption">Doc shard (full row)</span>

From a full row you will get the complete results.

![Grid architecture][img_maguro_6]
<span class="img-caption">Grid architecture</span>

With grid architecture we can increase our index size simply by expanding the number of columns and increase our capacity through adding more rows. The issue of this architecture is when one row is down for maintenance, the rest of the rows are serving the traffics. You don’t want that to happen, you want to have enough rows to keep up with the loads.

The capacity of each node with conventional architecture is fairly limited ~ 32 Millions web documents. How do we increase the capacity of a single node?

### Term Shard Architecture

![Nodes][img_maguro_7]
<span class="img-caption">Nodes</span>

![Segment][img_maguro_8]
<span class="img-caption">Segment</span>

The approach to increase the capacity of a single node is to replace the single node with a cluster of nodes, we call this node now a “*segment*“. Inside of the segment each node will work in collaboration to answer a query. This should solve the capacity problem, but new challenges arises:    

1. *Query execution becomes distributed*, some terms might be on one node and some other terms on different nodes. So how do you do intersects or union in distributed fashion?
2. *Transferring data over the network becomes bottleneck!* Hence the needs for at least 10Gbps network. Some of the other techniques we do to help solve the data transfer over the network is by combining multiple words into single word and doing the processing offline. So imagine the query “*william tanuwijaya*“, instead of thinking of the query as two words, let’s think of it as a single word and it will result in shorter list and max 250ms query execution latency.

![Cluster of nodes (segment)][img_maguro_9]
<span class="img-caption">Cluster of nodes (segment)</span>

To increase the capacity per node we also enable technology to serve query from memory, SSD, or disk.

![Segments][img_maguro_10]
<span class="img-caption">Segments</span>

Now each segment represent a super node that can hold up 4Bn docs, and to serve 1 Trillion documents we will use the same grid structure as before.

### Maguro architecture

![Maguro architecture][img_maguro_11]
<span class="img-caption">Maguro architecture</span>

Instead of treating each node as a single server, each node is now a segment, with a collection of segments to be called as a cluster of nodes, a serving node with term-shard architecture.

*“BSDBNS: Build Simple Design but NOT Simpler”*

**Further reading:**    
Risvik, K., Chilimbi, T., Tan, H., Kalyanaraman, K., & Anderson, C. (n.d.). Maguro, a system for indexing and searching over very large text collections. *Proceedings of the Sixth ACM International Conference on Web Search and Data Mining – WSDM ’13*. Retrieved October 7, 2015, from [http://dl.acm.org/citation.cfm?id=2433486][paper]

[microsoft]: http://www.microsoft.com/
[google]: https://www.google.com/
[paper]: http://dl.acm.org/citation.cfm?id=2433486
[power_law]: https://en.wikipedia.org/wiki/Power_law
[natgeo]: http://channel.nationalgeographic.com/wicked-tuna/articles/bluefin-tuna-101/
[neilpatel]: http://neilpatel.com/2015/03/26/how-to-generate-20000-monthly-search-visitors-through-long-tail-traffic/
[img_tuna]: {{ "/assets/img/upload/tuna.jpg" | prepend: site.baseurl }}
[img_maguro_1]: {{ "/assets/img/upload/maguro-1.png" | prepend: site.baseurl }}
[img_maguro_2]: {{ "/assets/img/upload/maguro-2.jpg" | prepend: site.baseurl }}
[img_maguro_3]: {{ "/assets/img/upload/maguro-3.png" | prepend: site.baseurl }}
[img_maguro_4]: {{ "/assets/img/upload/maguro-4.png" | prepend: site.baseurl }}
[img_maguro_5]: {{ "/assets/img/upload/maguro-5.png" | prepend: site.baseurl }}
[img_maguro_6]: {{ "/assets/img/upload/maguro-6.png" | prepend: site.baseurl }}
[img_maguro_7]: {{ "/assets/img/upload/maguro-7.png" | prepend: site.baseurl }}
[img_maguro_8]: {{ "/assets/img/upload/maguro-8.png" | prepend: site.baseurl }}
[img_maguro_9]: {{ "/assets/img/upload/maguro-9.png" | prepend: site.baseurl }}
[img_maguro_10]: {{ "/assets/img/upload/maguro-10.png" | prepend: site.baseurl }}
[img_maguro_11]: {{ "/assets/img/upload/maguro-11.png" | prepend: site.baseurl }}
