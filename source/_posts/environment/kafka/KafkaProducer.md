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

##### 1、默认序列化器

   生产者需要使用序列化器吧对象转换为字节数组才能通过网络发送给Kafka。前面的代码中序列化器使用了客户端自带的**StringSerializer**,除了用于String类型的序列化器，还有ByteArray、ByteBuffer、Bytes、Double、Integer、Long这几种类型，他们都实现了**org.apache.kafka.common.serialization.Serializer**接口，此接口有三个方法，分别是：



````
public void configure(Map<String,?> config,boolean isKey);

public byte[] serialize(String topic,T data);

public void close();
````



 configure() 方法用来配置当前类，在创建KafkaProducer的时候被调用，serialize()方法用来执行序列化操作。close()方法用来执行当前序列化器，一般情况下close()是一个空方法 ，如果实现了这个方法就需要保证这个方法的幂等性，因为这个方法会被KafkaProducer调用多次。

##### 2、自定义序列化器

   假设我们要发送的消息是一个Company对象，对象的属性含有name和address，具体如下（这里使用了lombok插件）：



````
import lombok.*;

@Setter
@Getter
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class Company {
    private String name;
    private String address;
}
````



  再来看一下Company对应的序列化器，CompanySerializer，如下所示：



````
public class CompanySerializer implements Serializer<Company> {
    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {

    }

    @Override
    public byte[] serialize(String s, Company company) {
        if (company == null){
            return null;
        }
        byte[] name,address;
        try {
            if (company.getName() != null){
                name = company.getName().getBytes("UTF-8");
            }else {
                name = new byte[0];
            }

            if (company.getAddress() != null){
                address = company.getAddress().getBytes("UTF-8");
            }else {
                address = new byte[0];
            }
            ByteBuffer buffer = ByteBuffer.allocate(4+4+name.length+address.length);
            buffer.putInt(name.length);
            buffer.put(name);
            buffer.putInt(address.length);
            buffer.put(address);
            return buffer.array();

        }catch (Exception ex){
            ex.printStackTrace();
        }
        return new byte[0];
    }

    @Override
    public byte[] serialize(String topic, Headers headers, Company data) {
        return new byte[0];
    }

    @Override
    public void close() {

    }
}
````



上面的这个自定义序列化器很简单，configure()和close()方法都为空。对应的类反序列化器将在**消费者详解**章节讲到，那么如何使用这个自定义的序列化器呢，下面做个简单的演示。



````
Properties properties = new Properties();
properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,StringSerializer.class.getName());

properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,CompanySerializer.class.getName());

properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);

KafkaProducer<String,Company> kafkaProducer = new KafkaProducer<String, Company>(properties);

Company company = Company.builder().name("hiddenkafka").address("China").build();

ProducerRecord<String,Company> producerRecord
            = new ProducerRecord<String,Company>(topic,company);

producer.send(producerRecord).get();
````



#### 五、分区器

   消息通过send()方法发送broker的过程中，有可能需要经过拦截器、序列化器、分区器的一系列操作之后才不能被真正的发往kafka，消息经过序列化之后就需要确认发往的分区，，如果消息ProducerRecord中指定了partition字段，就不需要分区器的作用，因为partition即是指定发往的分区。若没有指定分区则需要依赖分区器，根据key这个字段来计算分区的值。

   kafka中默认分区器是org.apache.kafka.clients.producer.internals.DefaultPartitioner,它实现了org.apache.kafka.clients.producer.Partitioner接口，这个接口中定义了如下两个方法：

````
public int partition(String topic,Object key,byte[] keyBytes,Object value,byte[] valueByte,Cluster cluster);

public void close();
````

   其中partition()方法用来计算分区号，方法中的参数分别为主题、键、序列化后的键、值、序列化后的值，以及集群的元数据信息。close()方法是空方法。

   partition()方法定义了主要的分区逻辑，如果key不为null，默认分区器会对key进行哈希（采用MurmurHash2算法，具备高运算性能及低碰撞率），最终根据得到的哈希值来计算分区号，拥有相同key的消息会被写入同一个分区，如果key为null，消息会以轮询的方式发往主题的各个可用分区。


   **注意：如果key不为null，那么计算得到的分区号可以是所有分区中的任意一个，，如果key为null且有可用分区时，那么计算得到的分区号仅为可用分区中的任意一个，注意两者之间的差别。**


   在不改变主题分区数量的情况下，key与分区之间的映射关系是保持不变的，不过主题中一旦增加分区就无法保证了。除了可以使用默认分区器之外还能通过实现Partitioner接口来自定义分区器。如下:

````
public class DemoPartitioner implements Partitioner {

    private final AtomicInteger counter = new AtomicInteger(0);

    @Override
    public int partition(String s, Object o, byte[] bytes, Object o1, byte[] bytes1, Cluster cluster) {
        List<PartitionInfo> partitionInfos = cluster.partitionsForTopic(s);
        int num = partitionInfos.size();
        if (bytes == null){
            return  counter.getAndIncrement() % num;
        }else {
            return Utils.toPositive(Utils.murmur2(bytes)) % num;
        }
    }
    @Override
    public void close() {}
    @Override
    public void configure(Map<String, ?> map) {}
}
````

   实现自定义的分区器后，需要通过配置参数**partitioner.class** 来显式指定这个分区器，如下：

````
properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,
                DemoPartitioner.class.getName());
````



#### 六、生产者拦截器

   kafka一共有两种拦截器，生产者拦截器和消费者拦截器。生产者拦截器既可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息内容等，也可以用来在发哦送回调逻辑前做一些定制化的需求，比如进行统计等。


   生产者拦截器主要是自定义实现**org.apache.kafka.clients.producer.ProduerInterceptor**接口，接口中包含三个方法：
````
public ProducerRecord<K,V> onSend(ProducerRecord<K,V> record);

public void onAcknowledgement(RecordMetadata metadata,Exception exception);

public void close();
````

   KafkaProducer在将消息序列化和计算分区之前会调用生产者的拦截器onSend()方法，来对消息进行定制化操作。一般不要对ProducerRecord的topic、key和partition等消息做修改，比如修改key会对分区计算机broker端日志压缩产生影响。


kafkaProducer会在消息被应答之前或者消息发送失败时调用生产者拦截器的onAcknowledgement()方法，优先于用户调用的callback之前调用，这个方法运行在producer的I/O线程中，所以这个方法逻辑越简单越好，否则会影响消息的发送速度。

   close()方法主要用于在关闭拦截器时执行的一些资源的清理工作。这三个方法中抛出的异常都会被记录到日志中，但不会再向上传递。


````
public class ProducerInterceptorPrefix implements ProducerInterceptor<String,String> {

    private volatile long sendSuccess = 0;
    private volatile long sendFailure = 0;

    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> producerRecord) {

        String modifiedValue = "prefix1-"+producerRecord.value();

        return new ProducerRecord<>(producerRecord.topic(),
                producerRecord.partition(),producerRecord.timestamp(),
                producerRecord.key(),modifiedValue,producerRecord.headers());
    }

    @Override
    public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {
        if (e == null){
            sendSuccess ++;
        }else {
            sendFailure ++;
        }
    }

    @Override
    public void close() {
        double successRatio = (double)sendSuccess/(sendFailure+sendSuccess);
        System.out.println("[Info]发送成功率="+String.format("%f",successRatio*100)+"%");
    }

    @Override
    public void configure(Map<String, ?> map) { }
}
````

   实现自定义的**ProducerInterceptorPrefix** 之后，需要在kafkaProducer配置参数 **interceptor.classes** 中指定这个拦截器，此参数默认值为" "，如下：

````
properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
                ProducerInterceptorPrefix.class.getName());
````

发送10条内容为 **kafka test message** 的信息，会发现消费了的信息都变成了 **prefix1-kafka test message**,kafka 中不仅可以指定一个拦截器，还可以指定多个拦截器行成一个拦截链，拦截链的执行顺序会按照参数（**interceptor.classes**）的配置顺序来执行,多个拦截器用逗号隔开。若存在另一个名为 **ProducerInterceptorPrefixPlus** 的拦截器，onSend()方法【配置如下：

````
 @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> producerRecord) {

        String modifiedValue = "prefix2-"+producerRecord.value();

        return new ProducerRecord<>(producerRecord.topic(),
                producerRecord.partition(),producerRecord.timestamp(),
                producerRecord.key(),modifiedValue,producerRecord.headers());
    }
````

   配置生产者拦截器 拦截链：

````
properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
                ProducerInterceptorPrefix.class.getName() +","+
                ProducerInterceptorPrefixPlus.class.getName());
````

   那么最终发送的消息为：**prefix2-prefix1-kafka test message**
   **注意：若拦截器间存在依赖关系，即当前拦截器依赖于上一个拦截器的执行结果，则会存在问题，比如上一个拦截器出现异常，导致执行失败的情况。**



#### 七、生产者整体架构简述

![生产者客户端整体架构](producer01.png)


   整个生产者客户端由2个线程协调运行，分别为主线程和sender线程（发送线程）。在主线程中由 afkaProducer 创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（ RecordAccumulator ，也称为消息收集器〉中。 Sender 线程负责从RecordAccumulator 获取消息并将其发送到 Kafka

   RecrdAccumulator主要用来缓存消息以便 Sender 线程可以批量发送，进 减少网络传输的资源消耗以提升性能 RecordAccumulator 缓存的大 小可以通过生产者客户端参数buffer memory 配置，默认值为 33554432B ，即 32MB 如果生产者发送消息的速度超过发送到服务器的速度 ，则会导致生产者空间不足，这个时候 KafkaProducer的 send（） 方法调用要么被阻塞，要么抛出异常，这个取决于参数 max block ms 的配置，此参数的默认值为 60000,即60秒。

   **注意ProducerBatch不是ProducerRecord，ProducerBatch中包含一至多个ProducerRecord**


#### 八、重要的生产者参数

##### 1、acks
   这个参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的。 acks 是生产者客户端中一个非常重要的参数 ，它涉及消息的可靠性和吞吐量之间的权衡。acks有三种设定值（均为字符串格式）

###### acks = 1
   默认值即为1，生产者消息发送成功后，只要分区的 **leader副本** 消息写入成功，就会给kafaka生产者返回一个成功响应。如果消息发送过程中leader副本挂了或者处于重新选举leader的过程中，则会返回发送错误的响应结果。若在其余副本同步leader副本之前leader崩溃，则还是会存在丢数的情况。这种设置是均衡吞吐量和可靠性的折中方案。

###### acks = 0
   生产者发送kafka之后无需等待kafka回复，若消息发送过程中出现异常导致消息发送失败也无从得知，但在其它配置相同的情况 **acks = 0** 可以达到最大的吞吐量。

###### acks = -1 / acks = all
   生产者在发送消息之后，需要等待所有的ISR副本都成功写入消息后才会返回成功给客户端。在其它环境配置相同的情况下，该配置能达到最高可靠性。但这不是绝对的，当一个分区只有一个副本时，此设置和asks=1的情况是一样的。要获取更高的消息可靠性需要配合 **min.insync.replicas** 等参数的联动。

**注意 acks 参数配置的值是一个字符串类型，而不是整数类型**，如下：

````
properties put("acks"，"1"）；
# 或者
properties.put(ProducerConfig.ACKS_CONFIG,"0"）；

# 若为properties put("acks"，1）；则会报错，因为值的类型应该为String。

````

![acks参数设置错误](producer_error01.png)


##### 2、max.request.size
   这个参数用来限制生产者客户端能发送的消息的最大值，默认值为 1048576B ，即 1MB.不建议读者盲目地增大这个参数的配置值,因为这个参数还涉及一些其它参数的联动。比如 broker 端的message.max bytes 参数。比如将 broker 端的 message max.bytes 参数配置为 10 ，而max.request size参数配置为20, 那么当我们 发送一条大小为 15B 的消息时，生产者客户端就会报出如下的异常

![acks参数设置错误](producer_error02.png)


##### 3、retries retry.backoff.ms
   retries 参数用来配置生产者重试的次数，默认值为0，即在发生异常的时候不进行任何重试动作.
   重试还和另一个参数 retry.backoff.ms 有关，这个参数的默认值为 100 它用来设定两次重试之间的时间间隔，避免无效的频繁重试。
   Kafka 可以保证同一个分区中的消息是有序的。如果生产者按照一定的顺序发送消息，那么这些消息也会顺序地写入分区，进而消费者也可以按照同样的顺序消费它们.


##### 4、compression.type
   这个参数用来指定消息的压缩方式，默认值为“none ”，即默认情况下，消息不会被压缩。该参数还可以配置为gzip、snappy 和z4 对消息进行压缩可以极大地减少网络传输量、降低网络I/O ，从而提高整体的性能 。


##### 5、connections.max.idle. ms
   这个参数用来指定在多久之后关闭限制的连接，默认值是 540000（ ms） ，即9分钟。


##### 6、 linger.ms
   这个参数用来指定生产者发送 ProducerBatch 之前等待更多消息（ ProducerRecord ）加入Producer Batch 时间，默认值为0。


##### 7、 receive.buffe.bytes
   这个参数用来设置 Socket 接收消息缓冲区（ SO_RECBUF ）的大小，默认值为 32768(B))，即32KB。如果设置为 -1 ，则使用操作系统的默认值。如果 Producer与Kafka 处于不同的机房
则可以适地调大这个参数值


##### 8、send.buffer.bytes
   这个参数用来设置 Socket 发送消息缓冲区（CSO_SNDBUF）的大 ，默认值 131072（B）,即128KB 。与 receive.buffer.bytes 参数一样 如果设置为 -1，则使用操作系统的默认值。


##### 9、request.timeout.ms
   这个参数用来配置 Producer 等待请求响应的 长时间，默认值为 30000 （ms ）。请求超时之后可以选择进行重试。注意这个参数需要比broker 端参数 replica.lag.time.max.ms值要大，这样可以减少因客户端重试而引起的消息重复的概率。







