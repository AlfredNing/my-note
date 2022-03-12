# flink

>流处理
>
>​	keyBy()	StreamExecutionEnvironment
>
>批处理
>
>​	groupBy()	ExecutionEnvironment
>
>
>flink 懒加载 -》env.execute()



# Flink - 常用API

## DataStream Api

#### 1.1 DataSource

1. 文件读取

   ```java
   public class SourceFromFile {
       public static void main(String[] args) throws Exception {
           String inputPath = "F:\\gitPro\\first-flink\\data\\input\\hello.txt";
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
           DataStreamSource<String> data = env.readTextFile(inputPath);
           SingleOutputStreamOperator<Tuple2<String, Integer>> wordAndOne = data.flatMap((FlatMapFunction<String, Tuple2<String, Integer>>) (s, collector) -> {
               for (String str : s.split("\\s+")) {
                   collector.collect(new Tuple2<String, Integer>(str, 1));
               }
           }).returns(Types.TUPLE(Types.STRING,Types.INT)); // java8 lambda 需要返回类型
           KeyedStream<Tuple2<String, Integer>, String> keyedStream = wordAndOne.keyBy((KeySelector<Tuple2<String, Integer>, String>) stringIntegerTuple2 -> stringIntegerTuple2.f0);
           keyedStream.sum(1).print();
   
           env.execute();
       }
   }
   Hdfs
   ```

   **	**

   ```java
   public class SourceFromIHdfs {
       public static void main(String[] args) throws Exception {
           String inputPath = "hdfs://lino2:9000/hi.txt";
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
           DataStreamSource<String> data = env.readTextFile(inputPath);
           SingleOutputStreamOperator<Tuple2<String, Integer>> wordAndOne = data.flatMap((FlatMapFunction<String, Tuple2<String, Integer>>) (s, collector) -> {
               for (String str : s.split("\\s+")) {
                   collector.collect(new Tuple2<String, Integer>(str, 1));
               }
           }).returns(Types.TUPLE(Types.STRING,Types.INT));
           KeyedStream<Tuple2<String, Integer>, String> keyedStream = wordAndOne.keyBy((KeySelector<Tuple2<String, Integer>, String>) stringIntegerTuple2 -> stringIntegerTuple2.f0);
           keyedStream.sum(1).print();
   
           env.execute();
       }
   }
   ```

   **pom.xml 添加依赖**

   ```xml
   	<dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-hadoop-compatibility_2.11</artifactId>
               <version>1.11.1</version>
           </dependency>
           <dependency>
               <groupId>org.apache.hadoop</groupId>
               <artifactId>hadoop-common</artifactId>
               <version>2.8.5</version>
           </dependency>
           <dependency>
               <groupId>org.apache.hadoop</groupId>
               <artifactId>hadoop-hdfs</artifactId>
               <version>2.8.5</version>
           </dependency>
           <dependency>
               <groupId>org.apache.hadoop</groupId>
               <artifactId>hadoop-client</artifactId>
               <version>2.8.5</version>
           </dependency>
   ```

2. 基于内存

   - socket

   ```java
   object SourceFromISocket {
     def main(args: Array[String]): Unit = {
       val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
   
       val value: DataStream[(String, Int)] = env.socketTextStream("localhost", 7777).flatMap(_.split("\\s+")).map((_, 1)).keyBy(0)
         .sum(1).setParallelism(1)
   
       value.print()
   
       env.execute()
     }
   
   }
   ```

   - 基于集合

     ```java
     public class SourceFromICollection {
         public static void main(String[] args) throws Exception {
             StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
             ArrayList<Person> list = new ArrayList<>();
             Person p1 = new Person("yzx", 18);
             Person p2 = new Person("nq", 28);
             Person p3 = new Person("yuge", 38);
             list.add(p1);
             list.add(p2);
             list.add(p3);
     
             env.fromCollection(list).filter((FilterFunction<Person>) person -> person.age > 20).print();
             env.execute();
         }
     }
     
     @Data
     @AllArgsConstructor
     @NoArgsConstructor
     class Person {
         String name;
         Integer age;
     }
     
     ```

3. 自定义数据源

   可以通过 env.addSource() 添加自定义数据源

   - implements SourceFunction :不能设置并行度

   - implements ParallelSourceFunction : 可以设置并行度

   - extends RichParallelSourceFunction

     **Flink 已经实现很多得连接器**

   1. kafka

      ```xml
      <dependency>
          <groupId>org.apache.flink</groupId>
          <artifactId>flink-connector-kafka_2.11</artifactId>
          <version>1.11.1</version>
      </dependency>
      ```

      ```java
      public class SourceFromKafka {
          public static void main(String[] args) throws Exception {
              StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      
              Properties props = new Properties();
              props.put("bootstrap.servers", "192.168.10.118:9092");
              FlinkKafkaConsumer<String> consumer = new FlinkKafkaConsumer<>("nq", new SimpleStringSchema(), props);
      
              DataStreamSource<String> data = env.addSource(consumer);
      
              SingleOutputStreamOperator<Tuple2<String, Integer>> wordAndOne = data.flatMap((FlatMapFunction<String, Tuple2<String, Integer>>) (s, collector) -> {
                  for (String str : s.split("\\s+")) {
                      collector.collect(new Tuple2<String, Integer>(str, 1));
                  }
              }).returns(Types.TUPLE(Types.STRING, Types.INT));
              KeyedStream<Tuple2<String, Integer>, String> keyedStream = wordAndOne.keyBy((KeySelector<Tuple2<String, Integer>, String>) stringIntegerTuple2 -> stringIntegerTuple2.f0);
              keyedStream.sum(1).print();
      
              env.execute();
          }
      }
      ```

   2. 自定义数据源

      - SourceFunction 

        ```java
        public class CustomSourceFunction implements SourceFunction<String> {
            long count = 0;
            boolean isRunning = true;
        
            @Override
            public void run(SourceContext<String> sourceContext) throws Exception {
                while (isRunning) {
                    count++;
                    sourceContext.collect(String.valueOf(count));
                    Thread.sleep(1000);
                }
            }
        
            @Override
            public void cancel() {
                isRunning = false;
            }
        }
        
        public class CustomSourceFunctionRun {
            public static void main(String[] args) throws Exception {
                StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
                DataStreamSource<String> streamSource = env.addSource(new CustomSourceFunction());
                streamSource.print();
                env.execute();
            }
        }
        /**
        *
        4> 1
        5> 2
        6> 3
        7> 4
        8> 5
        1> 6
        2> 7
        3> 8
        4> 9
        5> 10
        6> 11
        7> 12
        */
        ```

      - ParallelSourceFunction 

        ```java
        public class CustomParallelSourceFunction implements ParallelSourceFunction<String> {
            long count = 0;
            boolean isRunning = true;
        
            @Override
            public void run(SourceContext<String> sourceContext) throws Exception {
                while (isRunning) {
                    count++;
                    sourceContext.collect(String.valueOf(count));
                    Thread.sleep(1000);
                }
            }
        
            @Override
            public void cancel() {
                isRunning = false;
            }
        }
        
        public class CustomParallelSourceFunctionRun {
            public static void main(String[] args) throws Exception {
                StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
                DataStreamSource<String> streamSource = env.addSource(new CustomParallelSourceFunction());
                streamSource.print();
                env.execute();
            }
        }
        
        /**
        7> 1
        1> 1
        4> 1
        6> 1
        3> 1
        8> 1
        5> 1
        2> 1
        1> 2
        2> 2
        8> 2
        6> 2
        5> 2
        4> 2
        7> 2
        3> 2
        1> 3
        2> 3
        6> 3
        **/
        ```

      - RichParallelSourceFunction

        ```java
        public class CustomRichParallelSourceFunction extends RichParallelSourceFunction<String> {
            long count = 0;
            boolean isRunning = true;
        
            @Override
            public void run(SourceContext<String> sourceContext) throws Exception {
                while (isRunning) {
                    count++;
                    sourceContext.collect(String.valueOf(count));
                    Thread.sleep(1000);
                }
            }
        
            @Override
            public void cancel() {
                isRunning = false;
            }
        }
        
        public class CustomRichParallelSourceFunctionRunRun {
            public static void main(String[] args) throws Exception {
                StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
                DataStreamSource<String> streamSource = env.addSource(new CustomRichParallelSourceFunction());
                streamSource.print();
                env.execute();
            }
        }
        
        /**
        6> 1
        1> 1
        8> 1
        3> 1
        7> 1
        2> 1
        5> 1
        4> 1
        7> 2
        3> 2
        5> 2
        4> 2
        8> 2
        1> 2
        2> 2
        6> 2
        **/
        ```

        **SourceFunction 是flink stream 中顶层接口**

#### 1.2 Fink Tranformation

1. Map 
2. FlatMap
3. filter
4. KeyBy : 逻辑分区 ，同一个key 发送给Task
5. Reduce : rolling 过程
6. Fold: 提供初始值
7. Aggregations: sum min max
8. window: 窗口机制
   1. windowAll
   2. windowApply
   3. WindowReduce
   4. Window Fold
   5. Aggregations on windows
9. union
10. Window join
11. Interval join
12. Window CoGroup
13. Connect
14. Comap, CoFlatMap
15. Split
16. Select  Split 联合使用
17. Iterate

Eg:

```java
// connect demo
public class ConnectDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> parallelSource = env.addSource(new CustomParallelSourceFunction());
        DataStreamSource<String> richSource = env.addSource(new CustomRichParallelSourceFunction());
        ConnectedStreams<String, String> connectedStreams = parallelSource.connect(richSource);
        SingleOutputStreamOperator<String> outputStreamOperator = connectedStreams.map(new CoMapFunction<String, String, String>() {
            @Override
            public String map1(String s) throws Exception {
                return s;
            }

            @Override
            public String map2(String s) throws Exception {
                return s;
            }
        });

        outputStreamOperator.print();

        env.execute();

    }
}

// split + split demo
public class SplitSelectDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<Integer> streamSource = env.fromElements(1, 2, 3, 4, 5, 6, 7, 8, 9);
        SplitStream<Integer> splitStream = streamSource.split((OutputSelector<Integer>) integer -> {
            ArrayList<String> list = new ArrayList<>();
            if (integer % 2 == 0) {
                list.add("even");
            } else {
                list.add("odd");
            }
            return list;
        });
        splitStream.select("even").print();
        env.execute();
    }
}
```

#### 1.3 Sink

数据目的地，已经提供大量sink

- Redis

  ```xml
  		 <dependency>
              <groupId>org.apache.flink</groupId>
              <artifactId>flink-connector-redis_2.11</artifactId>
              <version>1.1.5</version>
          </dependency>
  ```

  ```java
  public class MyRedisSink {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          DataStreamSource<String> socketTextStream = env.socketTextStream("localhost", 7777);
          SingleOutputStreamOperator<Tuple2<String, String>> operator = socketTextStream
                  .map(x -> new Tuple2<>("nq", x)).returns(Types.TUPLE(Types.STRING,Types.STRING));
  
          FlinkJedisPoolConfig jredisConfig = new FlinkJedisPoolConfig.Builder().setHost("localhost").setPort(6379).build();
          RedisSink<Tuple2<String, String>> redisSink = new RedisSink<>(jredisConfig, new RedisMapper<Tuple2<String, String>>() {
              @Override
              public RedisCommandDescription getCommandDescription() {
                  return new RedisCommandDescription(RedisCommand.LPUSH);
              }
  
              @Override
              public String getKeyFromData(Tuple2<String, String> stringStringTuple2) {
                  return stringStringTuple2.f0;
              }
  
              @Override
              public String getValueFromData(Tuple2<String, String> stringStringTuple2) {
                  return stringStringTuple2.f1;
              }
          });
          operator.addSink(redisSink);
  
          env.execute();
      }
  }
  ```

- Mysql - 自定义Sink

  ```java
  public class MysqlSink {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          Student stu1 = new Student("nq", 20);
          Student stu2 = new Student("yzx", 18);
          DataStreamSource<Student> streamSource = env.fromElements(stu1, stu2);
          String url = "jdbc:mysql://localhost:3306/srm?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&useSSL=false";
          String username = "root";
          String password = "ningqiang";
  
          streamSource.addSink(new RichSinkFunction<Student>() {
              Connection connection = null;
              PreparedStatement pstmt = null;
  
              @Override
              public void open(Configuration parameters) throws Exception {
  
                  connection = DriverManager.getConnection(url, username, password);
                  String sql = "insert into student(name,age) values (?,?)";
                  pstmt = connection.prepareStatement(sql);
  
              }
  
              @Override
              public void invoke(Student value, Context context) throws Exception {
                  pstmt.setString(1, value.getName());
                  pstmt.setInt(2, value.getAge());
                  pstmt.executeUpdate();
              }
  
              @Override
              public void close() throws Exception {
                  if (connection != null) {
                      connection.close();
                  }
  
                  if (pstmt != null) {
                      pstmt.close();
                  }
              }
          });
  
          env.execute();
      }
  }
  
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  class Student {
      private String name;
      private Integer age;
  }
  ```

## DataSet API

>DataSet API同DataStream API一样有三个组成部分，各部分作用对应一致
>
>flink 1.9 之后有意融合一套Api

### Flink Talbe API  &&  SQL_API

>Apache Flink提供了两种顶层的关系型API，分别为Table API和SQL，Flink通过Table API&SQL实现了
>批流统一。其中Table API是用于Scala和Java的语言集成查询API，它允许以非常直观的方式组合关系运
>算符（例如select，where和join）的查询。Flink SQL基于Apache Calcite 实现了标准的SQL，用户可
>以使用标准的SQL处理数据集。Table API和SQL与Flink的DataStream和DataSet API紧密集成在一起，
>用户可以实现相互转化，比如可以将DataStream或者DataSet注册为table进行操作数据。值得注意的
>是，Table API and SQL目前尚未完全完善，还在积极的开发中，所以并不是所有的算子操作都可以通
>过其实现。

POM.XML

```xml
 <!--TABLE-API-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table</artifactId>
            <version>1.11.1</version>
            <type>pom</type>
            <scope>provided</scope>
        </dependency>
        <!-- Either... -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-api-java-bridge_2.11</artifactId>
            <version>1.11.1</version>
            <scope>provided</scope>
        </dependency>
        <!-- or... -->
        <!--        <dependency>-->
        <!--            <groupId>org.apache.flink</groupId>-->
        <!--            <artifactId>flink-table-api-scala-bridge_2.11</artifactId>-->
        <!--            <version>1.11.1</version>-->
        <!--            <scope>provided</scope>-->
        <!--        </dependency>-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner-blink_2.12</artifactId>
            <version>1.11.1</version>
            <scope>provided</scope>
        </dependency>
```

- eg:

  ```java
  public class MyTableDemo {
      public static void main(String[] args) throws Exception {
          // flink 执行环境 env
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          // env => table环境tEnv
          StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);
          // 获取流式数据源
          DataStreamSource<Tuple2<String, Integer>> dataStreamSource = env.addSource(new SourceFunction<Tuple2<String, Integer>>() {
              @Override
              public void run(SourceContext<Tuple2<String, Integer>> sourceContext) throws Exception {
                  while (true) {
                      sourceContext.collect(new Tuple2<>("name", 10));
                      Thread.sleep(1_000);
                  }
              }
  
              @Override
              public void cancel() {
  
              }
          });
          // 将流式数据源做成table
          Table table = tEnv.fromDataStream(dataStreamSource, $("name"), $("age"));
          // 查询table
          Table name = table.select($("name"));
          // 将处理结果输出到控制台
          DataStream<Tuple2<Boolean, Row>> res = tEnv.toRetractStream(name, Row.class);
          res.print();
          env.execute();
  
      }
  }
  ```

## Flink 时间窗口

>- 基于时间窗口
>- 基于事件窗口
>- 翻滚（滚动）窗口：Tumbling Window 无重叠，将数据依据固定的窗口长度对数据进行切分
>  - 时间对齐，窗口长度固定，没有重叠
>- 滑动窗口： Sliding Window 有重叠，滑动窗口是固定窗口的更广义的一种形式，滑动窗口由固定的窗口长度和滑动间隔组成
>  - 窗口长度固定，可以有重叠
>- 会话窗口：Session Window 活动间隙，
>  - 会话窗口不重叠，没有固定的开始和结束时间
>    与翻滚窗口和滑动窗口相反, 当会话窗口在一段时间内没有接收到元素时会关闭会话窗
>- 全局窗口

```shell
1. 获取数据源
2. 获取窗口
3. 操作窗口数据
4. 输出窗口数据
```

#### 滚动窗口

```java
public class TumblingWindow {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> streamSource = env.addSource(new SourceFunction<String>() {
            int count = 0;

            @Override
            public void run(SourceContext<String> sourceContext) throws Exception {
               while (true){
                   sourceContext.collect(count + " 号数据源");
                   count++;
                   Thread.sleep(1_000);
               }
            }

            @Override
            public void cancel() {

            }
        });
        SingleOutputStreamOperator<Tuple3<String, String, String>> map = streamSource.map(new MapFunction<String, Tuple3<String, String, String>>() {

            @Override
            public Tuple3<String, String, String> map(String s) throws Exception {
                String strDate = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS").withZone(ZoneId.of("UTC")).format(Instant.now());
                int nextInt = new Random().nextInt(10);
                return new Tuple3<>(s, strDate, String.valueOf(nextInt));
            }
        });

        KeyedStream<Tuple3<String, String, String>, String> keyedStream = map.keyBy(value -> value.f0);

        WindowedStream<Tuple3<String, String, String>, String, TimeWindow> timeWindow = keyedStream.timeWindow(Time.seconds(5));
        // 基于时间驱动
        SingleOutputStreamOperator<String> outputStreamOperator = timeWindow.apply(new WindowFunction<Tuple3<String, String, String>, String, String, TimeWindow>() {
            @Override
            public void apply(String s, TimeWindow timeWindow, Iterable<Tuple3<String, String, String>> iterable, Collector<String> collector) throws Exception {
                /**
                 * s 输入数据
                 * timewindow 窗口
                 * iterable，转换之后得stream集合
                 *
                 */
                StringBuilder sb = new StringBuilder();
                Iterator<Tuple3<String, String, String>> iterator = iterable.iterator();
                while (iterator.hasNext()) {
                    Tuple3<String, String, String> tuple3 = iterator.next();
                    sb.append(tuple3.f0 + "..." + tuple3.f1 + "..." + tuple3.f2);
                }

                String s1 = s + "===" + timeWindow.getStart() + "===" + sb;
                collector.collect(s1);
            }
        });
        outputStreamOperator.print();

        env.execute();
    }
}
```

CountWindow

```java
public class WindowCount {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> streamSource = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<Tuple3<String, String, String>> map = streamSource.map(new MapFunction<String, Tuple3<String, String, String>>() {

            @Override
            public Tuple3<String, String, String> map(String s) throws Exception {
                String strDate = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS").withZone(ZoneId.of("UTC")).format(Instant.now());
                int nextInt = new Random().nextInt(10);
                return new Tuple3<>(s, strDate, String.valueOf(nextInt));
            }
        });

        KeyedStream<Tuple3<String, String, String>, String> keyedStream = map.keyBy(value -> value.f0);
		// 基于事件驱动
        WindowedStream<Tuple3<String, String, String>, String, GlobalWindow> countWindow = keyedStream.countWindow(3);
        SingleOutputStreamOperator<String> outputStreamOperator = countWindow.apply(new WindowFunction<Tuple3<String, String, String>, String, String, GlobalWindow>() {
            @Override
            public void apply(String s, GlobalWindow globalWindow, Iterable<Tuple3<String, String, String>> iterable, Collector<String> collector) throws Exception {
                StringBuilder sb = new StringBuilder();
                Iterator<Tuple3<String, String, String>> iterator = iterable.iterator();
                while (iterator.hasNext()) {
                    Tuple3<String, String, String> tuple3 = iterator.next();
                    sb.append(tuple3.f0 + "..." + tuple3.f1 + "..." + tuple3.f2);
                }

                collector.collect(sb.toString());
            }
        });

        outputStreamOperator.print();

        env.execute();
    }
}
```

#### 滑动窗口

- 基于时间的

```java
public class SlidingWindow {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> streamSource = env.addSource(new SourceFunction<String>() {
            int count = 0;

            @Override
            public void run(SourceContext<String> sourceContext) throws Exception {
                while (true) {
                    sourceContext.collect(count + " 号数据源");
                    count++;
                    Thread.sleep(1_000);
                }
            }

            @Override
            public void cancel() {

            }
        });
        SingleOutputStreamOperator<Tuple3<String, String, String>> map = streamSource.map(new MapFunction<String, Tuple3<String, String, String>>() {

            @Override
            public Tuple3<String, String, String> map(String s) {
                DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS").withZone(ZoneId.of("UTC"));
                String strDate = formatter.format(Instant.now());
                int nextInt = new Random().nextInt(10);
                return new Tuple3<>(s, strDate, String.valueOf(nextInt));
            }
        });

        KeyedStream<Tuple3<String, String, String>, String> keyedStream = map.keyBy(value -> value.f0);

        WindowedStream<Tuple3<String, String, String>, String, TimeWindow> timeWindow = keyedStream.timeWindow(Time.seconds(5), Time.seconds(2));
        SingleOutputStreamOperator<String> outputStreamOperator = timeWindow.apply(new WindowFunction<Tuple3<String, String, String>, String, String, TimeWindow>() {
            @Override
            public void apply(String s, TimeWindow timeWindow, Iterable<Tuple3<String, String, String>> iterable, Collector<String> collector) throws Exception {
                /**
                 * s 输入数据
                 * timewindow 窗口
                 * iterable，转换之后得stream集合
                 *
                 */
                StringBuilder sb = new StringBuilder();
                Iterator<Tuple3<String, String, String>> iterator = iterable.iterator();
                while (iterator.hasNext()) {
                    Tuple3<String, String, String> tuple3 = iterator.next();
                    sb.append(tuple3.f0 + "..." + tuple3.f1 + "..." + tuple3.f2);
                }

                String s1 = s + "===" + timeWindow.getStart() + "|||" + timeWindow.getEnd() + "===" + sb;
                collector.collect(s1);
            }
        });
        outputStreamOperator.print();

        env.execute();
    }
}
```

- 基于事件的

- ```java
  public class SlideWindowCount {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          DataStreamSource<String> streamSource = env.socketTextStream("localhost", 7777);
          SingleOutputStreamOperator<Tuple3<String, String, String>> map = streamSource.map(new MapFunction<String, Tuple3<String, String, String>>() {
  
              @Override
              public Tuple3<String, String, String> map(String s) throws Exception {
                  String strDate = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS").withZone(ZoneId.of("UTC")).format(Instant.now());
                  int nextInt = new Random().nextInt(10);
                  return new Tuple3<>(s, strDate, String.valueOf(nextInt));
              }
          });
  
          KeyedStream<Tuple3<String, String, String>, String> keyedStream = map.keyBy(value -> value.f0);
  
          WindowedStream<Tuple3<String, String, String>, String, GlobalWindow> countWindow = keyedStream.countWindow(3, 2);
          SingleOutputStreamOperator<String> outputStreamOperator = countWindow.apply(new WindowFunction<Tuple3<String, String, String>, String, String, GlobalWindow>() {
              @Override
              public void apply(String s, GlobalWindow globalWindow, Iterable<Tuple3<String, String, String>> iterable, Collector<String> collector) throws Exception {
                  StringBuilder sb = new StringBuilder();
                  Iterator<Tuple3<String, String, String>> iterator = iterable.iterator();
                  while (iterator.hasNext()) {
                      Tuple3<String, String, String> tuple3 = iterator.next();
                      sb.append(tuple3.f0 + "..." + tuple3.f1 + "..." + tuple3.f2);
                  }
  
                  collector.collect(sb.toString());
              }
          });
  
          outputStreamOperator.print();
  
          env.execute();
      }
  }
  ```

#### 会话窗口

```java
public class WindowDemoSession {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> streamSource = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<String> streamOperator = streamSource.map(x -> x);
        KeyedStream<String, String> keyedStream = streamOperator.keyBy(v -> v);
        WindowedStream<String, String, TimeWindow> window = keyedStream.window(ProcessingTimeSessionWindows.withGap(Time.seconds(5)));
        SingleOutputStreamOperator<String> operator = window.apply(new WindowFunction<String, String, String, TimeWindow>() {
            @Override
            public void apply(String s, TimeWindow timeWindow, Iterable<String> iterable, Collector<String> collector) throws Exception {

                StringBuilder sb = new StringBuilder();
                Iterator<String> iterator = iterable.iterator();
                while (iterator.hasNext()) {
                    String str = iterator.next();
                    sb.append(str);
                }
                collector.collect(sb.toString());
            }
        });
        operator.print();
        env.execute();
    }
}
```

## Fink Time

### 时间语义

- 事件时间：事件发生的时间
- 摄入时间：数据进入Flink的时
- 处理时间：某个Flink节点执行某个operation的时间

### 数据延迟解决方案

Watermark：**水印(watermark)就是一个时间戳，Flink可以给数据流添加水印**

当数据流添加水印后，会按照水印时间来触发窗口计算

**水印时间 = 事件时间 - 允许延迟时间**

允许延迟时间： 目前生成环境中还是以测试为准设定

Demo

>1. 获取数据源
>2. 转化
>3. 声明水印
>4. 分组聚合，调用window
>5. 保存处理结果

**当使用EventTimeWindow时，所有的Window在EventTime的时间轴上进行划分，**
**也就是说，在Window启动后，会根据初始的EventTime时间每隔一段时间划分一个窗口，**
**如果Window大小是3秒，那么1分钟内会把Window划分为如下的形式：**
[00:00:00,00:00:03)
[00:00:03,00:00:06)
[00:00:03,00:00:09)
[00:00:03,00:00:12)

```java
public class WaterMarkDemo1 {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 设置时间语义，默认为摄入时间
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        // 设置水印周期执行时间，默认是2s
        env.getConfig().setAutoWatermarkInterval(1000);
        env.setParallelism(1);

        DataStreamSource<String> streamSource = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<Tuple2<String, Long>> mapOperator = streamSource.map(new MapFunction<String, Tuple2<String, Long>>() {
            @Override
            public Tuple2<String, Long> map(String s) throws Exception {
                String[] split = s.split(",");
                return new Tuple2<>(split[0], Long.valueOf(split[1]));
            }
        });
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
        SingleOutputStreamOperator<Tuple2<String, Long>> watermarks = mapOperator.assignTimestampsAndWatermarks(new WatermarkStrategy<Tuple2<String, Long>>() {
            @Override
            public WatermarkGenerator<Tuple2<String, Long>> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
                return new WatermarkGenerator<Tuple2<String, Long>>() {
                    long maxTimeStamp = Long.MIN_VALUE;

                    /**
                     * 每来一个数据执行一次
                     * @param stringLongTuple2
                     * @param l
                     * @param watermarkOutput
                     */
                    @Override
                    public void onEvent(Tuple2<String, Long> stringLongTuple2, long l, WatermarkOutput watermarkOutput) {
                        maxTimeStamp = Math.max(stringLongTuple2.f1, maxTimeStamp);
                        System.out.println("maxTimeStamp: " +maxTimeStamp + "fmt: " + sdf.format(maxTimeStamp));
                    }

                    /**
                     * 周期性提交水印
                     * @param watermarkOutput
                     */
                    @Override
                    public void onPeriodicEmit(WatermarkOutput watermarkOutput) {
                        System.out.println("...onPeriodicEmit...");
                        long maxOutOfOrderness = 3000L;
                        watermarkOutput.emitWatermark(new Watermark(maxTimeStamp - maxOutOfOrderness));
                    }
                };
            }

        }.withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {
            @Override
            public long extractTimestamp(Tuple2<String, Long> stringLongTuple2, long l) {
                return stringLongTuple2.f1;
            }
        }));
        KeyedStream<Tuple2<String, Long>, String> keyedStream = watermarks.keyBy(value -> value.f0);
        WindowedStream<Tuple2<String, Long>, String, TimeWindow> window = keyedStream.window(TumblingEventTimeWindows.of(Time.seconds(4)));
        SingleOutputStreamOperator<String> outputStreamOperator = window.apply(new WindowFunction<Tuple2<String, Long>, String, String, TimeWindow>() {
            @Override
            public void apply(String s, TimeWindow timeWindow, Iterable<Tuple2<String, Long>> iterable, Collector<String> collector) throws Exception {
                System.out.println("...."+sdf.format(timeWindow.getStart()));
                Iterator<Tuple2<String, Long>> iterator = iterable.iterator();
                ArrayList<Long> longArrayList = new ArrayList<>();
                while (iterator.hasNext()) {
                    Tuple2<String, Long> longTuple2 = iterator.next();
                    longArrayList.add(longTuple2.f1);
                }
                // 常规优化方案
                Collections.sort(longArrayList);
                String k = "key: " + s + "list(0)" + sdf.format(longArrayList.get(0)) + "list(end)" + sdf.format(longArrayList.get(longArrayList.size() - 1)) + "start: " + sdf.format(timeWindow.getStart()) + "edn" + sdf.format(timeWindow.getEnd());
                collector.collect(k);
            }
        });
        outputStreamOperator.print();
        env.execute();
    }
}
// 也就是说，窗口的开始时间flink按照自己的划分已经提前划分好
```

## State机制

### 状态类型

有状态：依赖之前或之后的事件

无状态计算：独立

根据数据结构不同，Flink定义了多种state，应用于不同的场景

- ValueState：即类型为T的单值状态。这个状态与对应的key绑定，是最简单的状态了。它可以通
  过update 方法更新状态值，通过value() 方法获取状态值。
- ListState：即key上的状态值为一个列表。可以通过add 方法往列表中附加值；也可以通过get()
  方法返回一个Iterable<T> 来遍历状态值。
- ReducingState：这种状态通过用户传入的reduceFunction，每次调用add 方法添加值的时候，
  会调用reduceFunction，最后合并到一个单一的状态值。
- FoldingState：跟ReducingState有点类似，不过它的状态值类型可以与add 方法中传入的元素类
  型不同（这种状态将会在Flink未来版本中被删除
- MapState：即状态值为一个map。用户通过put 或putAll 方法添加元素

**Keyed State:KeyedStream流上的每一个Key都对应一个State**

Demo

```java
public class StateDemo1 {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<Tuple2<Long, Long>> streamSource = env.fromElements(
                new Tuple2<Long, Long>(1L, 3L),
                new Tuple2<Long, Long>(1L, 5L),
                new Tuple2<Long, Long>(1L, 7L),
                new Tuple2<Long, Long>(1L, 5L),
                new Tuple2<Long, Long>(1L, 2L));

        KeyedStream<Tuple2<Long, Long>, Long> keyedStream = streamSource.keyBy(v -> v.f0);
        SingleOutputStreamOperator<Tuple2<Long, Long>> streamOperator = keyedStream.flatMap(new RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>>() {
            ValueState<Tuple2<Long, Long>> valueState;

            @Override
            public void open(Configuration parameters) throws Exception {
                // 设置state
                ValueStateDescriptor<Tuple2<Long, Long>> stateDescriptor = new ValueStateDescriptor<Tuple2<Long, Long>>(
                        "average",
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {
                        }),
                        Tuple2.of(0L, 0L)
                );

                valueState = getRuntimeContext().getState(stateDescriptor);
                super.open(parameters);
            }

            @Override
            public void flatMap(Tuple2<Long, Long> longLongTuple2, Collector<Tuple2<Long, Long>> collector) throws Exception {
                // 更新state
                Tuple2<Long, Long> currentState = valueState.value();
                Tuple2<Long, Long> tuple2 = new Tuple2<>(currentState.f0 + 1, currentState.f1 + longLongTuple2.f1);
                valueState.update(tuple2);
                if (tuple2.f0 >= 5) {
                    collector.collect(new Tuple2<>(longLongTuple2.f0, currentState.f1 / currentState.f0));

                    // 清空state
                    valueState.clear();
                }
            }
        });
        streamOperator.print();

        env.execute();

    }
}
```

![image-20220115122635718](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220115122635718.png)

Operator State

![image-20220115130606870](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220115130606870.png)

>flink创造Operator State 有两种方法， 实现**CheckpointedFunction** ，或者 **ListCheckPointed**
>
>snapshotState  和 initializeState
>
>​	initializeState ： 方法内，实例化状态
>
>​	snapshotState ： 检查点执行，将最新数据放到最新检查点上
>
>invoke： 每来一个算子调用一次，把所有数据放到缓冲器当中，目的是为了checkoukpoint拿出数据

Demo

```java
public class OperatorStateDemo implements SinkFunction<Tuple2<Long, Long>>, CheckpointedFunction {
    ListState<Tuple2<Long, Long>> listState;
    private List<Tuple2<Long, Long>> elements;

    public OperatorStateDemo(ListState<Tuple2<Long, Long>> listState, List<Tuple2<Long, Long>> elements) {
        this.listState = listState;
        this.elements = elements;
    }

    @Override
    public void invoke(Tuple2<Long, Long> value, Context context) throws Exception {
        elements.add(value);
        if (elements.size() == 5) {

        }
        elements.clear();
    }

    @Override
    public void snapshotState(FunctionSnapshotContext functionSnapshotContext) throws Exception {
        // 保存点调用
        listState.clear();;
        for (Tuple2<Long, Long> element : elements) {
            listState.add(element);
        }

    }

    @Override
    public void initializeState(FunctionInitializationContext functionInitializationContext) throws Exception {
        ListStateDescriptor<Tuple2<Long, Long>> operatorState = new ListStateDescriptor<>(
                "operatorState",
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {
                })
        );
        listState = functionInitializationContext.getOperatorStateStore().getListState(operatorState);
    }
}
```

```java
public class OperatorStateDemo implements SinkFunction<Tuple2<Long, Long>>, CheckpointedFunction {
    ListState<Tuple2<Long, Long>> listState;
    private List<Tuple2<Long, Long>> elements;
    private int threshold;

    public OperatorStateDemo(int threshold) {
        this.threshold = threshold;
        this.elements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<Long, Long> value, Context context) throws Exception {
        elements.add(value);
        if (elements.size() == threshold) {
            for (Tuple2<Long, Long> element : elements) {
                System.out.println("...out " + element);
            }
            elements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext functionSnapshotContext) throws Exception {
        System.out.println("....snaphostState");
        // 保存点调用
        listState.clear();

        for (Tuple2<Long, Long> element : elements) {
            listState.add(element);
        }

    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        System.out.println("...initializeState");
        ListStateDescriptor<Tuple2<Long, Long>> operatorState = new ListStateDescriptor<>(
                "operatorState",
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {
                })
        );
        listState = context.getOperatorStateStore().getListState(operatorState);
        if (context.isRestored()) { // 程序异常，为true
            Iterable<Tuple2<Long, Long>> iterable = listState.get();
            for (Tuple2<Long, Long> longTuple2 : iterable) {
                elements.add(longTuple2);
            }


        }
    }
}
```

### **Operator State** 与 keyState 区别

**Keyed State**

表示和Key相关的一种State，只能用于KeydStream类型数据集对应的Functions和 Operators之上。
Keyed State是 Operator State的特例，区别在于 Keyed State 事先按照key对数据集进行了分区，每个
Key State 仅对应ー个 Operator和Key的组合。Keyed State可以通过 Key Groups 进行管理，主要用于
当算子并行度发生变化时，自动重新分布Keyed State数据。在系统运行过程中，一个Keyed算子实例
可能运行一个或者多个Key Groups的keys。

**Operator State**

与 Keyed State不同的是， Operator State只和并行的算子实例绑定，和数据元素中的key无关，每个
算子实例中持有所有数据元素中的一部分状态数据。Operator State支持当算子实例并行度发生变化时
自动重新分配状态数据。

同时在 Flink中 Keyed State和 Operator State均具有两种形式，其中一种为**托管状态（ Managed**
**State**）形式，由 Flink Runtime中控制和管理状态数据，并将状态数据转换成为内存 Hash tables或
ROCKSDB的对象存储，然后将这些状态数据通过内部的接口持久化到 Checkpoints 中，任务异常时可
以通过这些状态数据恢复任务。另外一种是**原生状态（Raw State**）形式，由算子自己管理数据结构，
当触发 Checkpoint过程中， Flink并不知道状态数据内部的数据结构，只是将数据转换成bys数据存储
在 Checkpoints中，当从Checkpoints恢复任务时，算子自己再反序列化出状态的数据结构。
Datastream API支持使用 Managed State和 Raw State两种状态形式，在 Flink中推荐用户使用
**Managed State管理状态数据**，主要原因是 Managed State 能够更好地支持状态数据的重平衡以及更
加完善的内存管理。

### 广播状态

模式匹配应用程序如何处理用户操作和模式流

- processBroadcastElement() 为广播流的每个记录调用
- processElement() 为键控流的每个记录调用。它提供对广播状态的只读访问，以防止修改导致
  跨函数的并行实例的不同广播状态。
- onTimer() 在先前注册的计时器触发时调用。定时器可以在任何处理方法中注册，并用于执行计
  算或将来清理状态

```java
public class BroadCastDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        UserAction userAction1 = new UserAction(10001L, "login");
        UserAction userAction2 = new UserAction(10003L, "pay");
        UserAction userAction3 = new UserAction(10002L, "car");
        UserAction userAction4 = new UserAction(10001L, "logout");
        UserAction userAction5 = new UserAction(10003L, "car");
        UserAction userAction6 = new UserAction(10002L, "logout");
        DataStreamSource<UserAction> userActionDataStreamSource = env.fromElements(userAction1, userAction2, userAction3, userAction4, userAction5, userAction6);

        MyPattern myPattern = new MyPattern("login", "logout");
        DataStreamSource<MyPattern> patternDataStreamSource = env.fromElements(myPattern);

        KeyedStream<UserAction, Long> userActionLongKeyedStream = userActionDataStreamSource.keyBy(UserAction::getUserId);

        // 将模式广播到下游所有算子
        MapStateDescriptor<Void, MyPattern> mapStateDescriptor = new MapStateDescriptor<>("patterns", Types.VOID, Types.POJO(MyPattern.class));
        BroadcastStream<MyPattern> patternBroadcastStream = patternDataStreamSource.broadcast(mapStateDescriptor);

        SingleOutputStreamOperator<Tuple2<Long, MyPattern>> outputStreamOperator = userActionLongKeyedStream.connect(patternBroadcastStream).process(new PatternEvaluator());
        outputStreamOperator.print();


        env.execute();
    }
}


class PatternEvaluator extends KeyedBroadcastProcessFunction<Long, UserAction, MyPattern, Tuple2<Long, MyPattern>> {

    ValueState<String> valueState;

    @Override
    public void open(Configuration parameters) throws Exception {
        // 初始化keyState
        valueState = getRuntimeContext().getState(new ValueStateDescriptor<String>("lastAction", Types.STRING));
        super.open(parameters);
    }

    // 每来一个action数据，触发一次执行
    @Override
    public void processElement(UserAction userAction, ReadOnlyContext readOnlyContext, Collector<Tuple2<Long, MyPattern>> collector) throws Exception {
        ReadOnlyBroadcastState<Void, MyPattern> broadcastState = readOnlyContext.getBroadcastState(new MapStateDescriptor<>("patterns", Types.VOID, Types.POJO(MyPattern.class)));
        MyPattern myPattern = broadcastState.get(null);
        String preAction = valueState.value();
        if (preAction != null && myPattern != null) {
            if (myPattern.getFirstAction().equals(preAction) && myPattern.getSecondAction().equals(userAction.getUserAction())) {
                // 匹配成功
                collector.collect(new Tuple2<>(userAction.getUserId(),myPattern));
            } else {

            }
        }


        valueState.update(userAction.getUserAction());

    }

    // 每来一个pattern模式执行
    @Override
    public void processBroadcastElement(MyPattern myPattern, Context context, Collector<Tuple2<Long, MyPattern>> collector) throws Exception {
        BroadcastState<Void, MyPattern> broadcastState = context.getBroadcastState(new MapStateDescriptor<>("patterns", Types.VOID, Types.POJO(MyPattern.class)));
        broadcastState.put(null, myPattern);
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
class UserAction implements Serializable {
    private Long userId;
    private String userAction;
}
```

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class MyPattern {
    String firstAction;
    String secondAction;
}
// MyPattern 必须要求public
```

## 状态存储（扩展）

Flink 的一个重要特性就是有状态计算(stateful processing)。Flink 提供了简单易用的 API 来存储和获取
状态。但是，我们还是要理解 API 背后的原理，才能更好的使用。

### State 存储方式

Flink 为 state 提供了三种开箱即用的后端存储方式(state backend)：
1. Memory State Backend
2. File System (FS) State Backend
3. RocksDB State Backend

#### MemoryStateBackend

MemoryStateBackend 将工作状态数据保存在 taskmanager 的 java 内存中。key/value 状态和window 算子使用哈希表存储数值和触发器。进行快照时（checkpointing），生成的快照数据将和checkpoint ACK 消息一起发送给 jobmanager，jobmanager 将收到的所有快照保存在 java 内存中。
MemoryStateBackend 现在被默认配置成异步的，这样避免阻塞主线程的 pipline 处理。
MemoryStateBackend 的状态存取的速度都非常快，但是不适合在生产环境中使用。这是因为MemoryStateBackend 有以下限制：

- 每个 state 的默认大小被限制为 5 MB（这个值可以通过 MemoryStateBackend 构造函数设置）

- 每个 task 的所有 state 数据 (一个 task 可能包含一个 pipline 中的多个 Operator) 大小不能超过

  RPC 系统的帧大小(akka.framesize，默认 10MB)

- jobmanager 收到的 state 数据总和不能超过 jobmanager 内存

**MemoryStateBackend 适合的场景**

- 本地开发和调试
- 状态很小的作业

### FsStateBackend

FsStateBackend 需要配置一个 checkpoint 路径，例如“hdfs://namenode:40010/flink/checkpoints”
或者 “file:///data/flink/checkpoints”，我们一般配置为 hdfs 目录FsStateBackend 将工作状态数据保存在 taskmanager 的 java 内存中。进行快照时，再将快照数据写入上面配置的路径，然后将写入的文件路径告知 jobmanager。jobmanager 中保存所有状态的元数据信息(在 HA 模式下，元数据会写入 checkpoint 目录)。FSStateBackend 默认使用异步方式进行快照，防止阻塞主线程的 pipline 处理。可以通过FsStateBackend 构造函数取消该模式：`new FsStateBackend(path, false);`

FsStateBackend 适合的场景：

- 大状态、长窗口、大键值（键或者值很大）状态的作业
- 适合高可用方案

### RocksDBStateBackend

RocksDBStateBackend 也需要配置一个 checkpoint 路径，例如：
“hdfs://namenode:40010/flink/checkpoints” 或者 “file:///data/flink/checkpoints”，一般配置为 hdfs
路径。
RocksDB 是一种可嵌入的持久型的 key-value 存储引擎，提供 ACID 支持。由 Facebook 基于 levelDB
开发，使用 LSM 存储引擎，是内存和磁盘混合存储。
RocksDBStateBackend 将工作状态保存在 taskmanager 的 RocksDB 数据库中；checkpoint 时，RocksDB 中的所有数据会被传输到配置的文件目录，少量元数据信息保存在 jobmanager 内存中( HA模式下，会保存在 checkpoint 目录)。
RocksDBStateBackend 使用异步方式进行快照。
RocksDBStateBackend 的限制：

- 由于 RocksDB 的 JNI bridge API 是基于 byte[] 的，RocksDBStateBackend 支持的每个 key 或者
  每个 value 的最大值不超过 2^31 bytes((2GB))。
- 要注意的是，有 merge 操作的状态(例如 ListState)，可能会在运行过程中超过 2^31 bytes，导致
  程序失败

RocksDBStateBackend 适用于以下场景：

- 超大状态、超长窗口（天）、大键值状态的作业
- 适合高可用模式



## 并行度

>一个Flink程序由多个Operator组成(source、transformation和 sink)
>
>一个Operator由多个并行的Task(线程)来执行， 一个Operator的并行Task(线程)数目就被称为该
>Operator(任务)的并行度(Parallel)

### 指定方式

1. Operator Level（算子级别）(可以使用)
2. Execution Environment Level（Env级别）(可以使用)
3. Client Level(客户端级别,推荐使用)(可以使用)
4. System Level（系统默认级别,尽量不使用）
   在系统级可以通过设置flink-conf.yaml文件中的parallelism.default属性来指定所有执行环境的默认并
   行度

### 注意

1. 并行度的优先级：算子级别 > env级别 > Client级别 > 系统默认级别 (越靠前具体的代码并行度的优先
   级越高)
2. 如果source不可以被并行执行，即使指定了并行度为多个，也不会生效
3. 尽可能的规避算子的并行度的设置，因为并行度的改变会造成task的重新划分，带来shuffle问题，
4. 推荐使用任务提交的时候动态的指定并行度
5. slot是静态的概念，是指taskmanager具有的并发执行能力; parallelism是动态的概念，是指程序运行
   时实际使用的并发能力

## Flink Connector

### 源码理解

![image-20220116203718922](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220116203718922.png)

- Function : udf 处理数据的逻辑
- RichFunction: open/close管理函数声明周期的方法 ... RunTimeContext函数上下文
- SourceFunction: 提供自定义数据源能力，run 获取数据的方法
- ParallelSourceFunction: 并行处理数据源能力

open()  || run()方法

## Flink CEP

CEP 即Complex Event Processing - 复杂事件处理，Flink CEP 是在 Flink 中实现的复杂时间处理(CEP)
库。处理事件的规则，被叫做“模式”(Pattern)，Flink CEP 提供了 Pattern API，用于对输入流数据进行
复杂事件规则定义，用来提取符合规则的事件序列

​	Pattern API 大致分为三种：**个体模式**，**组合模式**，**模式组**

### 应用场景

- 实时监控
- 风险控制
- 营销广告等

### 基础

#### 定义

复合事件处理（Complex Event Processing，CEP）是一种基于动态环境中事件流的分析技术，事件在这里通常是有意义的状态变化，通过分析事件间的关系，利用过滤、关联、聚合等技术，根据事件间的时序关系和聚合关系制定检测规则，持续地从事件流中查询出符合要求的事件序列，最终分析得到更复杂的复合事件。

#### 特征

目标：从有序的简单事件流中发现一些高阶特征

输入：一个或多个简单事件构成的事件流

处理：识别简单事件之间的内在联系，多个符合一定规则的简单事件构成复杂事件

输出：满足规则的复杂事件

#### 功能

CEP用于分析低延迟、频繁产生的不同来源的事件流。CEP可以帮助在复杂的、不相关的时间流中找出有意义的模式和复杂的关系，以接近实时或准实时的获得通知或组织一些行为。
CEP支持在流上进行模式匹配，根据模式的条件不同，分为连续的条件或不连续的条件；模式的条件
允许有时间的限制，当条件范围内没有达到满足的条件时，会导致模式匹配超时。

功能：

① 输入的流数据，尽快产生结果；
② 在2个事件流上，基于时间进行聚合类的计算；
③ 提供实时/准实时的警告和通知；
④ 在多样的数据源中产生关联分析模式；
⑤ 高吞吐、低延迟的处理

市场上有多种CEP的解决方案，例如Spark、Samza、Beam等，但他们都没有提供专门的库支持。然而，Flink提供了专门的CEP库。

#### 主要组件

Flink为CEP提供了专门的Flink CEP library，它包含如下组件：Event Stream、Pattern定义、Pattern检测和生成Alert。

首先，开发人员要在DataStream流上定义出模式条件，之后Flink CEP引擎进行模式检测，必要时生成警告。![image-20220117083421671](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220117083421671.png)

### Pattern API

处理事件的规则，被叫作模式（Pattern）。Flink CEP提供了Pattern API用于对输入流数据进行复杂事件规则定义，用来提取符合规则的事件序列。

#### 分类

- 个体模式（Individual Patterns）：组成复杂规则的每一个单独的模式定义，就是个体模式
- 组合模式（Combining Patterns）：很多个体模式组合起来，就形成了整个的模式序列。模式序列必须以一个初始模式开始
- 模式组（（Group of Pattern）：将一个模式序列作为条件嵌套在个体模式里，成为一组模式

#### 个体模式

个体模式包括单例模式和循环模式。单例模式只接收一个事件，而循环模式可以接收多个事件

1. 量词：可以在一个个体模式后追加量词，也就是指定循环次数

2. 条件：

   每个模式都需要指定触发条件，作为模式是否接受事件进入的判断依据。CEP中的个体模式主要通过调用.where()、.or()和.until()来指定条件。按不同的调用方式

   - 简单条件：通过.where()方法对事件中的字段进行判断筛选，决定是否接收该事件
   - 组合条件：将简单的条件进行合并；or()方法表示或逻辑相连，where的直接组合就相当于与and
   - 终止条件：如果使用了oneOrMore或者oneOrMore.optional，建议使用.until()作为终止条件，以便清理状态
   - 迭代条件：能够对模式之前所有接收的事件进行处理；调用.where((value,ctx) => {…})，可以调用
     ctx.getEventForPattern(“name”)

#### 模式序列

1. 严格近邻: 所有事件按照严格的顺序出现，中间没有任何不匹配的事件，由.next()指定。例如对于模式“a next b”，事件序列“a,c,b1,b2”没有匹配

2. 宽松近邻：允许中间出现不匹配的事件，由.followedBy()指定。例如对于模式“a followedBy b”，事件序列“a,c,b1,b2”匹配为{a,b1}

3. 非确定性宽松近邻：进一步放宽条件，之前已经匹配过的事件也可以再次使用，由.followedByAny()指定。例如对于模式“a followedByAny b”，事件序列“a,c,b1,b2”匹配为{ab1}，{a,b2}。

   除了以上模式序列外，还可以定义“不希望出现某种近邻关系”：
   notNext()：不想让某个事件严格紧邻前一个事件发生。
   .notFollowedBy()：不想让某个事件在两个事件之间发生

**需要注意：**

- 所有模式序列必须以.begin()开始
- 模式序列不能以.notFollowedBy()结束
- “not”类型的模式不能被optional所修饰
- 可以为模式指定时间约束，用来要求在多长时间内匹配有效

#### 模式检测

指定要查找的模式序列后，就可以将其应用于输入流以检测潜在匹配。调用CEP.pattern()，给定输入
流和模式，就能得到一个PatternStream

#### 匹配事件的提取

创建PatternStream之后，就可以应用select或者flatSelect方法，从检测到的事件序列中提取事件
了。
select()方法需要输入一个select function作为参数，每个成功匹配的事件序列都会调用它。
select()以一个Map[String,Iterable[IN]]来接收匹配到的事件序列，其中key就是每个模式的名称，而
value就是所有接收到的事件的Iterable类型。

flatSelect通过实现PatternFlatSelectFunction实现与select相似的功能。唯一的区别就是flatSelect方
法可以返回多条记录，它通过一个Collector[OUT]类型的参数来将要输出的数据传递到下游

select process

#### 超时事件的提取

当一个模式通过within关键字定义了检测窗口时间时，部分事件序列可能因为超过窗口长度而被丢
弃；为了能够处理这些超时的部分匹配，select和flatSelect API调用允许指定超时处理程序

#### Flink CEP 开发流程

1. DataSource 中的数据转换为 DataStream；
2. 定义 Pattern，并将 DataStream 和 Pattern 组合转换为 PatternStream；
3. PatternStream 经过 select、process 等算子转换为 DataStraem；
4. 再次转换的 DataStream 经过处理后，sink 到目标库。

### NFA: 非确定有限自动机

FlinkCEP在运行时会将用户的逻辑转化成这样的一个NFA Graph (nfa对象)
所以有限状态机的工作过程，就是从开始状态，根据不同的输入，自动进行状态转换的过程。

![image-20220117233306231](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220117233306231.png)

上图中的状态机的功能，是检测二进制数是否含有偶数个 0。从图上可以看出，输入只有 1 和 0 两种。从 S1 状态开始，只有输入 0 才会转换到 S2 状态，同样 S2 状态下只有输入 0 才会转换到 S1。所以，二进制数输入完毕，如果满足最终状态，也就是最后停在 S1 状态，那么输入的二进制数就含有偶数个0。

### Flink CEP 案例

**pom.xml**

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-cep_2.12</artifactId>
    <version>1.11.1</version>
</dependency>
```

### 检测恶意登录

找出5秒内，连续登录失败的账号

>1. 数据源
>2. 做出watermark
>3. 根据id分组
>4. 做出模式pattern
>5. 在数据源进行模式匹配
>6. 提取匹配成功的数据

```java
public class CEPDemo1 {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        env.setParallelism(1);
        DataStreamSource<CEPLoginBean> streamSource = env.fromElements(
                new CEPLoginBean(1L, "fail", 1597905234000L),
                new CEPLoginBean(1L, "success", 1597905235000L),
                new CEPLoginBean(2L, "fail", 1597905236000L),
                new CEPLoginBean(2L, "fail", 1597905237000L),
                new CEPLoginBean(2L, "fail", 1597905238000L),
                new CEPLoginBean(3L, "fail", 1597905239000L),
                new CEPLoginBean(3L, "success", 1597905240000L)
        );
        SingleOutputStreamOperator<CEPLoginBean> watermarks = streamSource.assignTimestampsAndWatermarks(new WatermarkStrategy<CEPLoginBean>() {
            @Override
            public WatermarkGenerator<CEPLoginBean> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
                return new WatermarkGenerator<CEPLoginBean>() {
                    long maxTimeStamp = Long.MIN_VALUE;

                    @Override
                    public void onEvent(CEPLoginBean event, long eventTimestamp, WatermarkOutput output) {
                        maxTimeStamp = Math.max(event.getTs(), maxTimeStamp);
                    }

                    long maxOutOfOrderness = 500L;

                    @Override
                    public void onPeriodicEmit(WatermarkOutput output) {
                        output.emitWatermark(new Watermark(maxTimeStamp - maxOutOfOrderness));
                    }
                };
            }
        }.withTimestampAssigner((element, recordTimeStamp) -> element.getTs()));


        KeyedStream<CEPLoginBean, Long> keyed = watermarks.keyBy(CEPLoginBean::getId);

        Pattern<CEPLoginBean, CEPLoginBean> loginBeanPattern = Pattern.<CEPLoginBean>begin("start").where(new IterativeCondition<CEPLoginBean>() {
            @Override
            public boolean filter(CEPLoginBean cepLoginBean, Context<CEPLoginBean> context) throws Exception {
                return cepLoginBean.getLoginResult().equals("fail");
            }
        })
                .next("next").where(new IterativeCondition<CEPLoginBean>() {
                    @Override
                    public boolean filter(CEPLoginBean cepLoginBean, Context<CEPLoginBean> context) throws Exception {
                        return cepLoginBean.getLoginResult().equals("fail");
                    }
                })
                .within(Time.seconds(5));

        PatternStream<CEPLoginBean> pattern = CEP.pattern(keyed, loginBeanPattern);
        SingleOutputStreamOperator<Long> streamOperator = pattern.process(new PatternProcessFunction<CEPLoginBean, Long>() {
            @Override
            public void processMatch(Map<String, List<CEPLoginBean>> map, Context context, Collector<Long> collector) throws Exception {
                List<Long> start = map.get("start").stream().map(CEPLoginBean::getId).collect(Collectors.toList());
                collector.collect(start.get(0));
            }
        });

        streamOperator.print();
        env.execute();
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
class CEPLoginBean {
    private Long id;
    private String loginResult;
    private Long ts;

}
```

### 检测日活跃用户

找出24小时内，至少5次有效交易的用户

```java
public class DaoDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        env.setParallelism(1);
        DataStreamSource<ActiveUserBean> activeUserBeanDataStreamSource = env.fromElements(new ActiveUserBean("100XX", 0.0D, 1597905234000L),
                new ActiveUserBean("100XX", 100.0D, 1597905235000L),
                new ActiveUserBean("100XX", 200.0D, 1597905236000L),
                new ActiveUserBean("100XX", 300.0D, 1597905237000L),
                new ActiveUserBean("100XX", 400.0D, 1597905238000L),
                new ActiveUserBean("100XX", 500.0D, 1597905239000L),
                new ActiveUserBean("101XX", 0.0D, 1597905240000L),
                new ActiveUserBean("101XX", 100.0D, 1597905241000L));

        SingleOutputStreamOperator<ActiveUserBean> watermarks = activeUserBeanDataStreamSource.assignTimestampsAndWatermarks(new WatermarkStrategy<ActiveUserBean>() {
            @Override
            public WatermarkGenerator<ActiveUserBean> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
                return new WatermarkGenerator<ActiveUserBean>() {
                    long maxTimeStamp = Long.MIN_VALUE;
                    long maxOutofOrderness = 500L;

                    @Override
                    public void onEvent(ActiveUserBean event, long eventTimestamp, WatermarkOutput output) {
                        maxTimeStamp = Math.max(event.getTs(), maxTimeStamp);
                    }

                    @Override
                    public void onPeriodicEmit(WatermarkOutput output) {
                        output.emitWatermark(new Watermark(maxTimeStamp - maxTimeStamp));

                    }
                };
            }
        }.withTimestampAssigner((ele, recordTimestamp) -> ele.getTs()));

        KeyedStream<ActiveUserBean, String> keyedStream = watermarks.keyBy(ActiveUserBean::getUserId);

        Pattern<ActiveUserBean, ActiveUserBean> pattern = Pattern.<ActiveUserBean>begin("start").where(new IterativeCondition<ActiveUserBean>() {
            @Override
            public boolean filter(ActiveUserBean activeUserBean, Context<ActiveUserBean> context) throws Exception {
                return activeUserBean.getMoney() > 0;
            }
        }).timesOrMore(5).within(Time.hours(24));

        PatternStream<ActiveUserBean> userBeanPatternStream = CEP.pattern(keyedStream, pattern);

        SingleOutputStreamOperator<String> streamOperator = userBeanPatternStream.process(new PatternProcessFunction<ActiveUserBean, String>() {
            @Override
            public void processMatch(Map<String, List<ActiveUserBean>> map, Context context, Collector<String> collector) throws Exception {
                collector.collect(map.get("start").get(0).getUserId());
            }
        });
        streamOperator.print();

        env.execute();
    }
}


@Data
@AllArgsConstructor
@NoArgsConstructor
class ActiveUserBean {
    String userId;
    Double money;
    Long ts;
}
```

### 超时未支付

找出下单后10分钟没有支付的订单

```java
public class PayDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        DataStreamSource<PayEvent> streamSource = env.fromElements(
                new PayEvent(1L, "create", 1597905234000L),
                new PayEvent(1L, "pay", 1597905235000L),
                new PayEvent(2L, "create", 1597905236000L),
                new PayEvent(2L, "pay", 1597905237000L),
                new PayEvent(3L, "create", 1597905239000L)
        );

        SingleOutputStreamOperator<PayEvent> outputStreamOperator = streamSource.assignTimestampsAndWatermarks(new WatermarkStrategy<PayEvent>() {
            @Override
            public WatermarkGenerator<PayEvent> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
                return new WatermarkGenerator<PayEvent>() {
                    long maxTimeStamp = Long.MIN_VALUE;
                    long maxOutofOrderness = 500L;

                    @Override
                    public void onEvent(PayEvent event, long eventTimestamp, WatermarkOutput output) {
                        maxTimeStamp = Math.max(event.getTs(), maxTimeStamp);
                    }

                    @Override
                    public void onPeriodicEmit(WatermarkOutput output) {
                        output.emitWatermark(new Watermark(maxTimeStamp - maxOutofOrderness));
                    }
                };
            }
        }.withTimestampAssigner((ele, recordTimeStamp) -> ele.getTs()));

        KeyedStream<PayEvent, Long> keyedStream = outputStreamOperator.keyBy(PayEvent::getId);

        Pattern<PayEvent, PayEvent> pattern = Pattern.<PayEvent>begin("begin").where(new IterativeCondition<PayEvent>() {
            @Override
            public boolean filter(PayEvent payEvent, Context<PayEvent> context) throws Exception {
                return payEvent.getState().equals("create");
            }
        })
                .followedBy("pay")
                .where(new IterativeCondition<PayEvent>() {
                    @Override
                    public boolean filter(PayEvent payEvent, Context<PayEvent> context) throws Exception {
                        return payEvent.getState().equals("pay");
                    }
                }).within(Time.seconds(600));

        PatternStream<PayEvent> patternStream = CEP.pattern(keyedStream, pattern);
        OutputTag<PayEvent> output = new OutputTag<PayEvent>("output"){};

        SingleOutputStreamOperator<PayEvent> streamOperator = patternStream.select(output, new PatternTimeoutFunction<PayEvent, PayEvent>() {
            @Override
            public PayEvent timeout(Map<String, List<PayEvent>> map, long l) throws Exception {
                return map.get("begin").get(0);
            }
        }, new PatternSelectFunction<PayEvent, PayEvent>() {
            @Override
            public PayEvent select(Map<String, List<PayEvent>> map) throws Exception {
                return map.get("pay").get(0);
            }
        });
        DataStream<PayEvent> sideOutput = streamOperator.getSideOutput(output);
        sideOutput.print();

        env.execute();
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
class PayEvent {
    private long id;
    private String state;
    private long ts;
}
```

## Table API SQL

### 区别

Flink 本身是批流统一的处理框架，所以 Table API 和 SQL，就是批流统一的上层处理 API。
Table API 是一套内嵌在 Java 和 Scala 语言中的查询 API，它允许我们以非常直观的方式，
组合来自一些关系运算符的查询（比如 select、filter 和 join）。而对于 Flink SQL，就是直接可
以在代码中写 SQL，来实现一些查询（Query）操作。Flink 的 SQL 支持，基于实现了 SQL 标
准的 Apache Calcite（Apache 开源 SQL 解析工具）。
无论输入是批输入还是流式输入，在这两套 API 中，指定的查询都具有相同的语义，得到相同的结果。

Table Api Demo

```xml
<!--TABLE-API-->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table</artifactId>
    <version>1.11.1</version>
    <type>pom</type>
   <!--<scope>provided</scope>-->
</dependency>
<!-- Either... -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-java-bridge_2.11</artifactId>
    <version>1.11.1</version>
   <!--<scope>provided</scope>-->
</dependency>
<!-- or... -->
<!--        <dependency>-->
<!--            <groupId>org.apache.flink</groupId>-->
<!--            <artifactId>flink-table-api-scala-bridge_2.11</artifactId>-->
<!--            <version>1.11.1</version>-->
<!--           <scope>provided</scope>-->
<!--        </dependency>-->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner-blink_2.12</artifactId>
    <version>1.11.1</version>
   <!--<scope>provided</scope>-->
</dependency>
```

flink-table-api-java-bridge_2.1：桥接器，主要负责 table API 和 DataStream/DataSet API
的连接支持，按照语言分 java 和 scala。
flink-table-planner-blink_2.12：计划器，是 table API 最主要的部分，提供了运行时环境和生
成程序执行计划的 planner；
如果是生产环境，lib 目录下默认已 经有了 planner，就只需要有 bridge 就可以了
flink-table：flinktable的基础依赖

### Table API demo

```java
public class TableApiDemo1 {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);

        DataStreamSource<Tuple2<String, Integer>> dataStreamSource = env.addSource(new SourceFunction<Tuple2<String, Integer>>() {
            @Override
            public void run(SourceContext<Tuple2<String, Integer>> sourceContext) throws Exception {
                int num = 0;
                while (true) {
                    num++;
                    sourceContext.collect(new Tuple2<>("name" + num, num));
                    Thread.sleep(1000L);
                }
            }

            @Override
            public void cancel() {

            }
        });

        Table table = tEnv.fromDataStream(dataStreamSource, $("name"), $("age"));
        Table table1 = table.select($("name"),$("age")).filter($("age").mod(2).isEqual(0));
        DataStream<Tuple2<Boolean, Row>> retractStream = tEnv.toRetractStream(table1, Row.class);

        retractStream.print();
        env.execute();
    }
}
```

### SQL Demo

```java
public class SqlDemo2 {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);

        DataStreamSource<Tuple2<String, Integer>> dataStreamSource = env.addSource(new SourceFunction<Tuple2<String, Integer>>() {
            @Override
            public void run(SourceContext<Tuple2<String, Integer>> sourceContext) throws Exception {
                int num = 0;
                while (true) {
                    num++;
                    sourceContext.collect(new Tuple2<>("name" + num, num));
                    Thread.sleep(1000L);
                }
            }

            @Override
            public void cancel() {

            }
        });

     
        tEnv.createTemporaryView("newTable",dataStreamSource,$("name"),$("age"));
        Table table1 = tEnv.sqlQuery("select * from newTable where mod(age,2) = 0 ");
        DataStream<Tuple2<Boolean, Row>> retractStream = tEnv.toRetractStream(table1, Row.class);

        retractStream.print();
        env.execute();
    }
}
```

## 外部链接

### Connectors

![image-20220119081109376](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220119081109376.png)

![image-20220119081128085](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220119081128085.png)

### 从文件中获取 - CSV

```xml
<dependency>
<groupId>org.apache.flink</groupId>
<artifactId>flink-csv</artifactId>
<version>1.11.1</version>
</dependency>
```

```java
public class SourceFileDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);

        tEnv.connect(new FileSystem().path("F:\\gitPro\\first-flink\\data\\hi.txt"))
                .withFormat(new Csv())
                .withSchema(new Schema()
                        .field("name", DataTypes.STRING()))
                .createTemporaryTable("hi");

        String sql = "select * from hi ";
        Table table = tEnv.sqlQuery(sql);
        DataStream<Tuple2<Boolean, Row>> stream = tEnv.toRetractStream(table, Row.class);
        stream.print();
        env.execute();
    }
}
```

### Kafka

```java
public class SourceFromKafka {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);
        tEnv.connect(new Kafka()
                .version("universal")
                .topic("yw")
                .startFromEarliest()
                .property("bootstrap.servers", "192.168.10.118:9092"))
                .withFormat(new Csv())
                .withSchema(new Schema().field("age", DataTypes.STRING()))
                .createTemporaryTable("test");
        Table table = tEnv.sqlQuery("select * from test");
        DataStream<Tuple2<Boolean, Row>> tuple2DataStream = tEnv.toRetractStream(table, Row.class);
        tuple2DataStream.print();

        env.execute();
    }
}
```

### 输出表

#### 输出到文件

```java
public class SinkFileDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);
        DataStreamSource<Tuple2<String, Integer>> dataStreamSource = env.addSource(new SourceFunction<Tuple2<String, Integer>>() {
            @Override
            public void run(SourceContext<Tuple2<String, Integer>> sourceContext) throws Exception {
                int num = 0;
                while (true) {
                    num++;
                    sourceContext.collect(new Tuple2<>("name" + num, num));
                    Thread.sleep(1000L);
                }
            }

            @Override
            public void cancel() {

            }
        });
        Table table = tEnv.fromDataStream(dataStreamSource, $("name"), $("age"));


        tEnv.connect(new FileSystem().path("F:\\gitPro\\first-flink\\data\\output"))
                .withFormat(new Csv())
                .withSchema(new Schema()
                        .field("name", DataTypes.STRING()).field("age",DataTypes.INT()))
                .createTemporaryTable("tmpTable");

        table.executeInsert("tmpTable");

    }
}
```

#### 输出到kafka

```java
public class SinkKafkaDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);
        DataStreamSource<Tuple2<String, Integer>> dataStreamSource = env.addSource(new SourceFunction<Tuple2<String, Integer>>() {
            @Override
            public void run(SourceContext<Tuple2<String, Integer>> sourceContext) throws Exception {
                int num = 0;
                while (true) {
                    num++;
                    sourceContext.collect(new Tuple2<>("name" + num, num));
                    Thread.sleep(1000L);
                }
            }

            @Override
            public void cancel() {

            }
        });
        Table table = tEnv.fromDataStream(dataStreamSource, $("name"), $("age"));

        tEnv.connect(new Kafka()
                .version("universal")
                .topic("yw")
                .startFromEarliest()
                .property("bootstrap.servers", "192.168.10.118:9092"))
                .withFormat(new Csv())
                .withSchema(new Schema().field("name", DataTypes.STRING()).field("age", DataTypes.STRING()))
                .createTemporaryTable("test");

        table.executeInsert("test");

    }
}
```

## 作业提交

Flink的jar文件并不是Flink集群的可执行文件，需要经过转换之后提交给集群

转换过程：

1、在Flink Client中，通过反射启动jar中的main函数，生成Flink StreamGraph和JobGraph。将JobGraph提交给Flink集群。
2、Flink集群收到JobGraph后，将JobGraph翻译成ExecutionGraph，然后开始调度执行，启动成功之后开始消费数据
总结：
**Flink的核心执行流程就是，把用户的一系列API调用，转化为StreamGraph -- JobGraph --ExecutionGraph -- 物理执行拓扑（Task DAG）**

![image-20220119230352478](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220119230352478.png)

PipelineExecutor：流水线执行器：
是Flink Client生成JobGraph之后，将作业提交给集群运行的重要环节

![image-20220119230412396](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220119230412396.png)

Session模式：AbstractSessionClusterExecutor
Per-Job模式：AbstractJobClusterExecutor
IDE调试：LocalExecutor



### Session模式

作业提交通过: yarn-session.sh脚本
在启动脚本的时候检查是否已经存在已经启动好的Flink-Session模式的集群，然后在PipelineExecutor中，通过Dispatcher提供的Rest接口提交Flink JobGraph

Dispatcher为每一个作业提供一个JobMaser，进入到作业执行阶段

### Per-Job模式：一个作业一个集群，作业之间相互隔离。

在PipelineExecutor执行作业提交的时候，可以创建集群并将JobGraph以及所有需要的文件一起提交给
Yarn集群，在Yarn集群的容器中启动Flink Master（JobManager进程），进行初始化后，从文件系统
中获取JobGraph，交给Dispatcher，之后和Session流程相同。

流图

![image-20220119230517408](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220119230517408.png)

#### yarn提交作业流程图

![image-20220119232246090](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220119232246090.png)
