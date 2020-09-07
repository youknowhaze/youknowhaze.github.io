---
title: Kafka消费者详解
date: 2020-09-07
categories:
 - kafka
tags:
 - kafka
 - consumer
---


### Kafka 消费者

#### 一、消费者与消费者组
   消费者（ Consumer ）负责订阅 Kafka 中的主题（ Topic ），并且从订阅的主题上拉取消息与其他一些消息中间件不同的是：在 Kafka 的消费理念中还有一层消费组（ Consumer Group)的概念，每个消费者都有一个对应的消费组。当消息发布到主题后，只会被投递给订阅它的每个消费组中的一个消费者。

![多个消费者组](consumer01.png)

   如上图所示，主题存在四个分区（p0/p1/p2/p3），有两个消费者组A和B，A中的四个消费者每人消费了一个分区，而B中的消费者每人消费了2个分区，换而言之，kafka中的一个分区只允许被一个消费者消费。


![消费者组中只有一个消费者](consumer02.png)

   再如上图中，一个消费者组中只含有一个消费者，则这个消费者将消费所有的分区数据。组中此时新增一个消费者，则此时新增加的消费者会被分配尽可能平均的分区进行消费，如下图所示：

![消费者组中新增一个消费者](consumer03.png)

   对消费者组依次增加消费者数量，直到超过分区数为止，如下图所示，当消费者数量超过分区数时，会有部分消费者无法消费分区数据，因为一个分区只能背一个消费者消费。

![消费者组中新增一个消费者](consumer04.png)

   以上分配逻辑都是基于默认的分区分配策略进行分析的，可以通过消费者客户端参数
partition.assignment.strategy 来设置消费者与订阅主题之间的分区分配策略。



#### 二、消费者客户端开发

````
public class KafkaConsumerAnalysis {

    public static final String brokerList = "localhost:9092";
    public static final String topic = "topic-demo";
    public static final String groupid = "group.demo";

    public static final AtomicBoolean isRunning =new AtomicBoolean(true);

    public static Properties initConfig() {
        Properties props = new Properties();
        props.put("key.deserializer",
                "org.apache.kafka.common.seriali zation.StringDeserializer");
        props.put("value.deserializer",
                "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("bootstrap.servers", brokerList);
        props.put("group.id", groupid);
        props.put("client.id" ,"consumer.client.id.demo");
        return props;
    }

    public static void main (String [] args) {
        Properties props = initConfig();
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList(topic));
        try {
            while (isRunning.get()) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("topic = " + record.topic() + ",partition = " + record.partition() + ",offset = " + record.offset());
                    System.out.println("key = " + record.key() + ", value = " + record.value());

                }
            }
        }catch (Exception ex){
            ex.printStackTrace();
        }finally {
            consumer.close();
        }
    }
 }
````



































