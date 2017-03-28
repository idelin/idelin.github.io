#java8下spark-streaming结合kafka编程（spark 2.0 & kafka 0.10）
前面有说道[spark-streaming的简单demo](http://blog.csdn.net/jacklin929/article/details/53689365)，也有说到[kafka成功跑通的例子](http://blog.csdn.net/jacklin929/article/details/53767622)，这里就结合二者，也是常用的使用之一。

1.相关组件版本
首先确认版本，因为跟之前的版本有些不一样，所以才有必要记录下，另外仍然没有使用scala,使用java8,spark 2.0.0,kafka 0.10。

2.引入maven包
网上找了一些结合的例子，但是跟我当前版本不一样，所以根本就成功不了，所以探究了下，列出引入包。
```xml
<dependency>
	  <groupId>org.apache.spark</groupId>
	  <artifactId>spark-streaming-kafka-0-10_2.11</artifactId>
	  <version>2.0.0</version>
</dependency>
```
网上能找到的不带kafka版本号的包最新是1.6.3，我试过，已经无法在spark2下成功运行了，所以找到的是对应kafka0.10的版本，注意spark2.0的scala版本已经是2.11，所以包括之前必须后面跟2.11，表示scala版本。

3.SparkSteamingKafka类
需要注意的是引入的包路径是org.apache.spark.streaming.kafka010.xxx，所以这里把import也放进来了。其他直接看注释。
```java
import java.util.Arrays;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.streaming.Durations;
import org.apache.spark.streaming.api.java.JavaInputDStream;
import org.apache.spark.streaming.api.java.JavaPairDStream;
import org.apache.spark.streaming.api.java.JavaStreamingContext;
import org.apache.spark.streaming.kafka010.ConsumerStrategies;
import org.apache.spark.streaming.kafka010.KafkaUtils;
import org.apache.spark.streaming.kafka010.LocationStrategies;

import scala.Tuple2;

public class SparkSteamingKafka {
	public static void main(String[] args) throws InterruptedException {
		String brokers = "master2:6667";
	    String topics = "topic1";
		SparkConf conf = new SparkConf().setMaster("local[2]").setAppName("streaming word count");
		JavaSparkContext sc = new JavaSparkContext(conf);
		sc.setLogLevel("WARN");
		JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(1));
		
		Collection<String> topicsSet = new HashSet<>(Arrays.asList(topics.split(",")));
	    //kafka相关参数，必要！缺了会报错
		Map<String, Object> kafkaParams = new HashMap<>();
	    kafkaParams.put("metadata.broker.list", brokers) ;
	    kafkaParams.put("bootstrap.servers", brokers);
	    kafkaParams.put("group.id", "group1");
	    kafkaParams.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
	    kafkaParams.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
	    kafkaParams.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
	    //Topic分区
	    Map<TopicPartition, Long> offsets = new HashMap<>();
	    offsets.put(new TopicPartition("topic1", 0), 2L); 
	    //通过KafkaUtils.createDirectStream(...)获得kafka数据，kafka相关参数由kafkaParams指定
	    JavaInputDStream<ConsumerRecord<Object,Object>> lines = KafkaUtils.createDirectStream(
	            ssc,
	            LocationStrategies.PreferConsistent(),
	            ConsumerStrategies.Subscribe(topicsSet, kafkaParams, offsets)
	        );
	    //这里就跟之前的demo一样了，只是需要注意这边的lines里的参数本身是个ConsumerRecord对象
		JavaPairDStream<String, Integer> counts = 
				lines.flatMap(x -> Arrays.asList(x.value().toString().split(" ")).iterator())
				.mapToPair(x -> new Tuple2<String, Integer>(x, 1))
				.reduceByKey((x, y) -> x + y);	
		counts.print();
//	可以打印所有信息，看下ConsumerRecord的结构
//	    lines.foreachRDD(rdd -> {
//	        rdd.foreach(x -> {
//	          System.out.println(x);
//	        });
//	      });
		ssc.start();
		ssc.awaitTermination();
		ssc.close();
	}
}
```

4.运行测试
这里使用上一篇kafka初探里写的producer类，put数据到kafka服务端，我这是master2节点上部署的kafka，本地测试跑spark2。
```
UserKafkaProducer producerThread = new UserKafkaProducer(KafkaProperties.topic);
producerThread.start();
```
再运行3里的SparkSteamingKafka类，可以看到已经成功。
![运行生产者类](http://img.blog.csdn.net/20161226195407346?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamFja2xpbjkyOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![运行spark充当消费者](http://img.blog.csdn.net/20161226195350341?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamFja2xpbjkyOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

[![这里写图片描述](http://img.blog.csdn.net/20161227103155846?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamFja2xpbjkyOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](weibo.com/234601929)