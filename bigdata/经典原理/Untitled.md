1. Hadoop常用端口

   1. dfs.namenode.http-address:50070
   2. dfs.datanode.http-address:50075
   3. secondaryNameNode辅助名称节点端口号:50090
   4. dfs.datanode.address:50010
   5. fs.defaultFS:8020  9000
   6. Yarn.resourcemanager.webapp.address:8088
   7. 历史服务器web访问端口:19888

2. hdfs读写流程

   1. 读
      1. client向namenode请求下载文件
      2. namenode返回目标文件的元数据
      3. client去每个对应datanode请求读数据
      4. 每个datanode返回传输数据
   2. 写
      1. client向namenode发送上传文件的请求
      2. namenode响应可以上传
      3. client请求上传第一个block(0-128M),请返回datanode
      4. 返回对应节点
      5. client向第一个Datanode请求建立Block传输通道
      6. dn1ying 

   

3. MR的Shuffle过程及Hadoop优化

4. Yarn的Job提交流程

5. Yarn的调度器

6. Hadoop参数调优

7. 数仓分层