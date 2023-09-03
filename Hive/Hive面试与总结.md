> 鸣谢！！
>
> 参考文档：[Hive从入门到精通，HQL硬核整理四万字，全面总结，附详细解析，赶紧收藏吧！！](https://blog.csdn.net/qq_43791724/article/details/120222851?spm=1001.2014.3001.5501)

# 一、Hive基础



## Hive的数据存储格式

- Hive的数据存储基于Hadoop HDFS。
- Hive没有专门的数据文件格式，常见的有以下几种：TEXTFILE、SEQUENCEFILE、AVRO、RCFILE、ORCFILE、PARQUET。



### **ORCFile**

Hive从0.11版本开始提供了ORC的文件格式，**ORC文件**不仅仅是**一种列式文件存储格式**，最重要的是**有着很高的压缩比**，并且**对于MapReduce来说是可切分（Split）的**。因此，**在Hive中使用ORC作为表的文件存储格式，不仅可以很大程度的节省HDFS存储资源，而且对数据的查询和处理性能有着非常大的提升**，因为ORC较其他文件格式压缩比高，查询任务的输入数据量减少，使用的Task也就减少了。**ORC能很大程度的节省存储和计算资源，但它在读写时候需要消耗额外的CPU资源来压缩和解压缩**，当然这部分的CPU消耗是非常少的。

- **Parque**