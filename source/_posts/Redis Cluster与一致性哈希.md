---
title: Redis Cluster与一致性哈希
date: 2017-05-09 22:51:08
categories: [一致性哈希]
tags: [Redis Cluster,一致性哈希]
---
<Excerpt in index | 首页摘要>
Redis Cluster与一致性哈希<!-- more -->
<The rest of contents | 余下全文>

# 写在前面
当初由果推因，竟暗合一致性哈希的原理。感觉很有意思~
详见[分布式令牌桶设计实现（流控）](http://blog.1x1.space/2016/12/28/分布式令牌桶设计实现（流控）/)文章下：猜测一下Redis Cluster执行涉及多个key的原子操作的设计原理。
后来某一天突然看到集群伸缩与一致性哈希的文章，不谋而合。
