# scala-spark实例练习

#### 1.sparkstreaming+kafka测试消费

```
package demo

import org.apache.spark.streaming.dstream.{DStream, ReceiverInputDStream}
import org.apache.spark.streaming.kafka.KafkaUtils
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkConf, SparkContext}

object sparkstreamtest {
  def main(args: Array[String]): Unit = {
    val conf: SparkConf = new SparkConf().setMaster("local[2]").setAppName("testStreaming")
    val sc = new SparkContext(conf)
    val ssc = new StreamingContext(sc,Seconds(5))

    val lines: ReceiverInputDStream[(String, String)] = KafkaUtils.createStream(ssc,"hdp4.buptnsrc.com:2181","kafka_Test_lwh",Map("test_kafka_flume_hbase_lwh"->2))
//    val lines: DStream[String] = ssc.textFileStream("file:\\D:\\bigdata-project\\spark1\\input2")
    val words = lines.flatMap(_._2.split(" "))
    val wordCounts: DStream[(String, Int)] = words.map(x => (x, 1)).reduceByKey(_ + _)
    //println("lvwenhua"+lines.count())
    wordCounts.print()
    ssc.start()
    ssc.awaitTermination()
  }
}

```

#### 2. sparksql过滤指定数据

```
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{DataFrame, Dataset, SparkSession}

object sparkSqlFilter {
  def main(args: Array[String]): Unit = {

    val spark: SparkSession = SparkSession.builder().master("local[2]").appName("testSQl").getOrCreate()
    val df: DataFrame = spark.read.json("file:\\D:\\bigdata-project\\spark1\\input\\part-r-00004.dms")
    val nums: Long = df.count()
    df.createOrReplaceTempView("filter")
    val sqltest="select * from filter where  rule_100_id!='20190219000100050131' or rule_100_id is null"
    import spark.implicits._

    //val resultframe: DataFrame = spark.sql(sqltest)
    //resultframe.rdd.foreach(println(_))

    val result: RDD[String] = spark.sql(sqltest).toJSON.rdd
    println(result.count())
    println(nums)
    //result.saveAsTextFile("hdfs://node01:8020/lvwenhua/output-sparksql_1/")
   // println(nums2)
    //df.printSchema()

    spark.stop()
  }
}
```

