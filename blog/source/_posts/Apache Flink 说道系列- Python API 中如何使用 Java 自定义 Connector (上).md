---
title: Apache Flink 说道系列- Python API 中如何使用 Java 自定义 Connector (上)
categories: Apache Flink 说道
tags: [Flink, Python, Connector, 道德经, 孰能浊以静之徐清, 孰能安以动之徐生]
---

# 开篇说道
老子说"孰能浊以静之徐清? 孰能安以动之徐生?",大概是说谁能像浑浊的水流一样停止流动，安静下来慢慢变得澄清？谁可以像草木那样保持长时的静寂，却萌动生机而不息？对于与世为人来说，这句话是指谁能在浊世中保持内心的清净？只有智者可以静看花开花落，就像是莲花，出淤泥而不染。"孰能安以动之徐生" 一句则说在保持安静的同时，还要激发着无穷的生机，顺应自然的同时，默默改变这人世界。老子并不提倡举世皆浊我独清的态度，而是提倡“混兮其若浊”、“和其光，同其尘”，也就是说不要认为自己了不起，每个人的行为都有其道理，我们应该理解他人和接纳他人，和他人一样，表面看起来有点浑浑噩噩，但内心要"静",要"清"，就像一杯含有沙子的黄河水，只有在安静的状态下，才能慢慢变清，我们内心要从纷繁复杂的"浊"中理清头绪，变得清醒。但静静地弄清楚了还不够，还需要动起来，给外界反馈，发挥自己积极的作用。但切记要安静，即心态要平稳，如果急于求成，慌里慌张地做事情，那就会变成真的"浊"，变得稀里糊涂了。老子建议做一切事都要不急不躁，不“乱”不“浊”，一切要悠然"徐生"，水到渠成。
![](hehua.png)

# 问题
为什么要聊在Python API 中如何使用 Java 自定义 Source/Sink？目前在Apache Flink中集成了一些常用的connector，比如Kafka,elasticsearch6,filesystem等常用的外部系统。但这仍然无法满足需求各异的用户。比如用户用Kafka但是可能用户使用的Kafka版本Flink并没有支持（当然社区也可以添加支持，我们只是说可能情况），还比如用户想使用自建的内部产品，那一定无法集成用户的内部产品到Flink社区，这时候就要用户根据Flink自定义Connector的规范开发自己的Connector了。那么本篇我们就要介绍如果你已经完成了自定义connector，如何在Python Table API中进行使用。

# 如何自定义Java Connector
Apache Flink Table API/SQL中采用了Service Provider Interfaces (SPI)方式查找系统内置的和用户自定义的Connector。简单说任何Connector都需一个对应的[TableFactory](https://github.com/apache/flink/blob/release-1.9.0/flink-table/flink-table-common/src/main/java/org/apache/flink/table/factories/TableFactory.java)的实现，其核心方法是`requiredContext`,该方法定义对应Connector的唯一特征属性，connector.type，connector.version等。Connector的类型和版本就唯一决定了一个具体的Connector实现。当然任何Connector都要实现具体的[TableSource](https://github.com/apache/flink/blob/release-1.9.0/flink-table/flink-table-common/src/main/java/org/apache/flink/table/sources/TableSource.java)或[TableSink](https://github.com/apache/flink/blob/release-1.9.0/flink-table/flink-table-common/src/main/java/org/apache/flink/table/sinks/TableSink.java)。关于自定义Java Connector我们会在一篇完整的Blog进行详细叙述，本篇重点集中在如果已经完成了自定义的Connector，我们如何在Python Table API中使用。

# Python Table API中注册Connector
我们知道Flink Python Table API中采用Py4j的方式将Python Table API转换为Java Table API，而Java Table API中采用SPI的方式发现和查找Connector，也就是说Pytohn API需要定义一种方式能够告诉Java Table API 用户自定义的Connector信息。对于一个自定义Connector有两个非常重要的信息：
 
 - CustomConnectorDescriptor - Connector自有的属性定义；
 - CustomFormatDescriptor - Connector的数据格式定义；

所以在Python Table API中我们利用 `CustomConnectorDescriptor` 和`CustomFormatDescriptor`来描述用户自定义Connector。

## 自定义Connector类加载到Classpath

用户要使用自定义的Connector，那么第一步是将用户自定义的类要加载到Classpath中，操作方式与Python API中使用Java UDF一致，详见：《[Apache Flink 说道系列- 如何在IDE中运行使用Java UDFs 的Python 作业](http://1t.click/BSa)》

## Python Table API 描述自定义Connector
我们这里以自定义Kafka Source为例，用户编写Python Table API如下：
* 定义Connector属性
我们需要利用`CustomConnectorDescriptor`来描述Connector属性，如下：
```
custom_connector = CustomConnectorDescriptor('kafka', 1, True) \
        .property('connector.topic', 'user') \
        .property('connector.properties.0.key', 'zookeeper.connect') \
        .property('connector.properties.0.value', 'localhost:2181') \
        .property('connector.properties.1.key', 'bootstrap.servers') \
        .property('connector.properties.1.value', 'localhost:9092') \
        .properties({'connector.version': '0.11', 'connector.startup-mode': 'earliest-offset'})
```
上面属性与内置的Kafka 0.11版本一致。这里是示例如何利用`CustomConnectorDescriptor`描述自定义Connector属性。

* 定义Connector的数据格式
我们需要利用`CustomFormatDescriptor`来描述Connector数据格式，如下：
```
 # the key is 'format.json-schema'
    custom_format = CustomFormatDescriptor('json', 1) \
        .property('format.json-schema',
                  "{"
                  "  type: 'object',"
                  "  properties: {"
                  "    a: {"
                  "      type: 'string'"
                  "    },"
                  "    b: {"
                  "      type: 'string'"
                  "    },"
                  "    c: {"
                  "      type: 'string'"
                  "    },"
                  "    time: {"
                  "      type: 'string',"
                  "      format: 'date-time'"
                  "    }"
                  "  }"
                  "}") \
        .properties({'format.fail-on-missing-field': 'true'})
```
这个数据格式是沿用了在 《[Apache Flink 说道系列- Python API 中如何使用 Kafka](http://1t.click/aej7)》一篇中的示例。

* 定义Table Source
完成Connector属性的描述和Connector数据格式描述之后，我们就可以同使用内置Connector一样定义Table Source了。如下：
```
    st_env \
        .connect(custom_connector) \
        .with_format(custom_format) \
        .with_schema(...) \
        .register_table_source("source")
```
上面`with_schema(...)`是定义数据流入Flink内部后的Table 数据结构。

* 定义查询逻辑
```
 st_env.scan("source")
    .window(Tumble.over("2.rows")
        .on("proctime").alias("w")) \
    .group_by("w, a") \
    .select("a, max(b)")
    .insert_into("result")
```

查询逻辑是一个简单的每2行作为一个Tumble Window。计算按 `a` 分组统计 `b` 的最大值。


# 完整示例
完整的自定义Kafka Source示例:

```
if __name__ == '__main__':
    custom_connector = CustomConnectorDescriptor('kafka', 1, True) \
        .property('connector.topic', 'user') \
        .property('connector.properties.0.key', 'zookeeper.connect') \
        .property('connector.properties.0.value', 'localhost:2181') \
        .property('connector.properties.1.key', 'bootstrap.servers') \
        .property('connector.properties.1.value', 'localhost:9092') \
        .properties({'connector.version': '0.11', 'connector.startup-mode': 'earliest-offset'})

    # the key is 'format.json-schema'
    custom_format = CustomFormatDescriptor('json', 1) \
        .property('format.json-schema',
                  "{"
                  "  type: 'object',"
                  "  properties: {"
                  "    a: {"
                  "      type: 'string'"
                  "    },"
                  "    b: {"
                  "      type: 'string'"
                  "    },"
                  "    c: {"
                  "      type: 'string'"
                  "    },"
                  "    time: {"
                  "      type: 'string',"
                  "      format: 'date-time'"
                  "    }"
                  "  }"
                  "}") \
        .properties({'format.fail-on-missing-field': 'true'})

    s_env = StreamExecutionEnvironment.get_execution_environment()
    s_env.set_parallelism(1)
    s_env.set_stream_time_characteristic(TimeCharacteristic.ProcessingTime)
    st_env = StreamTableEnvironment.create(s_env)
    result_file = "/tmp/custom_kafka_source_demo.csv"

    if os.path.exists(result_file):
        os.remove(result_file)

    st_env \
        .connect(custom_connector) \
        .with_format(
        custom_format
    ) \
        .with_schema(  # declare the schema of the table
        Schema()
            .field("proctime", DataTypes.TIMESTAMP())
            .proctime()
            .field("a", DataTypes.STRING())
            .field("b", DataTypes.STRING())
            .field("c", DataTypes.STRING())
    ) \
        .in_append_mode() \
        .register_table_source("source")

    st_env.register_table_sink("result",
                               CsvTableSink(["a", "b"],
                                            [DataTypes.STRING(),
                                             DataTypes.STRING()],
                                            result_file))

    st_env.scan("source").window(Tumble.over("2.rows").on("proctime").alias("w")) \
        .group_by("w, a") \
        .select("a, max(b)").insert_into("result")

    st_env.execute("custom kafka source demo")

```

详细代码见:
[https://github.com/sunjincheng121/enjoyment.code/blob/master/myPyFlink/enjoyment/udc/CustomKafkaSourceDemo.py](https://github.com/sunjincheng121/enjoyment.code/blob/master/myPyFlink/enjoyment/udc/CustomKafkaSourceDemo.py)

* 运行CustomKafkaSourceDemo
上面示例的运行需要一个Kafka的集群环境，集群信息需要和`CustomConnectorDescriptor`中描述的一致，比如`connector.topic`是`user`， `zookeeper.connect`, `localhost:2181`, `bootstrap.servers`是 `localhost:9092`。 这些信息与《[Apache Flink 说道系列- Python API 中如何使用 Kafka](http://1t.click/aej7)》一篇中的Kafka环境一致，大家可以参考初始化自己的Kafka环境。

* 准备测试数据
运行之前大家要向 `user` Topic中写入一些测试数据，
```
if __name__ == '__main__':
    topic = 'user'
    topics = list_topics()
    if topic in topics:
        delete_topics(topic)

    create_topic(topic)
    msgs = [{'a': 'a', 'b': 1, 'c': 1, 'time': '2013-01-01T00:14:13Z'},
            {'a': 'b', 'b': 2, 'c': 2, 'time': '2013-01-01T00:24:13Z'},
            {'a': 'a', 'b': 3, 'c': 3, 'time': '2013-01-01T00:34:13Z'},
            {'a': 'a', 'b': 4, 'c': 4, 'time': '2013-01-01T01:14:13Z'},
            {'a': 'b', 'b': 4, 'c': 5, 'time': '2013-01-01T01:24:13Z'},
            {'a': 'a', 'b': 5, 'c': 2, 'time': '2013-01-01T01:34:13Z'}]
    for msg in msgs:
        send_msg(topic, msg)
    # print test data
    get_msg(topic)
```

完整代码可以参考：[https://github.com/sunjincheng121/enjoyment.code/blob/master/myPyFlink/enjoyment/kafka/prepare_data.py](https://github.com/sunjincheng121/enjoyment.code/blob/master/myPyFlink/enjoyment/kafka/prepare_data.py)

* 运行代码，并查看最终计算结果
![338566533211a283d7a715ba32009fd2](Apache Flink 说道系列- Python API 中如何使用 Java 自定义 Connector (上).resources/646B6616-4AA7-4A04-AAE8-A6D1EE006535.png)

# 待续
上面示例是在假设用户已经完成了 Java 自定义Connector的基础之上进行介绍如何利用Python Table API使用自定义Connector，但并没有详细介绍用户如何自定义Java Connector。所以后续我会为大家单独在一篇Blog中介绍Apache Flink 如何自定义Java Connector。[官方文档](https://ci.apache.org/projects/flink/flink-docs-master/dev/table/sourceSinks.html)

# 小结
本篇内容篇幅比较短小，具体介绍了Python Table API如何使用自定义Java Connector。其中Kafka环境相关内容需要结合《[Apache Flink 说道系列- Python API 中如何使用 Kafka](http://1t.click/aej7)》一篇一起阅读。同时在开篇说道中简单分享了老子关于"孰能浊以静之徐清? 孰能安以动之徐生?"的人生智慧，希望对大家有所启迪！

# 关于评论
如果你没有看到下面的评论区域，请进行翻墙！
![](comment.png)

