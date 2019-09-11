---
title: Apache Flink 说道系列- 如何在IDE中运行使用Java UDFs 的Python 作业
categories: Apache Flink 说道
tags: [Flink, 道德经, 持而盈之-不如其已]
---


# 开篇说道
《老子·第九章》："持而盈之，不如其已；揣而锐之，不可长保。金玉满堂，莫之能守；富贵而骄，自遗其咎。功成身退，天之道也" 在我看来老子说的这些内容，是告诉人们做事不能过于激进，不能贪得无厌，不能因为自己的得势就蛮横无理，任何事情要懂得知足常乐，懂得低调谦让，所谓谦受益满招损，过犹不及，物极必反！

如果你读了本篇文章能够有意识的今后不与人争长论短，那么将是极大的收获。如果你想听争长论短的坏处请评论区留言！

# 问题描述
Apache Flink 1.9版本版本中增加了Python Table API的支持，同时大家可以在Python API中使用Java的UDFs，但是这里有个问题在我的[直播](http://1t.click/BQx)中提到：目前官方没有提供如何在IDE中运行使用Java UDFs的 Python 作业的方式，原因是没有很好的机制将 Java UDFs的JAR集成到Python的执行环境中。

# 问题现象
如果大家在IDE中执行使用Java UDFs的Python作业会有怎样的现象呢？我还是以我直播中的代码为例:
```
t_env.register_java_function("len", "org.apache.flink.udf.UDFLength")
    elements = [(word, 1) for word in content.split(" ")]
    t_env.from_elements(elements, ["word", "count"]) \
         .group_by("word") \
         .select("word, len(word), count(1) as count") \
         .insert_into("Results")
```
上面我注册了一个Java UDF `org.apache.flink.udf.UDFLength`。然后在select中使用`len(word)`,如果我们直接执行上面代码，会有如下错误：

![](52A99F20-9648-4AB7-BA20-AA890797A9F0.png)
上面提示信息很明显，是找不到`org.apache.flink.udf.UDFLength`.

# 问题分析
上面的问题大家想必都清楚，就是`org.apache.flink.udf.UDFLength`没有在Classpath下面，所以只要我们想办法将`org.apache.flink.udf.UDFLength`所在的JAR添加到classpath就行可以。听了我直播分享的同学应该知道Apache Flink 1.9 Python API的架构，在执行Python的时候本质上是调用Java的API，Apache Flink Python API是如何在运行的时候将Flink Java API的JAR添加到Classpath下面的呢？
我们运行`python setup.py sdist` 之后，在生产的发布文件中，我们可以运行`tar -tvf dist/apache-flink-1.9.dev0.tar.gz |grep jar   `来查看Flink的Java JARs存放位置，如下图：

![](6F22FA6A-2892-4137-BA10-6D643C60DBEE.png)

我们发现只要在`deps/lib`目录下面的JARs在运行Python时候都是在Classpath下面的。所以我们只要想办法将我们自己定义的UDF放到deps/lib目录下就可以了。那么怎样才能将我们的JAR放到Classpath下面呢？
## 解决方案
如果你在安装pyflink之前可以将你需要的JAR放到 `build-target/lib`下面，然后执行`python setup.py sdist`进行打包和进行安装`pip install dist/*.tar.gz`这样你的的UDF的JAR也会在Classpath下面。

如果你已经安装好了，你也可以直接将你的JAR包拷贝到pip安装的目录，比如：

![](9C73FBBB-80AD-45DA-91E5-21436A7811C2.png)

我们可以直接将需要的JARs拷贝到你所使用的phython环境的`$PY_HOME/site-packages/pyflink/lib/`目录下。

# 未来规划
在Apache Flink 1.10版本我们会将这个过程自动化，使用一种比较方便的方式优化用户的使用体验！

# 小结
本篇有针对性的为大家讲解了如何在IDE中运行使用Java UDF的 Python 作业。并且 开篇说道 部分为大家分享了老子"持而盈之，不如其已"经典语句，并建议大家今后有意识的不要与人争长论短，谨记 谦受益满招损，努力做到谦逊低调，营造欢乐的交际氛围，最后祝福大家在学习Apache Flink Python API的同时，生活工作开心快乐！

# 关于评论
如果你没有看到下面的评论区域，请进行翻墙！
![](comment.png)
