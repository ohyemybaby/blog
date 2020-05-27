Spark介绍



1. 基本运算规则从存储介质中获取(采集)数据,然后进行计算,最后将结果存储到戒指中,所以主要应用于一次性计算,不适合数据挖掘和机器讯息
2. MR基于文件存储介质的操作,所以性能非常慢
3. MR和Hadoop紧密耦合在一起,无法动态替换



RresourceManager

NodeManager

ApplicationMaster

Driver(main方法)



