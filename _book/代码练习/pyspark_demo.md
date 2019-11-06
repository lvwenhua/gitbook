# pyspark实例练习

#### 1.pyspark过滤指定的一批数据--写sql语句

```
from pyspark import SparkConf,SparkContext
from pyspark.sql import SparkSession


spark=SparkSession.builder.master("local[2]").appName("testSQl").getOrCreate()
#读取文件
lines=spark.read.json("hdfs://node01:8020/lvwenhua/input/part-r-00004.dms")
#过滤前的条目数
count1=lines.count()
print count1
##写sql过滤
lines.createOrReplaceTempView("filter")
sqltest="select * from filter where  rule_100_id!='20190219000100050131' or rule_100_id is null"
#import spark.implicits._
result=spark.sql(sqlQuery=sqltest).toJSON()
print(result.count())
result.take(4)

result.saveAsTextFile("hdfs://node01:8020/lvwenhua/outputS1")
print "存储完毕！"
spark.stop()

```

#### 2.pyspark过滤指定的一批数据--通过filter方法

```
from pyspark import SparkConf,SparkContext
from pyspark.sql import SparkSession


spark=SparkSession.builder.master("local[2]").appName("testSQl").getOrCreate()
#读取文件
lines=spark.read.json("hdfs://node01:8020/lvwenhua/input/part-r-00004.dms")
#过滤前的条目数
count1=lines.count()
print count1
##不写sql
lines.createOrReplaceTempView("filter")

result=lines.filter("rule_100_id!='20190219000100050131' or rule_100_id is null").toJSON()

print(result.count())
result.take(4)

result.saveAsTextFile("hdfs://node01:8020/lvwenhua/outputS4")
print "存储完毕！"
spark.stop()
```



#### 3.pyspark 读取Hdfs文件进行一定操作并存储到kafka中

```
# -*- coding: utf-8 -*-

import findspark
findspark.init(spark_home="/usr/hdp/current/spark2-client/",python_path="/root/anaconda2/bin/python")
from functools import reduce 
from pyspark.sql import SparkSession
from pyspark.sql import SparkSession
from kafka import KafkaProducer
import json
from pyspark.sql.types import Row
from kafka import KafkaProducer,SimpleProducer,SimpleClient

from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils,TopicAndPartition

def get_spark(master="yarn", memory="1g", local_dir="/tmp",app_name="preporcess",queue="default",engine_id="asdf"):
    
    msg = "Please Note..... \n      spark up."
    '''
    创建spark content
    
    输入:
        master: spark 运行模式
        memory: spark 执行节点的内存
        local_dir: spark的本地目录位置 需要容量较大
    输出:
        spark content
    '''
    spark = SparkSession.builder.appName(app_name).master("yarn").config(
        "spark.shuffle.service.enabled", True
    ).config(
        "spark.executor.instances", 2
    ).config(
        "spark.executor.cores", 2
    ).config( "spark.driver.memory", "4G"
    ).config(
        "spark.executor.memory", "3G"
    ).config(
        "spark.shuffle.memoryFraction", "0.6"
    ).config(
        "spark.default.parallelism", 400
    ).config(
        "spark.local.dir", "/tmp"
    ).config(
        "spark.driver.maxResultSize", "5G"
    ).config(
        "spark.yarn.queue", "http_project"
    ).config(
        "spark.streaming.kafka.maxRatePerPartition",100
    ).getOrCreate()
    
    print("create spark session", spark.sparkContext.master)
    print(msg)
    return spark  

def send_kafka(rows):
    client = SimpleClient('hdp2.buptnsrc.com:6667,hdp4.buptnsrc.com:6667,hdp5.buptnsrc.com:6667')
    producer = SimpleProducer(client,async_send=True,batch_send_every_n=400,batch_send_every_t=10)
    for row in rows:
        producer.send_messages("testkafka",json.dumps(row.asDict()))
    producer.stop()
    
if __name__ == '__main__':
    print("111")
    #spark = SparkSession.builder.getOrCreate()
    src_path  = "/http-data/copy_file/2019-05-02/00-00-00/10.3.200.155/2019-05-02-00-05_INFO.txt"
    task_id  = "kafka-test"
    yesterday  = "2019-05-02-14-15-16"
    upload_host  = "10.3.200.155"
    spark = get_spark(master="yarn",app_name="kafka-test",queue="default")
    df=spark.read.option("mode","DROPMALFORMED").json(src_path)
    
    counts=df.count()
    print("lwh"+str(counts))

    ##将数据存入kafka
    
    df.foreachPartition(send_kafka)
    print("完成!!!")
    
    spark.stop()
        
```

