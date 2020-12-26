# 大数据组件基准测试

## 环境

整个集群运行于一台物理服务器上的多个虚拟机

网络CIDR为 `192.168.0.0/24` 网关为 `192.168.0.1`

宿主机硬件:

类型: ESXI
服务器型号: Dell Inc. PowerEdge R710
处理器类型: Intel(R) Xeon(R) CPU X5650 @ 2.67GHz

虚拟机:

|主机名称|IP |用户名|密码|操作系统|硬件|
|---|---|----- | ---- |---|---|
| cloudera-manager.hadoopcluster.local | 192.168.5.180 | root   | password |ubuntu-18.04|4vCPU 8GB内存 64GB磁盘|
| hadoop-01.hadoopcluster.local        | 192.168.5.181 | root   | password |ubuntu-18.04|8vCPU 16GB内存 64GB磁盘|
| hadoop-02.hadoopcluster.local        | 192.168.5.182 | root   | password |ubuntu-18.04|8vCPU 16GB内存 64GB磁盘|
| hadoop-03.hadoopcluster.local        | 192.168.5.183 | root   | password |ubuntu-18.04|8vCPU 16GB内存 64GB磁盘|

zookeeper: `hadoop-01.hadoopcluster.local:2181,hadoop-02.hadoopcluster.local:2181,hadoop-03.hadoopcluster.local:2181`

dfs: `hdfs://hadoop-01.hadoopcluster.local:8020`

dfs-web: http://192.168.5.181:9870

kafka: `hadoop-01.hadoopcluster.local:9092,hadoop-02.hadoopcluster.local:9092,hadoop-03.hadoopcluster.local:9092`

hbae-web: http://192.168.5.181:16010

hue: http://192.168.5.181:8888    hdfs/password   admin/admin

hive-web: http://192.168.5.181:10002

spark-web: http://192.168.5.181:18088
spark: yarn

## 基准测试

### hdfs

（1）HDFS的读写性能
HDFS单节点的写数据吞吐量达到50MB/s；
HDFS单节点的读数据吞吐量达到60MB/s；
Namenode处理TPS吞吐达到600次/s。

[Hadoop Benchmarking](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/Benchmarking.html)

写入测试(写10文件/每个文件100MB)：

```sh
$ HADOOP_USER_NAME=hdfs /etc/alternatives/hadoop jar /opt/cloudera/parcels/CDH/jars/hadoop-mapreduce-client-jobclient-3.0.0-cdh6.3.2-tests.jar TestDFSIO -write -nrFiles 10 -fileSize 100MB
20/11/16 09:59:44 INFO fs.TestDFSIO: ----- TestDFSIO ----- : write
20/11/16 09:59:44 INFO fs.TestDFSIO:             Date & time: Mon Nov 16 09:59:44 UTC 2020
20/11/16 09:59:44 INFO fs.TestDFSIO:         Number of files: 10
20/11/16 09:59:44 INFO fs.TestDFSIO:  Total MBytes processed: 1000
20/11/16 09:59:44 INFO fs.TestDFSIO:       Throughput mb/sec: 81.1
20/11/16 09:59:44 INFO fs.TestDFSIO:  Average IO rate mb/sec: 85.44
20/11/16 09:59:44 INFO fs.TestDFSIO:   IO rate std deviation: 22.68
20/11/16 09:59:44 INFO fs.TestDFSIO:      Test exec time sec: 28.29
```

读取测试：

```sh
$ HADOOP_USER_NAME=hdfs /etc/alternatives/hadoop jar /opt/cloudera/parcels/CDH/jars/hadoop-mapreduce-client-jobclient-3.0.0-cdh6.3.2-tests.jar TestDFSIO -read -nrFiles 10 -fileSize 100MB
20/11/16 09:33:07 INFO fs.TestDFSIO: ----- TestDFSIO ----- : read
20/11/16 09:33:07 INFO fs.TestDFSIO:             Date & time: Mon Nov 16 09:33:07 UTC 2020
20/11/16 09:33:07 INFO fs.TestDFSIO:         Number of files: 10
20/11/16 09:33:07 INFO fs.TestDFSIO:  Total MBytes processed: 1000
20/11/16 09:33:07 INFO fs.TestDFSIO:       Throughput mb/sec: 404.37
20/11/16 09:33:07 INFO fs.TestDFSIO:  Average IO rate mb/sec: 437.59
20/11/16 09:33:07 INFO fs.TestDFSIO:   IO rate std deviation: 117.96
20/11/16 09:33:07 INFO fs.TestDFSIO:      Test exec time sec: 25.47
```

事务处理TPS(namenode 吞吐量测试)：

```sh
$ HADOOP_USER_NAME=hdfs /etc/alternatives/hadoop org.apache.hadoop.hdfs.server.namenode.NNThroughputBenchmark   -fs hdfs://hadoop-01.hadoopcluster.local:8020 -op create -threads 10 -files 1000 -filesPerDir 4 -close
20/11/16 09:37:33 INFO namenode.NNThroughputBenchmark: Starting benchmark: create
20/11/16 09:37:33 INFO namenode.NNThroughputBenchmark: Generate 1000 intputs for create
20/11/16 09:37:33 FATAL namenode.NNThroughputBenchmark: Log level = ERROR
20/11/16 09:37:33 INFO namenode.NNThroughputBenchmark: Starting 1000 create(s).
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark:
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: --- create inputs ---
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: nrFiles = 1000
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: nrThreads = 10
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: nrFilesPerDir = 4
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: --- create stats  ---
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: # operations: 1000
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: Elapsed Time: 904
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark:  Ops per sec: 1106.1946902654868
20/11/16 09:37:34 INFO namenode.NNThroughputBenchmark: Average Time: 4
```

### hbase

HBase读写性能
单台RegionServer写数据吞吐率达到2500 ops/s以上；
单台RegionServer读数据吞吐率达到8000 ops/s以上。

[PerformanceEvaluation tool](https://hbase.apache.org/book.html#_hbase_pe)

创建测试表：

```sh
JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop /opt/cloudera/parcels/CDH/lib/hbase/bin/hbase shell
> create 'TestTable','info0'
```

顺序读：

```sh
JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop HADOOP_USER_NAME=hdfs /opt/cloudera/parcels/CDH/lib/hbase/bin/hbase pe  --autoFlush=true  --nomapred --rows=10000  sequentialRead 5
```

顺序写：

```sh
JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop HADOOP_USER_NAME=hdfs /opt/cloudera/parcels/CDH/lib/hbase/bin/hbase pe  --autoFlush=true  --nomapred --rows=10000 sequentialWrite 5
```

> 删除表： disable 'TestTable';drop 'TestTable'

### kafka

★数据采集接入（需提供具有CNAS认证，执行GJB141-2004标准的第三方测试报告扫描件）：
(1)文本采集速率不低于50000条/秒/单机；
(2)日志采集速率不低于50000条/秒/单机；
(3)数据库采集器数值50000行/秒/单机；

数据采集速率：

```sh
$ JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera /opt/cloudera/parcels/CDH/lib/kafka/bin/kafka-producer-perf-test.sh --topic bench  --throughput 100000 --num-records 1000000 --record-size 128 --producer-props bootstrap.servers=hadoop-01.hadoopcluster.local:9092,hadoop-02.hadoopcluster.local:9092,hadoop-03.hadoopcluster.local:9092 # 10万数据每秒，共计100万数据
20/11/16 09:56:59 INFO utils.AppInfoParser: Kafka version: 2.2.1-cdh6.3.2
20/11/16 09:56:59 INFO utils.AppInfoParser: Kafka commitId: unknown
20/11/16 09:56:59 INFO clients.Metadata: Cluster ID: rIIXVAl7Sx2dYekibQET8g
479813 records sent, 93530.8 records/sec (11.42 MB/sec), 10.7 ms avg latency, 330.0 ms max latency.
20/11/16 09:57:09 INFO producer.KafkaProducer: [Producer clientId=producer-1] Closing the Kafka producer with timeoutMillis = 9223372036854775807 ms.
1000000 records sent, 99820.323418 records/sec (12.19 MB/sec), 20.38 ms avg latency, 334.00 ms max latency, 1 ms 50th, 158 ms 95th, 280 ms 99th, 301 ms 99.9th.
```

### spark

★数据计算处理（需提供具有CNAS认证，执行GJB141-2004标准的第三方测试报告扫描件）
(1)并行计算：针对“WordCount”用例，平均每节点处理能力达到1.63 GB/分钟；
(2)流计算：采用Storm/Flink等流式计算框架，平均每个消息处理时延<200ms，集群平均每秒处理数据量达到80MB/秒，平均数据处理速度达到3,0000条/秒。

workdcount:

```sh
cd /root/HiBench
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/micro/wordcount/prepare/prepare.sh
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/micro/wordcount/hadoop/run.sh
```

```sh
HADOOP_USER_NAME=hdfs SPARK_HOME=/opt/cloudera/parcels/CDH/lib/spark SPARK_MASTER_HOST=yarn ~/spark-bench_2.3.0_0.4.0-RELEASE/bin/spark-bench.sh ~/spark-bench_2.3.0_0.4.0-RELEASE/conf/
```

```sh
bin/spark-submit --master yarn --class org.apache.spark.examples.streaming.NetworkWordCount  examples/jars/spark-examples_2.11-2.4.0-cdh6.3.2.jar  192.168.5.183 9999
```

### hive

[hive-testbench](https://github.com/hortonworks/hive-testbench)

★数据查询服务（需提供具有CNAS认证，执行GJB141-2004标准的第三方测试报告扫描件）
(1)执行查询、添加、编辑和删除业务时，平均响应时间不得超过3秒；
(2)执行跨库、跨表查询等综合业务时，平均响应时间不得超过15秒;

183 节点：

```sh
cd ~/hive-testbench;PATH=${PATH}:/etc/alternatives/ HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop HADOOP_USER_NAME=hdfs ./tpcds-setup.sh  2 #生成2GB测试数据
PATH=${PATH}:/etc/alternatives/ HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop HADOOP_USER_NAME=hdfs hive
> use tpcds_bin_partitioned_orc_2;
```

执行查询(可以进入hue界面进行查询)：

```sh

```

## Hi Bench

wordcount:

```sh
cd /root/HiBench
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/micro/wordcount/prepare/prepare.sh
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/micro/wordcount/hadoop/run.sh
```

查看报告：

```sh
cat report/wordcount/hadoop/bench.log
```

dfsio磁盘吞吐:

```sh
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/micro/dfsioe/prepare/prepare.sh
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/micro/dfsioe/hadoop/run.sh
```

```sh
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/streaming/identity/prepare/genSeedDataset.sh
HADOOP_USER_NAME=hdfs JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera bin/workloads/streaming/identity/prepare/dataGen.sh
```

sql:

```sh
JAVA_HOME=/usr/lib/jvm/java-8-oracle-cloudera HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop HADOOP_USER_NAME=hdfs bin/workloads/sql/scan/prepare/prepare.sh

```
