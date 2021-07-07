## GFS1

课程讲义 http://nil.lcs.mit.edu/6.824/2020/notes/l-gfs.txt

## 为什么分布式存储难

```
Why is distributed storage hard?
  high performance -> shard data over many servers
  many servers -> constant faults
  fault tolerance -> replication
  replication -> potential inconsistencies
  better consistency -> low performance
```

高性能需要很多切片化的数据分布在各种服务上，而多服务又会造成潜在的数据丢失，为了保证数据安全我们则需要通过复制来备份我们的数据，复制则就会出现一致性问题。为了更好的一致性最后又会造成性能下降。

其他的内容基本和论文解读差不多，就不专门写了，这节课和GFS都是大致了解其思想即可。