数仓搭建

1. 对用户行为数据解析
2. 对核心数据进行判空过滤
3. 对业务数据采用维度模型重新建模,即维度退化

DWD层(用户行为启动表数据解析)



启动hive

```shell
cd /opt/module/hive
nohup bin/hive --service metastore &
bin/hive
use gmall;
show tables;
select * from ods_start_log where dt='2020-03-10' limit 2;
drop table if exists dwd_start_log;
create table 
partitioned by (dt string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_start_log/'
tblproperties('parquet.compression'='lzo');
# parquet(列式存储+loz压缩切片)
# parquet对标的是orc  parquet是spark默认的存储方式

```



```sql
insert overwrite table dwd_start_log
partition(dt='2020-03-10')
select 
	get_json_object(line, "$.mid") mid_id,
	get_json_object(line, "$.uid") user_id
from ods_start_log
where dt='2020-03-10'
```



```shell
#!/bin/sh
APP=gmall
hive=/opt/module/hive/bin/hive
if [ -n "$1"]; then
    do_date=$1
else
    do_date=`date -d '-1 day' + %F`
fi

#时间换成$do_date  表换成${APP}.
sql=""
$hive -e "$sql"
```

```shell
chmod 777 ods_to_dwd_start_log.sh
ods_to_dwd_start_log.sh 2020-03-11
select * frm dwd_start_log where dt='2020-03-11' limit 2;
```

### 用户事件日志表

```sql
select * from ods_event_log where dt='2020-03-11' limit 2;
```

![image-20200511115454971](/Users/sjy/Library/Application Support/typora-user-images/image-20200511115454971.png)

































