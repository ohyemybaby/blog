

| 选中业务过程 | 选择粒度 | 确认维度 | 确定事实 |
| ------------ | -------- | -------- | -------- |
| 订单表       |          | 用户     |          |
| 订单详情     |          | 时间     |          |
| 评论表       |          | 商品     |          |
| 收藏表       |          | 优惠券   |          |
| 加购表       |          | 地区     |          |
| 优惠券领用   |          | 活动     |          |
| 支付表       |          |          |          |
| 退款         |          |          |          |
|              |          |          |          |



## ods层创建分区

hive的内部表外部表

自己用是内部表,删除的时候元数据和表数据一并删除

大家一起用是外部表、临时表也可能公用,也要用外部表,删除的时候只删除外部表,不删除元数据

启动事件日志处理

```sql
create databases gmall;
use gmall;
drop table if exist ods_start_log;
create external table ods_start_log(`line` string)
partitioned by (`dt` string)
stored as 
  inputformat 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  outputformat 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat' location '/warehouse/gmall/ods/ods_start_log';
  -- 创建之前先删除、创建外部表、ods用字符串保持不变、dt分区、输入是lzo压缩、输出是正常文本
```

加载

/origin_data/gmall/log/topic_start/2020-03-10

/warehouse/gmall/ods/ods_start_log

```shell
load data inpath '/origin_data/gmall/log/topic_start/2020-03-10' into table gmall.ods_start_log partition(dt='2020-03-10')
# 如果没有这个分区就直接创建了,有两个字段line 和dt
# 加载后原来路径数据消失,新表空间已经有数据
```

```sql
-- 检查hive数据是否加载成功
select * from ods_start_log where dt='2020-03-10' limit 2;
```

```shell
# 为表创建lzo压缩索引
hadoop jar /opt/module/hadoop-2.7.2/share/hadoop/common/hadoop-lzo-0.4.20.jar com.hadoop.compression.lzo.DistributedLzoIndexer /warehouse/gmall/ods/ods_start_log/dt=2020-03-10

# number of split:2
#同时可以查看文件
```

事件日志



```sql
drop table if exists ods_event_log;
create external table ods_event_log(`line` string) partitioined by (`dt` string) stored as inputformat 'com.hadoop.mapred.DeprecatedLzoTextInputFormat' outputformat 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat' location '/warehouse/gmall/ods/ods_event_log'

load data inpath '/origin_data/gmall/log/topic_event/2020-03-10' into table gmall.ods_event_log partition(dt='2020-03-10')

select * from ods_event_log where dt='2020-03-10' limit 2;

hadoop jar /opt/module/hadoop-2.7.2/share/hadoop/common/hadoop-lzo-0.4.20.jar com.hadoop.compression.lzo.DistributedLzoIndexer /warehouse/gmall/ods/ods_event_log/dt=2020-03-10
```



### shell中单引号和双引号区别

```shell
#!/bin/sh
do_date=$1
echo '$do_date'
echo "$do_date"
#取出变量值、谁在外面谁起作用
echo "'$do_date'"
echo '"$do_date"'
#执行某命令
echo `date`
```

每日导数据脚本

```shell
#!/bin/bash

#2定义变量 不能有空格
hive=/opt/module/hive/bin/hive
APP=gmall

#3获取时间
if [-n "$1"]; then
    do_date=$1
else
    do_date=`date -s '-1 day' + %F`
fi
#4sql

sql="load data inpath '/origin_data/gmall/log/topic_event/$do_date' into table ${APP}.ods_event_log partition(dt='$do_date'); load data inpath '/origin_data/gmall/log/topic_start/$do_date' into table ${APP}.ods_start_log partition(dt='$do_date');"

#5执行sql  -e 不需要启动客户端
$hive -e "$sql"

#创建lzo索引

hadoop jar /opt/module/hadoop-2.7.2/share/hadoop/common/hadoop-lzo-0.4.20.jar com.hadoop.compression.lzo.DistributedLzoIndexer /warehouse/gmall/ods/ods_event_log/$do_date
hadoop jar /opt/module/hadoop-2.7.2/share/hadoop/common/hadoop-lzo-0.4.20.jar com.hadoop.compression.lzo.DistributedLzoIndexer /warehouse/gmall/ods/ods_start_log/$do_date
```

```shell
chmod 777 hdfs_to_ods_log.sh #加权限
hdfs_to_ods_log.sh 2020-03-11 #执行脚本
```

```sql
select * from ods_start_log where dt='2020-03-11' limit 2;
select * from ods_event_log where dt='2020-03-11' limit 2;
```

**脚本执行时间**

每日凌晨30分~1点,大概执行40m~50m



### 业务数据导入

hql和业务数据基本一致,可以增加条件

```sql
drop table if exists ods_order_info;
create external table ods_order_info(
    `id` string comment '订单号',
    `final_total_amount` decimal(10,2) comment '订单金额'
) comment '订单表'
partitioned by (`dt` string)
row format delimited fields terminated by '\t'
stored as 
  inputformat 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  outputformat 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
  locatioin '/warehouse/gmall/ods/ods_order_info/';
```

```shell
load data inpath '/origin_data/gmall/db/order_info/$do_date' overwrite into table ${APP}.ods_order_info partition(dt=$do_date)
```

```shell
show tables;
```

```shell
#数据加载自动化脚本
#遇到日期换成$do_date
#遇到表加上${APP}
#有些特殊表不用每次都导
#业务表不需要创建lzo索引,因为sqoop创建的时候已经创建过了,日志的时候需要创建索引
sql2="load data inpath '' overwrite into table ${APP}.ods_base_province"

case $1 in
"first"){
    $hive -e "$sql1"
    $hive -e "$sql2"
};;;
"all"){
    $hive -e "$sql1"
}
esac
```

```shell
hdfs_to_ods_db.sh all 2020-03-11;
```

```sql
select * from ods_order_detail where dt='2020-03-11' limit 2;
```



数仓分层:

ODS层:

1. 保持数据原貌不做任何修改,备份
2. 创建分区表,防止后续的全表扫描
3. 采用lzo压缩,并创建索引(切片)
4. 创建外部表(多人共用),内部表(自己使用的临时表)

DWD层:维度聚合

1. 数仓维度建模(星型模型)-》维度退化 商品表+品类表+spu表+三级分类+二级分类+一级分类,减少后续大量join操作.省份+地区表-》地区表.  活动表+活动规则表=〉活动表
2. 数据清洗(ETL),hive sql、MR、Python、Kettle、SparkSQL
3. 采用lzo压缩
4. 采用列式存储 parquet
5. 脱敏(手机号、身份证号、个人信息 135****0013)
6. 对用户行为数据进行解析 event事件表(10张表,解析)

DWS层:每天各个主题的行为数据,会站在维度的角度去分析用户、商品、优惠券主题

DWT层:从开始创建,一直到现在的累积数据

ADS层:分析具体报表,给老板看的,直观数据(日活100w、gmv1000w)

为什么要做数仓、为什么数仓分层?

减少重复操作、方便定位、隔离原始数据

数据集市和数据仓库的概念

数据集市针对部门级,数据少一些

数据仓库针对共识,数据多一些

命名规范:

每一层,某一个主题,

蛇形命名规则

dim/fact    ***_tmp      XXX_log

数据源\_to\_目标\_db.sh



范式理论

1. 属性不可切割 5台电脑
2. 不存在部分函数依赖  AB-> C  A / B ->C 拆分
3. 不存在传递函数依赖  A-> B -> C 但是C推不出A

关系建模&维度建模

OLTP和LOAP 

维度建模

1. 星型维度.我们的追求
2. 雪花维度.灵活方便
3. 星座维度

维度表:名词

事实表:动词、度量值,可累加

1. 事务型事实表,一旦产生就不变化
2. 周期型快照事实表,按周期产生
3. 累积型快照事实表 ,按变化产生

DWD层

1. 选中业务过程
   1. 选择感兴趣:下单、支付、退款、活动
   2. 全部业务线(前提:时间允许)淘宝、天猫、支付宝 先做淘宝、后续逐渐做
2. 声明粒度
   1. 一行代表信息:一条订单、一天的订单、一周的订单
   2. 选择最小粒度
3. 确认维度
   1. 维度退化:时间地点人物
4. 确认事实
   1. 度量值  个数、件数、金额

DWS、DWT  主题宽表  在维度的视觉看待业务过程

ADS出报表

ODS







