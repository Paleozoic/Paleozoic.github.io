---
title: Spark RDD API
date: 2017-04-23 15:29:30
tags: [Spark]
---
<Excerpt in index | 首页摘要>
介绍Spark RDD API的含义与使用。<!-- more -->
<The rest of contents | 余下全文>
[Spark官方网站的描述](http://spark.apache.org/docs/latest/programming-guide.html#transformations)
# 常用接口
## Spark主要类
- SparkContext：是Spark对外接口，负责向调用该类的scala应用提供Spark的各种功能，如连接Spark集群、创建RDD等。
- SparkConf：Spark应用配置类，如配置应用名称，执行模式，executor内存等。
- RDD（Resilient Distributed Dataset）：用于Spark应用程序中定义RDD的类，该类提供数据集的操作方法，如map，filter。
- PairRDDFunctions：为key-value对的RDD数据提供运算操作，如groupByKey。
- Broadcast：广播变量类，广播变量允许保留一个只读的变量，缓存在每一台机器上，而非每个任务保存一份拷贝。
- StorageLevel：数据存储级别，有内存（MEMORY_ONLY），磁盘（DISK_ONLY），内存+磁盘（MEMORY_AND_DISK）等。
**RDD支持2中类型的操作：transformation和action。**
**transformation实质是一个逻辑的action，记录了RDD的演变过程。transformation采用的是懒策略。只有action被提交时才会触发transformation的计算动作。**
>Each RDD has 2 sets of parallel operations: transformation and action.
(1)Transformation:Return a MappedRDD[U] by applying function f to each element
(2)Action:return T by reducing the elements using specified commutative and associative binary operator

>Operations which can cause a shuffle include repartition operations like repartition and coalesce, ‘ByKey operations (except for counting) like groupByKey and reduceByKey, and join operations like cogroup and join.

# Transformations
|方法|说明|宽依赖or窄依赖|
|---|---|---|
|map(func)|对调用map的RDD数据集中的每个element都使用func方法，生成新的RDD。|窄依赖|
|filter(func)|对RDD中所有元素调用func方法，生成将满足条件数据集以RDD形式返回。|窄依赖|
|flatMap(func)|对RDD中所有元素调用func方法，然后将结果扁平化，生成新的RDD。理解为**降维**|窄依赖|
|mapPartitions(func)|类似于Map，不过Map作用的对象是每个元素，而mapPartitions作用的对象是分区。由于分区必然不跨节点，所以通过mapPartitions来实现一些资源在分区内共享，比如数据库连接等|窄依赖|
|mapPartitionsWithIndex(func)|类似于mapPartitions，提供多1个index参数，表示分区的索引|窄依赖|
|sample(withReplacement, fraction, seed)|抽样，返回RDD一个子集。withReplacement同一个元素是否可以重复抽样，fraction样本大小占总样本大小的百分比|窄依赖|
|union(otherDataset)|返回一个新的RDD，包含源RDD和给定RDD的元素的集合。求并集，且不去除重复集合。|窄依赖|
|intersection(otherDataset)|求交集，去除重复元素。|未知|
|distinct([numTasks]))|去除重复元素，生成新的RDD。|窄依赖|
|groupByKey([numTasks])|返回`(K,Iterable[V])`，将key相同的value组成一个集合。|宽依赖|
|reduceByKey(func, [numTasks])|对key相同的value调用func。按key聚合。|宽依赖|
|aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])|暂时略，通过例子说明。|宽依赖|
|combineByKey(createCombiner, mergeValue, mergeCombiners)|暂时略，通过例子说明。|宽依赖|
|foldByKey(zeroValue)(func)|暂时略，通过例子说明。|宽依赖|
|aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])|暂时略，通过例子说明。|宽依赖|
|sortByKey([ascending], [numTasks])|按照key来进行排序，Key需实现Ordered接口。ascending升序还是降序，使用的是RangePartitioner。|宽依赖|
|join(otherDataset, [numTasks])|当有两个KV的dataset(K,V)和(K,W)，返回的是(K,(V,W))的dataset,numPartitions为并发的任务数。|Hash分区窄依赖，Range分区宽依赖|
|cogroup(otherDataset, [numTasks])|将当有两个key-value对的dataset(K,V)和(K,W)，返回的是(K, (Iterable[V], Iterable[W]))的dataset,numPartitions为并发的任务数。|宽依赖|
|cartesian(otherDataset)|返回该RDD与其它RDD的笛卡尔积。|宽依赖|
|pipe(command, [envVars])|以管道的方式对分区执行脚本命令，处理当前进程的标准输出流，比如perl、bash。返回结果是RDD[String]|窄依赖|
|coalesce(numPartitions)|合并分区为numPartitions个分区，在RDD结果多次计算后数据减少可以通过合并分区提高效率|宽依赖|
|repartition(numPartitions)|重分区，随机Reshuffle RDD中的数据以增加/减少分区，并在其间平衡。分区数据的Shuffle通过网络完成。|宽依赖|
|repartitionAndSortWithinPartitions(partitioner)|根据给定的分区器partitioner重新分区RDD，并且在每个生成的分区中，通过它们的键对记录进行排序。这比调用repartition，然后在每个分区中排序更有效，因为it can push the sorting down into the shuffle machinery。|宽依赖|

**说一下Map与flatMap，flatMap是Map之后再flat，即是将((a,b),(c),(d,e))转化为(a,b,c,d,e)**

# Actions
 |方法|说明|
|---|---|
|reduce(func)|对RDD中的元素调用f，f必须是1个可交换和可关联的函数，以便Spark可以进行并行计算。即聚合所有RDD的元素为1个元素|
|collect()|返回包含RDD中所有元素的一个数组|
|count()|返回dataset中element的个数|
|first()|返回dataset中的第一个元素|
|take(n)|返回前n个elements|
|takeSample(withReplacement, num, [seed])|对dataset随机抽样，返回有num个元素组成的数组。withReplacement同一个元素是否可以重复抽样，num表示抽样个数|
|takeOrdered(n, [ordering])|使用自然顺序或自定义比较器返回RDD的前n个元素。|
|saveAsTextFile(path)|把dataset写到一个text file中，或者hdfs，或者hdfs支持的文件系统中，spark把没条记录都转换为一行记录，然后写到file中。|
|saveAsSequenceFile(path)|只能用在key-value对上，然后生成SequenceFile写到本地或者hadoop文件系统。|
|saveAsObjectFile(path)|生成ObjectFile写到本地或者hadoop文件系统。|
|countByKey()|对每个key出现的次数做统计，返回一个Map。|
|foreach(func)|在数据集的每一个元素上，运行函数func。|
|countByValue()(implicitord: Ordering[T] = null):Map[T, Long]|对RDD中每个元素出现的次数进行统计。|
