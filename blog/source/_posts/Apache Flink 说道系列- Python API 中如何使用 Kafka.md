---
title: Apache Flink 说道系列- Python API 中如何使用 Kafka
categories: Apache Flink 说道
tags: [Flink, 道德经, 圣人为腹不为目]
---

# 开篇说道
老子说"圣人为腹不为目"，核心意思是说看问题要看本质，不要着眼于外表。交人处事也是一样，不要看其外表和语言要看内心和本质。这里和大家分享一个故事，"九方皋(Gao)相马"。

春秋时期的伯乐是识别千里马的大师，在他上了年纪之后 秦穆公 对他说:"在你的子孙里面给我推荐一个能够识别千里马的人才吧？"，伯乐说:"在我的子孙中没有这样的人才，不过我有个朋友，名叫 九方皋 是个相马的高手，我推荐给您吧"。

于是 秦穆公 召见 九方皋，让他去出巡千里马，过了数月九方皋回来禀告说:"我已经找到千里马了，就在沙丘那个地方"。于是 秦穆公 兴奋的问:"那是一匹什么样子的马啊"？九方皋 回答:"是一匹黄色的母马"。
不过 秦穆公 派人把那匹马找来发现是一匹黑色的公马。秦穆公很生气，就派人把伯乐叫来说:"你推荐的相马的人实在糟透了，他连马的颜色和公母都分不清，怎么能认识千里马呢"？ 伯乐听了内心无比佩服的对 秦穆公 说:"这就是他比我还高明的地方，他重视的是马的精神，忽略了马的外表，重视马的内在品质，而忽略了马的颜色和雌雄，九方皋只关注他所关注的重点，而忽略不重要的特征，这样才真的能找到难寻的好马啊"。后来经过检验这的确是一匹天下少有的千里马。


所以"圣人为腹不为目"，注重本质而忽略外在。祝愿大家在日常生活和工作中能看清事情的内在和深层次的所在，而后做出你最正确的决定。

# 如何创建Source
Apache Flink 1.9 有两种方式创建Source：
* Table Descriptor
* Table Source

## Table Descriptor
利用Descriptor方式创建Source是比较推荐的做法，以简单的CSV示例如下：
```
t_env.connect(FileSystem()
    .path(source_path)
    .with_format(OldCsv()
        .field("word",  DataTypes.STRING()))
    .with_schema(Schema()
        .field("word", DataTypes.STRING()))      
    .register_table_source("source")

table = t_env.scan("source")
```

## Table Source
创建TableSource的方式也可以完成Source的创建，如下:
```
csv_source = CsvTableSource(
    source_path, 
    ["word"],  
    [DataTypes.STRING()])
    
t_env.register_table_source("source", csv_source)

table = t_env.scan("source")
```

# 如何创建Sink
和创建Source一样，Apache Flink 1.9 有两种方式创建Sink：
* Table Descriptor
* Table Sink

## Table Descriptor
利用Descriptor方式创建Sink是比较推荐的做法，以简单的CSV示例如下：
```
t_env.connect(FileSystem()
    .path(result_path)
    .with_format(OldCsv()
        .field("word",  DataTypes.STRING()))
    .with_schema(Schema()
        .field("word", DataTypes.STRING()))      
    .register_table_sink("results")

result_table.insert_into("results")
```

## Table Sink
创建TableSink的方式也可以完成Sink的创建，如下:
```
csv_sink = CsvTableSink(
    ["word"],  
    [DataTypes.STRING()])
    
t_env.register_table_sink("results", csv_sink)
result_table.insert_into("results")
```

## 关于Format和Schema
不论是Source和Sink在利用Table Descriptor进行实例化的时候都是需要设置Format和Schema的，那么什么是Format和Schema呢？
* Format - 是描述数据在外部存储系统里面的数据格式。
* Schema - 是外部数据加载到Flink系统之后，形成Table的数据结构描述。

# Kafka connector JAR加载
如《Apache Flink 说道系列- 如何在IDE中运行使用Java UDFs 的Python 作业》所描述的问题一样，我们需要显示的将Kafka connector JARs添加到classpath下面。所以在源码构建PyFlink发布包时候，需要在完成源码编译之后，将Kafka相关的JARs复制到 `build-target/lib`下面。并且需要对应的Format的JARs。以`kafka-0.11`和`Json` Format为例：

Copy Kafka相关JARs
```
cp flink-connectors/flink-sql-connector-kafka-0.11/target/flink-sql-connector-kafka-0.11_*-SNAPSHOT.jar build-target/lib
```

Copy Json Format相关JARs
```
cp flink-formats/flink-json/target/flink-json-*-SNAPSHOT-sql-jar.jar build-target/lib
```
构建PyFlink发布包并安装：
```
cd flink-python; python setup.py sdist 
pip install dist/*.tar.gz --user
```
环境安装的详细内容可以参考《Apache Flink 说道系列- Python Table API 开发环境搭建》。

# 安装Kafka环境
如果你已经有Kafka的集群环境，可以忽略本步骤，如果没有，为了完成本篇的测试，你可以部署一个简单的Kafka环境。
* 下载Kafka安装包
```
wget https://archive.apache.org/dist/kafka/0.11.0.3/kafka_2.11-0.11.0.3.tgz
```
* 解压
```
tar zxvf kafka_2.11-0.11.0.3.tgz
```
* 启动Zookeeper
```
cd kafka_2.11-0.11.0.3; bin/zookeeper-server-start.sh config/zookeeper.properties

最终输出：
[2019-08-28 08:47:16,437] INFO maxSessionTimeout set to -1 (org.apache.zookeeper.server.ZooKeeperServer)
[2019-08-28 08:47:16,478] INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)
```

* 支持删除Topic
  要支持删除Topic，我们需要修改server.properties配置，将`delete.topic.enable=true` 打开。
  
* 启动Kafka
```
bin/kafka-server-start.sh config/server.properties

最终输出：
[2019-08-28 08:49:20,280] INFO Kafka commitId : 26ddb9e3197be39a (org.apache.kafka.common.utils.AppInfoParser)
[2019-08-28 08:49:20,281] INFO [Kafka Server 0], started (kafka.server.KafkaServer)
```
如上过程我们完成了简单Kafka的环境搭建。更多细节参考Kafka[官方文档](https://kafka.apache.org/quickstart)
# 编写一个简单示例
这个示例我们从Kafka读取数据，然后创建一个简单的tumble window，最终将计算结果写到CSV文件系统里面。

## 数据结构
我们在Kafka中以Json的数据格式进行存储，假设有如下数据结构：
```
{
	"type": "object",
	"properties": {
		"a": {
			"type": "string"
		},
		"b": {
			"type": "string"
		},
		"c": {
			"type": "string"
		},
		"time": {
			"type": "string",
			"format": "date-time"
		}
	}
}
```
如上数据结构中time是为了演示event-time window，其他字段没有特殊含义。

## 数据准备
我们按如上数据结构来向Kafka中写入测试数据，我用python操作Kafka，需要安装Kafka的PythonAPI，如下：
```
pip install kafka-python
```
将测试数据写入Kafka：
```
# -*- coding: UTF-8 -*-

from kafka import KafkaProducer
from kafka import KafkaAdminClient
from kafka import KafkaConsumer
from kafka.admin import NewTopic
import json

# 写入消息
def send_msg(topic='test', msg=None):
    producer = KafkaProducer(bootstrap_servers='localhost:9092',
                             value_serializer=lambda v: json.dumps(v).encode('utf-8'))
    if msg is not None:
        future = producer.send(topic, msg)
        future.get()

# 读取消息
def get_msg(topic='test'):
    consumer = KafkaConsumer(topic, auto_offset_reset='earliest')
    for message in consumer:
        print(message)

# 查询所有Topic
def list_topics():
    global_consumer = KafkaConsumer(bootstrap_servers='localhost:9092')
    topics = global_consumer.topics()
    return topics
# 创建Topic
def create_topic(topic='test'):
    admin = KafkaAdminClient(bootstrap_servers='localhost:9092')
    topics = list_topics()
    if topic not in topics:
        topic_obj = NewTopic(topic, 1, 1)
        admin.create_topics(new_topics=[topic_obj])

# 删除Topic
def delete_topics(topic='test'):
    admin = KafkaAdminClient(bootstrap_servers='localhost:9092')
    topics = list_topics()
    if topic in topics:
        admin.delete_topics(topics=[topic])


if __name__ == '__main__':
    topic = 'mytopic'
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
源码:[prepare_data.py](https://github.com/sunjincheng121/enjoyment.code/blob/master/myPyFlink/enjoyment/kafka/prepare_data.py)

运行上面的代码，控制台会输出如下：
![](7A1AA068-AA3C-4439-898D-5DE507CC24C6.png)
如上信息证明我们已经完成了数据准备工作。

# 开发示例
## 创建Kafka的数据源表
我们以Table Descriptor的方式进行创建：
```
st_env.connect(Kafka()
     .version("0.11")
     .topic("user")
     .start_from_earliest()
     .property("zookeeper.connect", "localhost:2181")
            .property(
            "bootstrap.servers", "localhost:9092"))
     .with_format(Json()
            .fail_on_missing_field(True)
            .json_schema(
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
                "}"
             )
         )
    .with_schema(Schema()
        .field("rowtime", DataTypes.TIMESTAMP())
        .rowtime(Rowtime()
            .timestamps_from_field("time")
            .watermarks_periodic_bounded(60000))
            .field("a", DataTypes.STRING())
            .field("b", DataTypes.STRING())
            .field("c", DataTypes.STRING())
         )
     .in_append_mode()
     .register_table_source("source")
```

## 创建CSV的结果表
我们以创TableSink实例的方式创建Sink表：
```
st_env.register_table_sink(
"result", 
CsvTableSink(
    ["a", "b"],
    [DataTypes.STRING(),
    DataTypes.STRING()],
    result_file)
)
```

## 创1小时的Tumble窗口聚合
```
st_env.scan("source")
    .window(Tumble
        .over("1.hours").on("rowtime").alias("w"))
    .group_by("w, a")
    .select("a, max(b)")
    .insert_into("result")
```

完整代码 [tumble_window.py](https://github.com/sunjincheng121/enjoyment.code/blob/master/myPyFlink/enjoyment/kafka/tumble_window.py)
运行源码，由于我们执行的是Stream作业，作业不会自动停止，我们启动之后，执行如下命令查看运行结果：
```
cat /tmp/tumble_time_window_streaming.csv
```
![](02C1D4A6-7290-408E-832A-1FBDCDED75E5.png)

# 小结
本篇核心是向大家介绍在Flink Python Table API如何读取Kafka数据，并以一个Event-Time的TumbleWindow示例结束本篇介绍。开篇说道部分想建议大家要做到 "圣人为腹不为目"，任何事情都要追求其本质。愿你在本篇中有所收获！

# 关于评论
如果你没有看到下面的评论区域，请进行翻墙！
![](comment.png)
