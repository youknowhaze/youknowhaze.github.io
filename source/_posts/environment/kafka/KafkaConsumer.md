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


#### 三、必要的参数配置

##### 1、bootstrap.servers

   用来指定连接 Kafka 集群所需的 broker 地址清单，具体内容形式为hostl:portl,host2 post ，可以设置一个或多个地址，中间用逗号隔开。


##### 2、group.id

   消费者组默认为空，若为空启动KafkaConsumer，则会抛出**InvalidGroupldException**异常


##### 3、key.deserializer 和 value.deserializer

   分别指定key和value的反序列化器，与生产者客户端的key.serializer 和 value.serializer 对应。这里必须填写反序列化器类的全限定名，如：org.apache.kafka.common.serialization.StringDeserializer。单单只指定StringDeserializer是错误的。


##### 4、client.id

   KafkaConsumer对应的客户端id，默认为空，若不进行设置，则KafkaConsumer会自动生成一个非空字符串，内容形式如 “consumer-1”、“consumer-2”，即 **“consumer-”** 与数字的拼接。


##### 5、ConsumerConfig

   由于KafkaConsumer的配置参数很容易写错，故与生产者客户端一样引入了配置参数类 **ConsumerConfig**，如修改初始化配置方法如下：

````

public static Properties initConfig() {
        Properties props = new Properties();
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                     StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
        			StringDeserializer.class.getName());
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,brokerList);
        props.put(ConsumerConfig.GROUP_ID_CONFIG,groupid);
        props.put(ConsumerConfig.CLIENT_ID_CONFIG,"consumer.client.id.demo");
        return props;
        KafkaConsumer<String,String> consumer= new KafkaConsumer<> (props );
    }
````


#### 四、订阅主题与分区

   一个消费者可以订阅一个或多个主题，支持以集合的形式订阅多个主题，也支持以正则表达式的形式订阅特定的主题及特定分区的订阅。三种订阅方式分别代表了三种不同的订阅状态：集合订阅（AUTO_TOPICS）、正则订阅（AUTO_PATTERN）和指定分区订阅（USER_ASSIGNED）,若没有订阅则订阅状态为NONE。这三种订阅状态是互斥的，一个消费者只能使用一种订阅方式，若使用多个则会爆出IllegalStateException异常。
````
java.lang.IllegalStateException:Subscription to topics partitions and pattern are mutually exclusive.
````

##### 1、集合订阅

如果消费者前后三次订阅了不同的主题，那么以最后一次订阅为准，如下代码所示，最后生效的订阅主题为 **topic2**
````
consumer.subscribe(Arrays.asList(topic10，topic11));
consumer.subscribe(Arrays.asList(topic1));
consumer.subscribe(Arrays.asList(topic2));
````

##### 2、正则表达式订阅

如果消费者采用正则表达式的方式订阅了主题，则当新建一个主题，且这个主题符合消费者的正则订阅时，消费者将可以消费到新增的主题中的内容，正则订阅方式如下：

````
consumer.subscribe (Pattern.compile("topicdemo_*"));
````


##### 3、主题特定分区订阅

````
public void assign(Collection<TopicPartition> partitions);

// 说明一下TopicPartition类，含有partition和topic两个属性，分别为分区和主题
````

现在模拟订阅topic-demo主题的0分区，如下代码所示：

````
// 这里也是以集合订阅形式订阅的
consumer.assign(Arrays.asList(new TopicPartition(”topic-demo”,0))) ;
````

但是，若是我们不知道kafka的服务器端配置，怎么知道主题有多少个分区呢？KafkaConsumer提供了partitionsFor()方法用来查询特定主题的元数据信息。

````
// 这里返回的list集合大小，主题有多少分区，集合大小就为多少
public List<PartitionInfo> partitionsFor(String topic);

//Partitionlnfo 为主题的分区元数据信息。结构如下：
public class Partitioninfo {
	private final String topic ;      //主题名称
	private final int partition ;     //分区编号
	private final Node leader;        //leader 副本所在的位置
	private final Node[] replicas;    //分区的 AR集合
	private final Node[] inSyncReplicas;    // ISR集合
	private final Node[] offlineReplicas;   // OSR 集合
	／／这里省略了构造函数、属性提取 toStr ng 等方法
}
````

这里使用指定订阅方式来量topic-demo主题的所有分区进行订阅，代码如下：

````
List<TopicPartition> topicPartitionList = new ArrayList<>();
// 得到主题的各个分区元数据信息
List<PartitionInfo> partitionInfoList = consumer.partitionsFor("topic-demo");
// 得到需要订阅的集合
partitionInfoList.stream().forEach(partitionInfo -> {
            topicPartitionList.add(
                    new TopicPartition(partitionInfo.topic(),partitionInfo.partition()));
        });
// 进行指定分区的集合订阅
consumer.assign(topicPartitionList);
````


##### 4、取消订阅

有订阅就有取消订阅，与订阅相似，取消订阅的方式也支持集合取消和正则取消，当然也支持指定分区订阅取消，通过unsubscribe()方法实现取消订阅。示例如下：
````
// 我的理解是取消已订阅的所有主题/分区，不需要指定参数，是无参方法
consumer.unsubscribe();
````

取消订阅后，若不订阅任何主题或分区便执行消费程序，则会抛出IllegalStateException异常，通过subscribe() 方法订阅主题具有消费者自动再均衡的功能 ，在朵儿消费者的情况下可以根据分区分配策略来自动分配各个消费者与分区的关系。当消费组内的消费者增加或减少时，分区分配关系会自动调整，以实现消费负载均衡及故障自动转移。 而通过assign()方法订阅分区时， 是不具备消费者自动均衡的功能的。



#### 五、反序列化

##### 1、反序列化器介绍

KafkaProducer通过实现Serializer接口 实现自定义序列化器的操作，为了能够将数据成功反序列化，KafkaConsumer可以通过实现Deserializer 反序列化接口来自定义反序列化器，与自定义的序列化器进行对应。与Serializer相似的是，Deserializer也有三个方法，如下：

````
// 配置当前类
public void configure(Map<String, ?> cofigs,boolean isKey) :
// 用来执行反序列化。如果 data为null，那么处理的时候直接返回 null 而不是抛出一个异常。
public byte[] serialize(String topic,T data);
//  用来关闭当前序列化器。
public void close();
````

kafka 自带的反序列化器 **StringDeserializer** 实现如下：

````
public class StringDeserializer implements Deserializer<String> {
        private String encoding ="UTF8";
        @Override
        public void configure(Map<String, ?> configs, boolean isKey) {
            String propertyName = isKey ? "key.deserializer.encoding":
                    "value.deserializer.encoding";
            Object encodingValue = configs.get(propertyName) ;
            if (encodingValue == null)
            encodingValue = configs.get ("deserializer.encoding");
            if (encodingValue != null && encodingValue instanceof String)
            encoding = (String) encodingValue ;
        }

        @Override
        public String deserialize(String topic, byte [] data){
                try {
                    if (data == null)
                        return null;
                    else
                        return new String(data, encoding);
                } catch (UnsupportedEncodingException e){
                    throw new SerializationException("Error when"
                            + " deserializing byte[] to string due to " +
                "unsupported encoding "+encoding);
                }
        }

        @Override
        public void close () {}
}
````

configure()方法中有三个参数，**key.deserializer.encoding**、**value.deserializer.encoding** 和 **deserializer.encoding** 用来配置反序列化的编码类型，这三个参数都是用户自定义的参数类型


##### 2、自定义反序列化器的代码实现

前面章节说到了生产者的自定义序列化器，这里演示一下与之对应的反序列化器，代码如下：

````
public class CompanyDeserializer implements Deserializer<Company> {
    @Override
    public void configure(Map<String, ?> configs, boolean isKey) { }

    @Override
    public Company deserialize(String s, byte[] bytes) {
        if (bytes == null) {
            return null;
        }

        if (bytes.length < 8) {
            throw new SerializationException("Size of data received");
        }
        ByteBuffer buffer = ByteBuffer.wrap(bytes);
        int nameLen, addressLen;
        String name, address;

        nameLen = buffer.getInt();
        byte[] nameBytes = new byte[nameLen];
        buffer.get(nameBytes);
        addressLen = buffer.getInt();

        byte[] addressBytes = new byte[addressLen];
        buffer.get(addressBytes);

        try {
            name = new String(nameBytes, "UTF-8");
            address = new String(addressBytes, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            throw new SerializationException("Error occur when deserializing1");
        }
        return new Company(name, address);
    }

    @Override
    public Company deserialize(String topic, Headers headers, byte[] data) {
        return null;
    }

    @Override
    public void close() {}
}
````

使用配置如下：
````
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                CompanyDeserializer.class.getName());
````

**注意：若无特殊需要，不建议使用自定义的序列化器和反序列化器，因为这样会增加消费者和生产者之间的耦合度，还要面对序列化和反序列化的兼容性**



#### 六、消息消费

   kafka中消息的消费是基于拉模式的，消息的消费一般有两种模式：推模式和拉模式，推模式是服务端将消息主动推送给消费者，拉模式是消费者向服务端发起请求获取数据。kafka中的消息消费就是一个不断轮询的过程，消费者通过重复的调通poll()方法，得到主题分区中的数据，poll()方法定义如下：
````
// timeout为超时时间参数，在消费者的缓冲区里没有可用数据时会发生阻塞
public ConsumerRecords<K, V> poll(f nal long timeout)
````
