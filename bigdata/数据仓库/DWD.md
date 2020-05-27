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



![手绘图.png](https://github.com/ohyemybaby/blog/blob/master/数据仓库/screenshorts/image-20200511115454971.png)

DWD层创建基础明细表分析

UDF:一进一出

UDTF:一进多出

自定义UDF:

```java
//自定义UDF函数,根据传入进来的key,获取对应的value值
String x = new BaseFieldUDF().evaluate(line,"mid");
//将传入的line,用"|"切割:取出服务器事件serverTime和json数据
//根据切割后获取的json数据,创建一个JSONObject对象
//判断输入的key值,如果key为st,返回serverTime
//判断输入的key值,如果key为et,返回上述JSONObject对象的et
//判断输入的key值,如果key既不是st,又不是et,先获取JSONObject的cm,然后根据key值,获取cmJSON中的value.
```

1. 创建maven工程:hivefunction

2. 创建包名, 

3. hive的https设置

   ```properties
   #配置在MAVEN.Runner的VM Options
   -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true
   -Dmaven.wagon.http.ssl.ignore.validity.dates=true
   ```

   Pentane-aggdesigner-algorithm.jar包下载失败

   因为官方把这个包挪到了spring-plugin仓库里,但是阿里并没有修改

   ```xml
   <repositories>
       <repository>
         <id>spring-plugin</id>
         <url>https://repo.spring.io/plugins-release/</url>
       </repository>
   </repositories>
   ```

   内存溢出,可以在maven.runner的vmoptions里配置-Xss

   ```java
   public class BaseFieldUDF extends UDF{
     public String evaluate(String line , String key){
       //切割
       String[] log = line.split("\\|");
       // 非法性判断
       if(log.length != 2 || StringUtils.isBlank(log[1].trim())){
         return "";
       }
       JSONObject json = new JSONObject(log[1].trim());
       String result = "";
       
       // 根据传入数据的取值
       if("st".equals(key)){
         //返回服务器事件
         return log[0].trim();
       } else if("et".equals(key)){
         if(json.has("et")){
            result = json.getString("et");
         }
       }else{
         //获取cm对应的value
         JSONObject cm = json.getJSONObject("cm");
         if(cm.has(key)){
           result = json.getString(key);
         }
       }
       retunr result;
     }
   }
   ```

   ### 自定义UDTF

   UDF取出json,传入UDTF

需要继承GenericUDTF,重写initialize();process();close();



```java
public class EventJsonUDTF extends GenericUDTF{
  public StructObjectInspector initialize(StructObjectInspector argOIs){
    // 定义UDTF返回值类型和名称
    
    List<String> fieldName = new ArrayList<>();
    fieldName.add("event_name");
    fieldName.add("event_json");
    fieldType.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
    fieldType.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector)
    return ObjectInspectorFactory.getStandardstructobjectInspector(fieldName,fieldType);
  }
  
  public void process(Object[] objects) throws HiveException{
    //传入的是json array=>UDF传入et
    String input = object[0].toString();
    if(StringUtils.isBlank(input)){
      return;
    }else{
      JSONArray ja = new JSONArray(input);
      if(ja == null){
        return;
      }
      for(int i = 0;i<ja.length();i++){
        try{
          String[] result = new String[2];
          result[0] = ja.getJSONObject(i).getString("en");
          result[1] = ja.getJSONObject(i);
        }catch(JSONException ex){
          continue;
        }
        forward(result);
      }
    }
  }
}
```

打包

package 

抛异常 StackOverflowError

-Xmx512m -Xms128m -Xss2m





要上传不带依赖的jar包`hivefunction-1.0-snapshot.jar`

```shell
hadoop fs -mkdir -p /user/hive/jars
hadoop fs -put hivefunction-1.0-snapshot.jar /user/hive/jars
```



本地模式,要上传到集群变成永久函数

创建永久函数与开发好的java class关联

```shell
#UDF
hive(gmall)> create function base_analizer as 'com.xxx.udf.BaseFieldUDF' using jar 'hdfs://hadoop102:9000/user/hive/jars/hivefunction-1.0-snapshot.jar';
#UDTF
create function flat_analizer as 'com.xxx.udtf.EventJsonUDTF' using jar 'hdfs://hadoop102:9000/user/hive/jars/hivefunction-1.0-snapshot.jar';
```

jar包更新,替换jar包,重启hive客户端



```shell
#!/bin/shell
hive=/opt/module/hive/bin/hive
APP=gmall

if[-n "$1"]; then
  do_date=`date -d '-1 day' +%F`
else
  do_date=$1
fi


sql="
use ${APP};
insert overwrite table ${APP}.dwd_base_event_log
partition(dt='$do_date')
select 
	base_analizer(line,"mid") mid_id,
	event_name,
	event_json,
	base_analizer(line,'st') as server_time
from ${APP}.ods_event_log lateral viewe flat_analizer(base_analizer(line,'et')) tmp_flat as event_name,event_json
where dt='$do_date' and base_analizer(line, "et")<>'';"

hive -e "$sql"
```



```sql
set hive.exec.dynamic.partition.mode=nonstrict; -- 动态分区,非严格模式

insert overwrite table dwd_fact_coupon_use
partition(dt)
select
	if(new.id is null,old.id,new.id),
	if(new.coupon_id is null, old.coupon_id,new.coupon_id),
	if(new.user_id is null,old.user_id,new.user_id),
	if(new.order_id is null,old.order_id,new.order_id),
	if(new.coupon_status is null,old.coupon_status,new.coupon_status),
	if(new.get_time is null,old.get_time,new.get_time),
	if(new.using_time is null,old.using_time,new.using_time),
	if(new.used_time is null,old.used_time,new.used_time),
	date_format(	if(new.get_time is null,old.get_time,new.get_time),'yyyy-MM-dd')
from
(
  select 
  id,
  coupon_id,
  user_id,
  coupon_status,
  get_time,
  using_time,
  used_time
  from dwd_fact_coupon_use   -- 老数据
  where dt in
  (
  	select date_format(get_time,'yyyy-MM-dd')
    from ods_coupon_use
    where dt='2020-03-10' -- 3月10号新增和修改的数据
  )
)old
full outer join -- 全表链接
(
	select
  id,coupon_id,user_id,order_id,coupon_status,get_time,using_time,used_time
  from ods_coupon_use
  where dt='2020-03-10' -- 这个时间可能是领取时间、使用时间和支付时间
)new 
on old.id = new.id
```



















