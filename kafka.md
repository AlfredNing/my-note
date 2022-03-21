# 概述

1. 消息模式有很多可选项，kafka较为普遍选取Apache Avro 支持强类型和模式进化，版本向前兼容，向后兼容
2. kafka通过**主题**分类，主题被划分为多个**分区**，**一个分区就是一个提交日志。消息以追加的方式写入分区，然**
   **后以先入先出的顺序读取**。 分区内有序，分区间无序
3. kafka通过生产者生产消息，其他发布订阅系统中，**称为发布者或者写入者**，消息写入通过消息键和分区器来实现
4. 消费者组内消费者消费数据，一个分区只能被同一个消费组里面的消费者消费，通过偏移量定位消费的位置
5. broker是一个独立的kafka服务器，接受生产者消息。多个broker形成broker集群，每个集群中有一个broker充当集群控制器角色，控制器负责管理工作，包括将分区分配给broker 和监控broker
6. 保留消息，broker默认的消息策略为七天或者达到一定的大小字节数，达到之后，旧消息就会被删除。注意：**每个主题可以设置单独的保留规则，满足不同消费者的需求**。

## 特性

- 多个生产者
- 多个消费者
- 基于磁盘的数据存储
- 伸缩性
- 高性能

# 生产者

![	](G:\note\draw_save\kafka生产者.jpg)

## 发送流程

1. 创建ProducerRecord对象，包含主题和发送内容，可选指定键和分区
2. 键值对象序列化成字节数组
3. 数据发送给分区器，没有指定分区情况下，根据键选择一个分区
4. 分区选择完毕，这条消息添加到一个批次当中，这个批次的**所有记录归属于相同主题和分区**
5. 独立线程将记录批次发送到相应的broker上
6. broker返回响应
   1. 消息写入broker成功，返回RecordMetaData对象，包含主题和分区，以及记录在分区的偏移量
   2. 消息写入broker失败，返回错误，生产者根据配置进行重试，重试次数达到仍然失败，返回错误信息

## 发送消息的方式

- 发送忘记：消息发送给服务器，但并不关心它是否正常到达，错误自动重试
- 同步发送：send() 方法发送消息，它会返回一个Future 对象，调用get() 方法进行等待，判断是否发送成功
- 异步发送：send() 方法，并指定一个回调函数，服务器在返回响应时调用该函数

## 生产者发送消息-Coding

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class MessageProducer {
    public static void main(String[] args) {
        // 生产者配置
        Properties producerConfig = new Properties();
        // bootstrap.servers
        producerConfig.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-node01:9092,kafka-node02:9092,kafka-node03:9092,");
        // 设置key的序列化器
        producerConfig.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        // 设置value的序列化器
        producerConfig.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        // 创建kafka生产者
        KafkaProducer<String, String> producer = new KafkaProducer<String, String>(producerConfig);
        String topic = "test-vip";
        ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(topic, "messageKey", "messageValue");

        // 同步发送
//        try {
//            // 如果服务器返回错误，get()抛出异常，发送成功返回RecordMetadata 对象
//            producer.send(producerRecord).get();
//        } catch (Exception e) {
//            e.printStackTrace();
//        }

        // 异步发送
        try {
            // 如果服务器返回错误，get()抛出异常，发送成功返回RecordMetadata 对象
            producer.send(producerRecord, new ProducerCallBack());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

class ProducerCallBack implements Callback {
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        // 如果服务器返回错误，会抛出非空异常
        if(e != null){
            e.printStackTrace();
        }
    }
}
```

## 生产者配置参数

- acks: 指定必须要有多少个分区副本收到消息，生产者才会认为消息写入成功
  - acks = 0: 生产者在成功写入消息之前不会等待任何来自服务器的响应
  - acks = 1: 集群首领节点收到消息，返回相应给生产者确认消息发送成功，否则收到错误响应，生产者重新发送消息
  - acks = all: 所有复制节点全部收到消息，集群响应生产者
- buffer.memory：设置生产者内存缓冲区的大小，生产者用它缓冲要发送到服务器的消息，如果生产者发送消息速度超过发送到服务器的速度，导致生产者空间不足。**send() 方法调用要么被阻塞，要么抛出异常**取决于max.block.ms：阻塞时长
- compression.type：默认情况下，消息发送时不会被压缩。该参数可以设置为snappy、gzip 或lz4，它指定了
  消息被发送给broker 之前使用哪一种压缩算法进行压缩。可选值：snappy,gazip等
- retries: 生产者可以重发消息的次数，如果达到这个次数，生产者会放弃重试并返回错误;。默认情况下，生产者会在每次重试之间等待100ms,取决于retry.backoff.ms 参数
- batch.size： 指定一个批次使用的内存大小 字节数
- linger.ms： 参数指定了生产者在发送批次之前等待更多消息加入批次的时间
- client.id: 任意的字符串，服务器会用它来识别消息的来源

- max.in.flight.requests.per.connection: 指定了生产者在收到服务器响应之前可以发送多少个消息，值越高，内存占据越多**设为1 可以保证消息是按照发送的顺序写入服务器的，即使生了重试。**
- timeout.ms： 指定broker 等待同步副本返回消息确认的时间，与asks 的配置相匹配——如果在指定时间内没有收到同步副本的确认，那么broker 就会返回一个错误
- request.timeout.ms: 生产者在发送数据时等待服务器返回响应的时间
- metadata.fetch.timeout.ms： 指定了生产者在获取元数据（时等待服务器返回响应的时间。**如果等待响应超时，那么生产者要么重试发送数据，要么返回一个错误抛出异常或执行回调**
- max.block.ms: 调用send() 方法或使用partitionsFor() 方法获取元数据时生产者的阻塞时间，超出该时间，生产者抛出异常
- max.request.size： 控制生产者发送的请求大小。它可以指能发送的单个消息的最大值，也可以指
  单个请求里所有消息总的大小
- message.max.bytes: **该参数由broker设置**，与max.request.size要匹配
- receive.buffer.bytes；send.buffer.bytes：TCP socket 接收和发送数据包的缓冲区大小，设为-1，就使用操作系统的默认值

###  发送消息的顺序问题

kafka保证单个分区内有序，但如果retries 设为非零整数max.in.flight.requests.per.connection设为比1 大的数，那么就由可能发生顺序相反。保证严格顺序下，可以设置：retries 设为0；max.in.flight.requests.per.connection = 1

## 序列化器

- Avro序列化（常用）
- json（常用）
- 自定义（不推荐）

## 分区

### 默认分区器

1. 消息键可以设置为**null**, 如果使用默认的分区器，记录将随机的发送到主题内各个可用的分区。分区使用**轮询（Round Robin）**算法将消息均衡地分布到各个分区上
2. 键不为**null**,根据散列值将消息映射到特定的分区，同一个键总是被映射到同一个分区

### 自定义分区器

```java
public class CustomPartitioner implements Partitioner {

    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        Integer numPartitions = cluster.partitionCountForTopic(topic);
        // 分区逻辑
        if ((keyBytes == null) || (!(key instanceof String))){
            throw new InvalidRecordException("expect the record has the key");
        }
        if("svip".equals(key.toString())){
            // 返回分区编号
            return numPartitions - 1;
        }
        return (Math.abs(Utils.murmur2(keyBytes) % (numPartitions - 2)));
    }

    public void close() {

    }

    public void configure(Map<String, ?> map) {

    }
}
```

配置生产者调用：

![image-20220313185312638](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220313185312638.png)

# 消费者

>Kafka 消费者从属于消费者群组。一个群组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息。同一个主题下的分区只能被同一个消费组下的一个消费者消费，不同消费组之间互不影响。

## 分区重分配

在主题添加分区，消费者突然下线，离开群组，发生分区重分配

## 再均衡

分区的所有权从一个消费者转移到另一个消费者，这样的行为称为再平衡。

再均衡发生时，消费者无法读取消息，**避免不必要的在均衡**。

### 触发时机

1. 消费者向组协调器的broker发送心跳来维持群组的从属关系和分区的所有权关系，只要以正常的时间发送心跳，则认为是活跃的，如果超时，则会触发一次再均衡	
2. 一个消费者发生崩溃，并停止读取消息，群组协调器会等待几秒钟，确认它死亡了才触发再均衡
3. 在清理消费者时，消费者会通知协调器它将要离开群组，协调器会立即触发一次再均衡

### 分配分区

消费者加入群组，向组协调器发送JoinGroup请求，第一个加入的消费者成为”群主“，群主从协调器那里获得群组的成员列表，并负责给每一个消费者分配分区。

## 消费者消费消息-Coding

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class MessageConsumer {
    public static void main(String[] args) {
        Properties consumerConfig = new Properties();
        // bootstrap.servers
        consumerConfig.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-node01:9092,kafka-node02:9092,kafka-node03:9092,");
        // 设置key的序列化器
        consumerConfig.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // 设置value的序列化器
        consumerConfig.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // group id
        consumerConfig.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group-test");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
        consumer.subscribe(Collections.singletonList("test-vip"));

        while (true){
            // 控制poll() 方法的超时时间，该参数被设为0，poll() 会立即返回，否则它会在指定的毫秒数内一直等待broker 返回数据
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100L));
            for (ConsumerRecord<String, String> record : records) {
                String data = String.format("topic = %s, partition = %s, offset = %d, key = %s, value = %s\n",
                        record.topic(), record.partition(), record.offset(), record.key(), record.value());
                System.out.println(data);
            }
        }
    }
}
```

## 消费者参数配置

- fetch.min.bytes： 指定了消费者从服务器获取记录的最小字节数，broker 在收到消费者的数据请求时，
  如果可用的数据量小于fetch.min.bytes 指定的大小，那么它会等到有足够的可用数据时才把它返回给消费者
- fetch.max.wait.ms：通过fetch.min.bytes 告诉Kafka，等到有足够的数据时才把它返回给消费者，默认是500ms，与fetch.min.bytes相互影响
- max.partition.fetch.bytes：指定了服务器从每个分区里返回给消费者的最大字节数。它的默认值是1MB。
- session.timeout.ms：指定了消费者在被认为死亡之前可以与服务器断开连接的时间，如果消费者没有在session.timeout.ms 指定的时间内发送心跳给群组协调器，就被认为已经死亡，协调器就会触发再均衡，把它的分区分配给群组里的其他消费者
- heartbeat.interval.ms： 指定了poll() 方法向协调器发送心跳的频率，session.timeout.ms 则指定了消费者可以多久不发送心跳。**所以，一般需要同时修改这两个属性**

- auto.offset.reset: 指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下（因消费者长
  时间失效，包含偏移量的记录已经过时并被删除）如何处理。可选值：latest,earliest
- enable.auto.commit: 是否自动提交偏移量
- auto.commit.interval.ms: 自动提交频率
- partition.assignment.strategy： 分区分配策略
- client.id：该属性可以是任意字符串，broker 用它来标识从客户端发送过来的消息，通常被用在日志、
  度量指标和配额里
- max.poll.records：该属性用于控制单次调用poll() 方法能够返回的记录数量
- receive.buffer.bytes 和 send.buffer.bytes: socket 在读写数据时用到的TCP 缓冲区也可以设置大小。如果它们被设为-1，就使用操作系统的默认值

## 提交和偏移量

**提交：消费者更新当前位置的操作**

1. __consumer_offset主题保存着每个分区的偏移量	
2. 如果提交的偏移量小于客户端处理的最后一个消息的偏移量，那么处于两个偏移量之间的消息就会被重复处理
3. 如果提交的偏移量大于客户端处理的最后一个消息的偏移量，那么处于两个偏移量之间的消息将会丢失

## 提交方式

1. 自动提交：

   - enable.auto.commit ： true
   - auto.commit.interval.ms: 提交时间间隔
   - 自动提交：每次调用轮询方法会把上一次调用返回的偏移量提交上去

   **问题：**

   1. 重复消费

      假设每5s提交一次，在最近一次提交之后的3s中发生再均衡，再均衡之后，从最后一次提交的偏移量拉取，就会被重复处理

2. 同步提交

   - auto.commit.offset 设为false
   - 调用commitSync() 方法提交当前批次最新的偏移量

   问题：再均衡的情况下也会发生重复消费

3. 异步提交

   - auto.commit.offset 设为false
   - 调用commitAsync()

**在成功提交或碰到无法恢复的错误之前，commitSync() 会一直重试，但是commitAsync()不会**

4. 组合提交

## 提交代码-Coding

```java
public class MessageCommitConsumer {
    public static void main(String[] args) {
        Properties consumerConfig = new Properties();
        // bootstrap.servers
        consumerConfig.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-node01:9092,kafka-node02:9092,kafka-node03:9092,");
        // 设置key的序列化器
        consumerConfig.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // 设置value的序列化器
        consumerConfig.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // group id
        consumerConfig.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group-test");
        // 关闭自动提交
        consumerConfig.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
        consumer.subscribe(Collections.singletonList("test-vip"));
        // ================同步提交==========================
        executeCommitSync(consumer);
        // ================异步提交==========================
        executeCommitAsync(consumer);
        // ================组合提交==========================
        executeCombinationCommit(consumer);
        // ================提交特定偏移量==========================
        executeSpecialCommit(consumer);


    }

    private static void executeSpecialCommit(KafkaConsumer<String, String> consumer) {
        HashMap<TopicPartition, OffsetAndMetadata> tp = new HashMap<>();
        int count = 0;
        while (true) {
            // 控制poll() 方法的超时时间，该参数被设为0，poll() 会立即返回，否则它会在指定的毫秒数内一直等待broker 返回数据
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100L));
            for (ConsumerRecord<String, String> record : records) {
                tp.put(new TopicPartition(record.topic(), record.partition()), new OffsetAndMetadata(record.offset() + 1, "metadata"));
                String data = String.format("topic = %s, partition = %s, offset = %d, key = %s, value = %s\n",
                        record.topic(), record.partition(), record.offset(), record.key(), record.value());
                System.out.println(data);
            }
            // 正常，异步提交
            if (count % 1000 == 0) {
                // 中间提交偏移量
                consumer.commitAsync(tp,null);
            }
            count++;
        }
    }

    private static void executeCombinationCommit(KafkaConsumer<String, String> consumer) {
        try {
            while (true) {
                // 控制poll() 方法的超时时间，该参数被设为0，poll() 会立即返回，否则它会在指定的毫秒数内一直等待broker 返回数据
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100L));
                for (ConsumerRecord<String, String> record : records) {
                    String data = String.format("topic = %s, partition = %s, offset = %d, key = %s, value = %s\n",
                            record.topic(), record.partition(), record.offset(), record.key(), record.value());

                    System.out.println(data);
                }
                // 正常，异步提交
                consumer.commitAsync();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                // 重试
                consumer.commitSync();
            } finally {
                consumer.close();
            }
        }
    }

    private static void executeCommitAsync(KafkaConsumer<String, String> consumer) {
        while (true) {
            // 控制poll() 方法的超时时间，该参数被设为0，poll() 会立即返回，否则它会在指定的毫秒数内一直等待broker 返回数据
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100L));
            for (ConsumerRecord<String, String> record : records) {
                String data = String.format("topic = %s, partition = %s, offset = %d, key = %s, value = %s\n",
                        record.topic(), record.partition(), record.offset(), record.key(), record.value());
                System.out.println(data);
            }
            try {
                // 带有回调
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        exception.printStackTrace();
                    }
                });
            } catch (CommitFailedException e) {
                e.printStackTrace();
            }
        }
    }

    private static void executeCommitSync(KafkaConsumer<String, String> consumer) {
        while (true) {
            // 控制poll() 方法的超时时间，该参数被设为0，poll() 会立即返回，否则它会在指定的毫秒数内一直等待broker 返回数据
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100L));
            for (ConsumerRecord<String, String> record : records) {
                String data = String.format("topic = %s, partition = %s, offset = %d, key = %s, value = %s\n",
                        record.topic(), record.partition(), record.offset(), record.key(), record.value());
                System.out.println(data);
            }
            try {
                consumer.commitSync();
            } catch (CommitFailedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## 再均衡监听器

当消费者重分配，分区再均衡之前，利用监听器，做一些清理工作

```java
consumer.subscribe(Collections.singletonList("test-vip"), new ConsumerRebalanceListener() {
    // 再均衡开始之前和消费者停止读取消息之后被调用
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // 提交已经处理过的偏移量，而不是最新偏移量
    }

    // 在重新分配分区之后和消费者开始读取消息之前被调用
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {

    }
});
```

## 从特定位置开始处理记录

应用： 从处理消费数据到数据库中，提交偏移量到kafka，如果发生重复消费，数据库产生重复记录。

解决：如果可以保证原子操作，避免情况，或者做好重复消费，或者将偏移量保存到数据库中，做成一个事务。

seek()方法特定位置开始处理记录

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.Properties;

public class MessageSpecialCommitConsumer {
    public static void main(String[] args) {
        Properties consumerConfig = new Properties();
        // bootstrap.servers
        consumerConfig.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-node01:9092,kafka-node02:9092,kafka-node03:9092,");
        // 设置key的序列化器
        consumerConfig.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // 设置value的序列化器
        consumerConfig.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // group id
        consumerConfig.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group-test");
        // 关闭自动提交
        consumerConfig.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
        consumer.subscribe(Collections.singletonList("test-vip"), new ConsumerRebalanceListener() {
            // 再均衡开始之前和消费者停止读取消息之后被调用
            @Override
            public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
                // 重平衡之前提交事务
                commitDbTransaction();
            }

            // 在重新分配分区之后和消费者开始读取消息之前被调用
            @Override
            public void onPartitionsAssigned(Collection<TopicPartition> partitions) {

                for (TopicPartition partition : partitions) {
                    // 从数据库获取偏移量
                    consumer.seek(partition,getOffsetFromDB(partition));
                }
            }
        });
        // 马上调用一次
        consumer.poll(Duration.ofMillis(0L));
        for (TopicPartition topicPartition : consumer.assignment()) {
            // seek() 方法只更新我们正在使用的位置
            consumer.seek(topicPartition,getOffsetFromDB(partition));
        }
        while (true) {
            // 控制poll() 方法的超时时间，该参数被设为0，poll() 会立即返回，否则它会在指定的毫秒数内一直等待broker 返回数据
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100L));
            for (ConsumerRecord<String, String> record : records) {
                 processRecord(record);
                 // 保存偏移量到数据库
                storeOffsetInDB(record.topic(), record.partition(), record.offset()); 

            }
            commitDbTransaction();
        }
    }
}
```

## 退出

1. 确定要退出循环，需要通过**另一个线程**调用consumer.wakeup() 方法

   **`consumer.wakeup() `是消费者唯一一个可以从其他线程里安全调用的方法**

2. 退出线程之前调用`consumer.close()`

   提交任何还没有提交的东西，并向群组协调器发送消息，告知自己要离开群组，接下来就会触发再均衡，而不需要等待会话超时

   ```java
   Runtime.getRuntime().addShutdownHook(new Thread(){
       @Override
       public void run() {
           consumer.wakeup();
           try {
               mainThread.join();
           }catch (InterruptedException e){
               
           }
       }
   });
   
               try {
                   consumer.commitSync();
               }catch (WakeupException e){
                   // 忽略关闭异常
               }catch (Exception e) {
                   e.printStackTrace();
               }finally {
                   consumer.close();
               }
   ```

   ## 独立消费者 

   一个消费者可以订阅主题（并加入消费者群组），或者为自己分配分区，但不能同时做这两件事情

```java
List<PartitionInfo> partitionInfos = null;
// 向集群请求主题可用的分区。如果只打算读取特定分区，可以跳过这一步
partitionInfos = consumer.partitionsFor("topic");
if (partitionInfos != null) {
    Collection<TopicPartition> partitions = null;
    for (PartitionInfo partition : partitionInfos) {
        partitions.add(new TopicPartition(partition.topic(),
                partition.partition()));
    }
    // 知道需要哪些分区之后，调用assign() 方法
    consumer.assign(partitions);
    while (true) {
        ConsumerRecords<String, String> records =
                consumer.poll(Duration.ofMillis(1000L));
        for (ConsumerRecord<String, String> record: records) {
          processRecord(record);
        }
        consumer.commitSync();
    }
}

// 注意： ，如果主题增加了新的分区，消费者并不会收到通知。所以，要么周期性地调用consumer.
//     partitionsFor() 方法来检查是否有新分区加入，要么在添加新分区后重启应用程序
```

# 原理-1

## 集群成员关系

![image-20220316080509617](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220316080509617.png)

- broker 停机、出现网络分区或长时间垃圾回收停顿时，broker 会从Zookeeper 上断开连接，broker 在启动时创建的临时节点会自动从Zookeeper 上移除。监听broker 列表的Kafka 组件会被告知该broker 已移除
- 关闭broker 时，它对应的节点也会消失，不过它的ID 会继续存在于其他数据结构中
- 如果使用相同的ID 启动另一个全新的broker，它会立即加入集群，并拥有与旧broker相同的分区和主题

## 控制器

控制器其实就是一个broker，只不过它除了具有一般broker 的功能之外，还负责分区首领的选举

控制器创建流程：

- 集群里第一个启动的broker 通过在Zookeeper 里创建一个临时节点/controller 让自己成为控制器
- 其他broker 在启动时也会尝试创建这个节点，不过它们会收到一个“节点已存在”的异常
- 控制器被关闭或者与Zookeeper 断开连接，Zookeeper 上的临时节点就会消失。集群里的其他broker 通过watch 对象得到控制器节点消失的通知，它们会尝试让自己成为新的控制器

分区首领选择规则：

- 控制器发现一个broker 已经离开集群（通过观察相关的Zookeeper 路径）
- 控制器遍历这些分区，并确定谁应该成为新首领
- 向所有包含新首领或现有跟随者的broker 发送请求。该请求消息包含了谁是新首领以及谁是分区跟随者的信息
- 新首领开始处理来自生产者和消费者的请求，而跟随者开始从新首领那里复制消息

新broker加入集群：

- 控制器发现一个broker 加入集群时，它会使用broker ID 来检查新加入的broker 是否包
  含现有分区的副本。

- 如果有，控制器就把变更通知发送给新加入的broker 和其他broker，新broker 上的副本开始从首领那里复制消息

**Kafka 使用Zookeeper 的临时节 点来选举控制器，并在节点加入集群或退出集群时通知控制器。控制器负责在节点加入或离开集群时进行分区首领选举。控制器使用epoch 来避免“脑裂”。“脑裂”是指两个节点同时认为自己是当前的控制器**

## 复制

>Kafka 使用主题来组织数据，每个主题被分为若干个分区，每个分区有多个副本。那些副
>本被保存在broker 上，每个broker 可以保存成百上千个属于不同主题和分区的副本

副本类型：

1. 首领副本

   每个分区都有一个首领副本。为了保证一致性，所有生产者请求和消费者请求都会经过这个副本

2. 跟随者副本

   首领以外的副本都是跟随者副本。跟随者副本不处理来自客户端的请求，它们唯一的任
   就是从首领那里复制消息，保持与首领一致的状态。如果首领发生崩溃，其中的一个跟随者会被提升为新首领

如果跟随者在10s 内没有请求任何消息，或者虽然在请求消息，但在10s 内没有请求最新的数据，那么它就会被认为是不同步的。如果一个副本无法与首领保持一致，在首领发生失效时，它就不可能成为新首领——毕竟它没有包含全部的消息。

**持续请求得到的最新消息副本被称为同步的副本。在首领发生失效时，只有同步副本才有可能被选为新首领**

## broker配置

### 复制系数

主题级别： replication.factor

在broker 级别：可以通过default.replication.factor 来配置自动创建的主题

**假设主题的复制系数都是3，也就是说每个分区总共会被3 个不同的**broker 复制3 次

**在要求可用性的场景里把复制系数设为3**

###  不完全的首领选举

**unclean.leader.election** 只能在broker 级别（实际上是在集群范围内）进行配置，它的默 	认值是true。

如果我们允许不同步的副本成为首领，那么就要承担丢失数据和出现数据不一致的风险。如果不允许它们成为首领，那么就要接受较低的可用性，因为我们必须等待原先的首领恢复到可用状态

### 最少同步副本

**在主题级别和broker 级别上，这个参数都叫min.insync.replicas**

根据Kafka 对可靠性保证的定义，消息只有在被写入到所有同步副本之后才被认为是已提交的,但有些极端情况下只有一个同步副本

### 警示

- 根据可靠性需求配置恰当的acks 值。
- 在参数配置和代码里正确处理错误

### 已提交消息和已提交偏移量

**已提交消息与之前讨论过的已提交消息是不一样的，它是指已经被写入所有同步副本并且对消费者可见的消息**

**已提交偏移量是指消费者发送给Kafka 的偏移量，用于确认它已经收到并处理好的消息位置**

### 消费者可靠性保证

- group.id
- auto.offset.reset
- enable.auto.commit
- auto.commit.interval.ms

### 消费者消费消息处理

####  重复消息

- 生产者发送消息中加入唯一标识，消费者可以感知到有重复消息
- 消费者做好冥等性处理重复消息

### 消费者重试模型

- 可重试错误=》提交最后一个处理成功的偏移量=》没有处理的保存在缓冲区=》pause() => 重试处理，重试成功，重试错误记录异常消息=》resume()=》轮询获取数据
- 可重试错误=》写入一个独立的主题，继续处理=》一个独立的消费这群组从主题读取错误消息，进行重试，重试时期暂停该主题。Dead-letter-queue

**Kafka 本身就使用了回压策略:必要时可以延后向生产者发送确认**
