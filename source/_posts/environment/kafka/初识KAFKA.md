---
title: 初识KAFKA
date: 2020-08-26
categories:
 - kafka
tags:
 - kafka
---


### 初识KAFKA

#### 一、 kafka基本概念介绍

##### 1、kafka扮演的角色
  **消息系统**：系统解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等
  **存储系统**：将数据消息持久化保存到磁盘
  **流式处理平台**：提供完整的流式处理类库，比如窗口、连接、交换和聚合等操作。

##### 2、kafka的专业术语

 **Producer**：生产者，消息发送方，负责将消息发送到kafka队列中

 **Consumer**：消费者，消息接收方，负责接收kafka队列中的数据并进行相应的业务逻辑处理

 **Broker**：服务代理节点，一般情况下一个broker可以看做是一个kafka服务器

##### 3、主题（topic）、分区（Partition）及副本(Replica)
**(1)、** 主题是一个逻辑上的概念，分为多个分区，一个分区只属于单个主题。同一主题下的不同分区消息是不同的。一个分区分为多个副本（大于等于1个）

**(2)、** 一条消息被发送到broker之前，会根据分区规则选择存储到哪个具体的分区。如果分区设置合理，消息会被均匀的分配到不同的分区中。kafka为分区引入了副本机制，增加副本数量可以提升容灾能力。同一分区的副本存储的相同消息，但是同一时刻副本内的消息不一定都是一样的（follwer副本的消息更新一般会稍微滞后于leader），副本间是一主多从关系，leader负责读写请求，follwer只负责与leader的消息进行同步。

**(3)、** 分区中所有副本统称为**AR**。所有与leader副本保持一定程度同步的副本（包括leader）组成**ISR**。与leader信息同步程度滞后太多的副本称为**OSR**。ISR和OSR是AR的一个子集，AR = OSR + ISR。当副本滞后leader太多，该副本会被leader从ISR中剔除。

**(4)、HW ** 标识了一个特别的偏移量，消费者只能拉取到这个值之前的消息（比如消费者能消费到的数据是0到5，那么HW的值就为6），HW的值取决于follwer副本与leader副本的同步情况，HW的值为同步的最小offset加1（若leader的值为15，follwer1位12，follwer2为7，则HW的值为8）。若分区队列中消息数量为0到8，则LEO=9，他标识当前日志文件中下一条待写入消息的offset。分区ISR集合中的每个副本都会维护自身的LEO。


#### 二、 kafka的安装与配置

详情见另外文档（地址待贴出）


#### 三、kafka脚本生产与消费

#####1、创建主题
````
# 进入kafka的根目录下，使用bin目录下的kafka-topics.sh脚本创建主题

bin/kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-demo --replication-factor 3 --partitions 4


--zookeeper           指定了kafka所连接的ZK服务地址
--topic               指定了要创建的主题名称
--replication-factor  指定了副本因子
--create              创建主题的动作命令
````

#####2、查看主题信息

````
# 通过 --describe展示主题的具体信息
bin/kafka-topics.sh --zookeeper localhost:2181/kafka --describe --topic topic-demo
````

#### 3、订阅主题
  创建主题后通过kafka提供的两个脚本来检查是否可以正常地发送和消费数据
````
# 使用kafka-console-consumer.sh来订阅主题

bin/kafka-console-consumer.sh --bootstrap-server localhost:9200 --topic topic-demo


--bootstrap-server    指定了连接的kafka集群地址
--topic               指定了消费者订阅的主题
````

目前该主题暂无信息写入，所以脚本无法消费数据，再打开另一个终端，在kafka根目录下执行消息发送脚本，发送一条Hello Kafka!的信息

````
# 使用kafka-console-producer.sh

bin/kafka-console-producer.sh --broker-list localhost:9092 --topic topic-demo
>Hello kafka!
>


--broker-list      指定了连接kafka集群的地址
--topic            指定了发送消息时的主题
第二行>后面的信息为发送至kafka主题中的信息
````

回到消费kafka的窗口，可以看到刚刚被发送至kafka的信息被获取到


#### 四、java客户端kafka的发送与消费

##### 1、kafka的java客户端需要的maven依赖如下
````
<dependency>
   <groupId>org.apache.kafka</groupId>
   <artifactId>kafka-clients</artifactId>
   <version>2.5.0</version>
</dependency>
````

##### 2、生产者客户端代码

````
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

public class ProducerFastStart{

    public static final String brokerList = "localhost:9092";

    public static final String topic = "topic-demo";

    public static String sendMsg = "this is kafka test message";

    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                brokerList);
        KafkaProducer<String,String> producer
                = new KafkaProducer<String, String>(properties);
        ProducerRecord<String,String> record
                = new ProducerRecord<>(topic,sendMsg);

        try {
            producer.send(record);
        }catch (Exception ex){
            ex.printStackTrace();
        }finally {
            producer.close();
        }
    }

}
````

##### 3、消费者客户端代码
````
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class KafkaConsumerTest {

    public static final String brokerList = "localhost:9092";

    public static final String topic = "topic-demo";

    public static final String groupId = "group.demo";

    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                StringDeserializer.class.getName());
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,brokerList);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG,groupId);

        // 创建一个kafka消费者客户端实例
        KafkaConsumer<String,String> consumer = new KafkaConsumer<String, String>(properties);
        // 订阅主题
        consumer.subscribe(Collections.singletonList(topic));

        while (true){
            ConsumerRecords<String,String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String,String> record:records){
                System.out.println(record.value());
            }
        }
    }


}
````

