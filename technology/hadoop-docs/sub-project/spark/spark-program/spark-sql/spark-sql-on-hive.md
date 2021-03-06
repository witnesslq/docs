# Spark Sql ON Hive

## 一、配置文件

### 1. spark-defaults.conf

$SPARK_HOME/conf/spark-defaults.conf

```
spark.master  yarn
spark.submit.deployMode client
spark.eventLog.enabled  true

spark.driver.cores  4
spark.driver.memory  8g

spark.executor.cores 4
spark.executor.memory 8g

spark.eventLog.dir hdfs://dw2:8020/tmp/spark

spark.serializer org.apache.spark.serializer.KryoSerializer
```

### 2. hive-site.xml

ln -s $HIVE_HOME/conf/hive-site.xml $SPARK_HOME/conf/hive-site.xml

``` xml
<!-- 设置 metastore thrift 地址 -->
<property>
  <name>hive.metastore.uris</name>
  <value>thrift://uhadoop-ociicy-master1:9083,thrift://uhadoop-ociicy-master2:9083</value>
</property>
```

### 3. yarn-site.xml

ln -s $HADOOP_HOME/conf/yarn-site.xml $SPARK_HOME/conf/hive-site.xml



### 4. 配置优化

- Spark SQL 已经使用了 Catalyst 查询优化器模块, 优化过了, 所以可以提供给我们的优化参数不多

``` sql

以下的单位是字节: 256 * 1024 * 1024

*. 小文件合并，默认是4M，可以调大点，不然每个小文件就是一个Task ,
  spark.sql.files.maxPartitionBytes=268435456;

*. 调节每个partition大小，默认 128M，可以适当调大点
  spark.sql.files.openCostInBytes=268435456;

*. 两个表shuffle，如join。这个最有用，经常使用的。 默认是10M，调成100M，甚至是1G。
  spark.sql.autoBroadcastJoinThreshold=268435456;

*. 并行度的优化，这要根据自己集群的配置来调节，默认情况下是 200
  spark.sql.shuffle.partitions=6;

*. 如果开启, 表示所有的 JDBC/ODBC connection 共用用一份 SQL 配置和临时函数注册表
  spark.sql.hive.thriftServer.singleSession=false;

```



## 二、thrift-server 服务

- 开启 thrift-server 服务
- [spark thrift-jdbcodbc-serve 文档](http://spark.apache.org/docs/latest/sql-programming-guide.html#running-the-thrift-jdbcodbc-server)

### 1. 启动 Thriftserver 进程

``` sh

1. Yarn 模式
  (1) yarn-client 模式 , 让 Yarn 管理进程 , Driver 运行在客户端 ,Work 运行在 NodeManager 上
    $SPARK_HOME/sbin/start-thriftserver.sh \
    --master yarn \
    --deploy-mode client \
    --name spark-sql-test \
    --driver-cores 2 \
    --driver-memory 4096M \
    --num-executors 2 \
    --executor-memory 2048M \
    --jars file://path/xxx.jar,file://path/xxx.jar \
    --hiveconf hive.server2.thrift.port=10002

  (2) yarn-cluster模式, 集群模式目前不支持
    ./sbin/start-thriftserver.sh \
    --master yarn \
    --deploy-mode cluster \
    --name spark-sql \
    --driver-cores 2 \
    --driver-memory 500M \
    --hiveconf hive.server2.thrift.port=10002

2. standalone 模式
  $SPARK_HOME/sbin/start-thriftserver.sh \
  --master spark://uhadoop-ociicy-task3:7077 \
  --deploy-mode client \
  --name spark-sql \
  --driver-cores 2 \
  --driver-memory 500M \
  --hiveconf hive.server2.thrift.port=10002


3. JDBC 操作 hive
  $SPARK_HOME/bin/beeline !connect jdbc:hive2://hostname:10002

```


### 2. Thriftserver 实际案例

``` sh
启动服务
  $SPARK_HOME/sbin/start-thriftserver.sh \
  --master yarn \
  --deploy-mode client \
  --name spark-sql-client-2.x \
  --driver-cores 2 \
  --driver-memory 2048M \
  --num-executors 2 \
  --executor-memory 2048M \
  --jars file://$HIVE_HOME/lib/hive-json-serde.jar,file://$HIVE_HOME/lib/hive-contrib.jar,file://$HIVE_HOME/lib/hive-serde.jar \
  --hiveconf hive.server2.thrift.port=10002 \
  --conf spark.sql.hive.thriftServer.singleSession=false \
  --conf spark.sql.files.maxPartitionBytes=268435456 \
  --conf spark.sql.files.openCostInBytes=268435456 \
  --conf spark.sql.autoBroadcastJoinThreshold=268435456 \
  --conf spark.sql.shuffle.partitions=12 \
  --conf spark.broadcast.compress=true \
  --conf spark.io.compression.codec=org.apache.spark.io.SnappyCompressionCodec

连接
  $SPARK_HOME/bin/beeline -u jdbc:hive2://hostname:10002/default -nhadoop -phadoop

```


## 三、Spark Sql Udf

- Hive UDF 与 Spark UDF 通用
- [UDF](technology/hadoop-docs/sub-project/hive/hive-udf.md)
