---
title: Kafka生产者详解
date: 2020-07-05
categories:
 - kafka
tags:
 - kafka
---

### Kafka生产者详解

#### 一、客户端开发

##### 1、客户端代码示例

上节对生产者客户端做了基本演示，这节对其进行修改后做具体分析

````
import java.util.Properties;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

public class KafkaProducerTest {
    public static final String brokerList = "localhost:9092";
    public static final String topic = "topic-demo";
    public static String sendMsg = "this is kafka test message";

    public static Properties initConfig(){
        Properties properties = new Properties();
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                brokerList);
        properties.put(ProducerConfig.CLIENT_ID_CONFIG,"producer.client.id.demo");
        return properties;
    }

    public static void main(String[] args) {
        Properties properties = initConfig();
        // 配置生产者客户端参数并创建kafkaProducer实例
        KafkaProducer<String,String> producer = new KafkaProducer<String, String>(properties);
        // 构建需要发送的消息
        ProducerRecord<String,String> record = new ProducerRecord<>(topic,sendMsg);

        try {
            // 发送消息
            producer.send(record);
        }catch (Exception ex){
            ex.printStackTrace();
        }finally {
            // 关闭生产者客户端实例
            producer.close();
        }
    }
}
````

##### 2、ProducerRecord消息对象
````
public class ProducerRecord<K,V>{
    private final String topic;        //主题
    private final Integer partition;   // 分区号
    private final Headers headers;     // 消息头部
    private final K key;               // 键
    private final V value;             // 值
    private final Long timestamp;      // 消息的时间戳
    // 省略其他成员方法和构造方法
}
````

kafkaProducer是线程安全的，可以在多个线程中共享单个的kafkaProducer实例，也可以将kafkaProducer实例进行池化来供其他线程调用。






#### 二、必要的参数配置

在创建真正的生产者实例前需要配置相应的参数，如KafkaProducer中有3个参数是必填的

##### 1、bootstrap.servers
   指定生产者客户端连接kafka需要的broker地址清单，内容格式为 ip1:port1,ip2:port2，可以设置一个或者多个地址，中间用逗号隔开，默认值为“ ”,并不需要设置所有的地址，能根据一个地址找到集群中的其余地址。但是建议设置多个，当一个地址宕机时，能够切换到其余地址上进行连接。

##### 2、key.serializer 和 value.serializer
   boker接收消息必须以字节数组的形式存在。生产者在往broker发送消息前需要将消息中对应的key和value进行相应的序列化操作来转换成字节数组，key.serializer 和 value.serializer两个参数分别指定了序列化的序列化操作器，必须填写全限定名

##### 3、client.id
   用来设置KafkaProducer对应的客户端id，默认值为“ ”，若客户端不设置，则KafkaProducer自动生成一个非空字符串，内容形如“producer-1”，“producer-2”，即字符串“producer-”与数字的拼接。






#### 三、消息的发送

   创建生产者的实例和构建消息后就可以开始发送消息了，发送消息主要有三种模式：**发后即忘**、**同步**、**异步**。
   KafkaProducer的send()方法并非是void类型的，而是Future<RecordMetadata>类型，send()有两个重载方法，分别对应同步和异步

````
# 同步
public Future<RecordMetadata> send(ProducerRecord<K,V> record);

# 异步
public Future<RecordMetadata> send(ProducerRecord<K,V> record,Callback callback);
````

##### 1、即忘型

   前面的代码就即忘型，发送之后不管消息是否到达，大多数情况下这种发送方式没有问题，但发生不可重试异常时会造成消息丢失。这种方式性能最高，可靠性却是最差。


##### 2、同步

   要实现同步的消息发送，可以利用返回的Future对象实现，如下：
````
try {
    // 发送消息
    producer.send(record).get();
}catch (Exception ex){
    ex.printStackTrace();
}finally {
    // 关闭生产者客户端实例
    producer.close();
}
````

   send()方法本身就是异步的，send()方法返回的Future对象可以使调用方稍后获得发送的结果，在执行send()方法后直接链式调用了get()方法来阻塞等待kafka的响应，直到消息发送成功或者发生异常。如下：

````
try {
    // 发送消息
    Future<RecordMetadata> future = producer.send(record);
    RecordMetadata recordMetadata = future.get();
    System.out.println(recordMetadata.topic() + "-" + recordMetadata.partition() + ":" + recordMetadata.offset());

}catch (Exception ex){
    ex.printStackTrace();
}finally {
    // 关闭生产者客户端实例
    producer.close();
}
````

   这样可以获取一个RecordMetadata对象，RecordMetadata对象中包含了消息的一些元数据信息，比如当前消息的主题、分区号、分区中的偏移量（offset）、时间戳等。

   KafkaProducer中一般会发生两种类型的异常：可重试异常和不可重试异常。常见的可重试异常有：网络异常、分区leader副本不可用异常（发生在leader副本下线而新leader未被选举出来的时候）。不可重试异常如：RecordTooLargeException（发送消息太大），无法重试，会直接抛出异常。

   对于可重试异常，如果配置了**retries**参数，只要在规定的重试次数内自行恢复就不会抛出异常，默认为0,配置参考如下：
````
properties.put(ProducerConfig.RETRIES_CONFIG,5);
````


##### 3、异步

   异步的发送方式一般是send()方法里指定一个Callback回调函数，kafka在返回响应时调用该函数来实现异步发送消息的确认，异步发送示例如下：
````
producer.send(record, new Callback() {
                @Override
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if (e !== null){
                        e.printStackTrace();
                    }else {
                        System.out.println(recordMetadata.topic() + "-" + recordMetadata.partition() + ":" + recordMetadata.offset());
                    }
                }
});
````

   这只是一个简单的调用demo，实际情况中对于异常肯定会进行更加妥善的处理，消息发sing成功时，异常为空而recordMetadata不为空，消息发送异常时，recordMetadata为空而异常不为空。

````
producer.send(record1, callback1);

producer.send(record2, callback2);
````

   对于同一个分区而言，如果record1比record2先发送，那么可以保证callback1比callback2先调用，也就是说回调函数的调用可以保证分区有序。

````
try {
    int i = 0;
    while (i < 100){
        ProducerRecord<String,String> recordDemo = new ProducerRecord<>(topic,"test"+i);
        producer.send(recordDemo).get();
    }
}catch (Exception ex){
    ex.printStackTrace();
}finally {
    // 关闭生产者客户端实例
    producer.close();
}
````

  close()方法会阻塞等待之前所有的发送请求完成后再关闭KafakProducer。除此之外KafakProducer还提供了一个带超时时间的close()方法。如下：
````
public void close(long timeout, TimeUnit timeUnit);
````

   如果调用了带超时时间timeout的close()方法，那么只会在等待timeout时间内来完成尚未处理完成的请求处理，然后强行退出，实际使用中，一般使用无参方法。


#### 四、序列化

   生产者需要使用序列化器吧对象转换为字节数组才能通过网络发送给Kafka。前面的代码中序列化器使用了客户端自带的**StringSerializer**,除了用于String类型的序列化器，还有ByteArray、ByteBuffer、Bytes、Double、Integer、Long这几种类型，他们都实现了**org.apache.kafka.common.serialization.Serializer**接口，此接口有三个方法，分别是：

````
public void configure(Map<String,?> config,boolean isKey);

public byte[] serialize(String topic,T data);

public void close();
````

   config() 方法用来配置当前类，serialize()方法用来执行序列化操作。close()方法用来执行当前序列化器，一般情况下close()是一个空方法















