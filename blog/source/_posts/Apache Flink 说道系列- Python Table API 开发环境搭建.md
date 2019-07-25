---
title: Apache Flink 说道系列- Python Table API 开发环境搭建
date: 2019-07-25 21:05:15
categories: Apache Flink 说道
tags: [Flink, 道德经, 曲则全]
---
# 开篇说道 - 曲则全
"曲则全，枉则直，洼则盈，敝则新，少则得，多则惑" 语出《道德经》第二十二章，整句充分反映了老子的辩证思想，老子用曲则全，枉则直，洼则盈，敝则新，少则得和多则惑来讲述了"道"顺其自然的理论。

所谓的顺其自然，有很多方面，比如前面一篇讲的"上善若水"，水自高而下的流淌也是顺其自然的体现，如果不顺其自然,而是阻止水的自上而下的流淌会发生什么？那就是 灾难，水灾。大家知道在 大禹治水 之前人类是如何处理洪水的？"拦截" 无目的的随处拦截，不是拦截不好，而是只拦截不疏通就违背自然之道了。再想想大禹治水，则是在顺应自然的同时，在疏通水道的同时在恰当的地方进行拦截，使得在不违背水自上而下的自然本性的同时合理影响水流的方向，进而造福人类。

那么本篇的"曲则全，枉则直，洼则盈，敝则新，少则得，多则惑"如何理解呢？直译一下就是："弯曲才可保全，委屈才可伸展，低洼将可充满，敝旧将可更新，少取反而获得，多取反而迷惑"。其实不难理解，比如:"洼则盈", "洼"也就是低洼，"盈"就是满，只有低洼的的地势才能赢得水流，也就是只有空的杯子才能装的进水。满的杯子很难再容下其他物品。其实老子无时无刻不在论证为人之道，谦卑将自己放空，才能容纳他人，聚贤聚德。下面我们再以一个简短的故事解释"曲则全"来结束本篇的论道：

春秋时期，齐国君主齐景公非常喜欢捕鸟，当时有个名叫 烛雏 的大臣专门为其捕鸟。但有一次由于 烛雏 的疏忽将捕来的鸟弄飞了。这使得齐景公勃然大怒，要杀了 烛雏。当时 晏子 也附和齐景王对 烛雏 说：你有三大罪状：1. 你的疏忽把鸟放跑了，这是第一罪状，2. 因为鸟飞了使得大王要去杀人，使大王背上了喜欢杀人的坏名声，这是你第二大罪状，3. 这件事情传出去天下人会认为大王把鸟看的比人命还重要，破坏了大王的威望，这是你第三大罪状。所以，大王请将 烛雏 处死吧？可想而知 齐景王并没有完全失去理智，虽然 晏子 没有直接劝阻他不要杀死 烛雏，但他听出了 晏子 的用意，所以自然赦免了 烛雏。

所以真正的智者会顺势而为，见机行事，无法直言的时候就要学会迂回，这也是对老子"曲则全"理论的典型运用！

# 概要
欲善其事必利其器，要想利用Apache Flink Python Table API进行业务开发，首先要先搭建一个开发环境。本文将细致的叙述如何搭建开发环境，最后在编写一个简单的WordCount示例程序。

# 依赖环境

* JDK 1.8+ (1.8.0_211)
* Maven 3.x (3.2.5)
* Scala 2.11+ (2.12.0)
* Python 2.7+ (2.7.16)
* Git 2.20+ (2.20.1)

大家如上依赖配置如果有问题，可以在评论区留言。

# 安装PyFlink
我们要想利用Apache Flink Python API 进行业务开发，需要将PyFlink发布包进行安装。目前PyFlink并没有发布到Python仓库，比如著名的PyPI(正在社区讨论，[详见](http://apache-flink-mailing-list-archive.1008284.n3.nabble.com/DISCUSS-Publish-the-PyFlink-into-PyPI-td30095.html))，所以我们需要从源码构建。
## 源码构建发布包
### 下载flink源代码
```
git clone git@github.com:apache/flink.git
```
签出release-1.9分支(1.9版本是Apache Flink Python Table API的第一个版本）
```
git fetch origin release-1.9
git checkout -b release-1.9 origin/release-1.9
```
输出如下证明已经签出了release-1.9代码：
```
Branch 'release-1.9' set up to track remote branch 'release-1.9' from 'origin'.
Switched to a new branch 'release-1.9'
```
### 构建Java二进制包
查看我们在relese-1.9分支并进行源码构建发布包（JAVA）
```
mvn clean install -DskipTests -Dfast
```
上面命令会执行一段时间(时间长短取决于你机器配置),最终输出如下信息，证明已经成功的完成发布包的构建：
```
...
...
[INFO] flink-ml-lib ....................................... SUCCESS [  0.190 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 14:47 min
[INFO] Finished at: 2019-07-25T17:38:06+08:00
[INFO] Final Memory: 537M/2085M
[INFO] ------------------------------------------------------------------------
```
在flink-dist/target/下的lib目录和opt目录中flink-python*.jar 和 flink-table*.jar 都是PyFlink必须依赖的JARs, 在下面构建PyFlink包的时候，会将这些依赖JRAs也包含在里面。

### 构建PyFlink发布包
一般情况我们期望以pip install的方式安装python的类库，我们要想安装PyFlink的类库，也需要构建可用于pip install的发布包。执行如下命令：
```
cd flink-python; python setup.py sdist 
```
最终输出如下信息，证明是成功的：
```
...
...
copying pyflink/table/types.py -> apache-flink-1.9.dev0/pyflink/table
copying pyflink/table/window.py -> apache-flink-1.9.dev0/pyflink/table
copying pyflink/util/__init__.py -> apache-flink-1.9.dev0/pyflink/util
copying pyflink/util/exceptions.py -> apache-flink-1.9.dev0/pyflink/util
copying pyflink/util/utils.py -> apache-flink-1.9.dev0/pyflink/util
Writing apache-flink-1.9.dev0/setup.cfg
creating dist
Creating tar archive
removing 'apache-flink-1.9.dev0' (and everything under it)
```
在dist目录的apache-flink-1.9.dev0.tar.gz就是我们可以用于pip install的PyFlink包.

## 安装PyFlink
上面我们构建了PyFlink的发布包，接下来我们利用pip进行安装，如下：
```
pip install dist/*.tar.gz
```
输出如下信息，证明已经安装成功：
```
Building wheels for collected packages: apache-flink
  Building wheel for apache-flink (setup.py) ... done
  Stored in directory: /Users/jincheng.sunjc/Library/Caches/pip/wheels/34/b1/23/fca6e31a6de419c9c9d75a6d11df197d538c2ef67e017d79ea
Successfully built apache-flink
Installing collected packages: apache-flink
Successfully installed apache-flink-1.9.dev0
```
用pip命令检查是否安装成功：
```
IT-C02YL16BJHD2:flink-python jincheng.sunjc$ pip list
Package         Version    
--------------- -----------
apache-flink    1.9.dev0   
configparser    3.7.4      
entrypoints     0.3        
enum34          1.1.6      
flake8          3.7.8      
functools32     3.2.3.post2
kafka-python    1.4.6      
mccabe          0.6.1      
pip             19.1.1     
py4j            0.10.8.1   
pycodestyle     2.5.0      
pyflakes        2.1.1      
python-dateutil 2.8.0      
setuptools      41.0.1     
six             1.12.0     
tld             0.9.3      
typing          3.7.4      
virtualenv      16.6.1     
wheel           0.33.4     

```
其中 `apache-flink    1.9.dev0`就是我们刚才安装的版本。  

# 配置IDE开发环境
开发Python最好用的集成开发环境应该是PyCharm了，我们接下来介绍如果配置PyCharm环境。
## 下载安装包
从 [https://www.jetbrains.com/pycharm/](https://www.jetbrains.com/pycharm/) 选择你需要的版本，如果你需要mac版本，直接打开如下链接下载：
[https://download.jetbrains.8686c.com/python/pycharm-community-2019.2.dmg](https://download.jetbrains.8686c.com/python/pycharm-community-2019.2.dmg)
## 安装配置并创建项目
上面下载的应该都是可以直接执行的，一路next之后，就可以对PyCharm进行配置了。我们在启动界面选择创建一个项目，如下：
![](5A3AE975-B828-45D4-BD83-591A1F6A1EFF.png)
点击 "Create New Project"之后，出现如下界面：
![](4ED03355-1070-4A51-8FD1-DB049AB7FEFE.png)
1. 我们要项目取一个名字，比如"myPyFlink"。
2. 选择"Existing interpreter"。
3. 配置"interpreter" **非常重要**,这个配置必须和你刚才安装PyFlink的时候(pip install)的python完全一致。
4. 如果没有你需要的python版本，可以点击 4 所示位置进行添加。
5. 如何知道你pip install对应的python，请使用5所示的命令"which is python"进行查看。

如果需要进行4所需要的操作，点击之后界面如下：
![](8308BB64-2AB2-480C-9B2E-4AADD515F183.png)
选择配置"System Interpreter",并配置"which is python"说输出的路径。
点击 "OK"。
在顺利完成上面配置之后，我们点击 "Create"，完成项目的创建，如下图：
![](77394C3C-D401-4DE6-A005-D4ADFFBFB892.png)
我们可以核实一下，我们依赖的External Libraries是
"which is python"说输出的Python。

# 开发WordCount
如果上面的过程一切顺利，那么我们的器已经很锋利了，开始善其事了，新建一个package,名为enjoyment,然后新建一个word_count.py。如下：
![](6EB7A4C3-B19E-43E0-8E88-6CD2D95936AB.png)
然后，在弹出的界面写入word_count之后，点击 "OK"。
![](E371FA28-BAAE-448F-A930-E795CBF09B0E.png)

## 完整代码
```
# -*- coding: utf-8 -*-

from pyflink.dataset import ExecutionEnvironment
from pyflink.table import TableConfig, DataTypes, BatchTableEnvironment
from pyflink.table.descriptors import Schema, OldCsv, FileSystem

import os

# 数据源文件
source_file = 'source.csv'
#计算结果文件
sink_file = 'sink.csv'

# 创建执行环境
exec_env = ExecutionEnvironment.get_execution_environment()
# 设置并发为1 方便调试
exec_env.set_parallelism(1)
t_config = TableConfig()
t_env = BatchTableEnvironment.create(exec_env, t_config)

# 如果结果文件已经存在,就先删除
if os.path.exists(sink_file):
    os.remove(sink_file)

# 创建数据源表
t_env.connect(FileSystem().path(source_file)) \
    .with_format(OldCsv()
                 .line_delimiter(',')
                 .field('word', DataTypes.STRING())) \
    .with_schema(Schema()
                 .field('word', DataTypes.STRING())) \
    .register_table_source('mySource')

# 创建结果表
t_env.connect(FileSystem().path(sink_file)) \
    .with_format(OldCsv()
                 .field_delimiter(',')
                 .field('word', DataTypes.STRING())
                 .field('count', DataTypes.BIGINT())) \
    .with_schema(Schema()
                 .field('word', DataTypes.STRING())
                 .field('count', DataTypes.BIGINT())) \
    .register_table_sink('mySink')

# 非常简单的word_count计算逻辑
t_env.scan('mySource') \
    .group_by('word') \
    .select('word, count(1)') \
    .insert_into('mySink')

# 执行Job
t_env.execute("wordcount")
```

## 执行并查看运行结果
在执行job之前我们需要先创建一个数据源文件,在enjoyment包目录下，创建 source.csv文件，内容如下：
```
enjoyment,Apache Flink,cool,Apache Flink
```
现在我们可以执行，job了：

![](270E89C8-DB14-4AD7-9EF7-EA5518FC394D.png)
1. 右键word_count.py, 2. 点击运行'word_count'。
运行之后，当控制台出现："Process finished with exit code 0"，并且在enjoyment目录下出现sink.csv文件，说明我们第一个word_count就成功运行了:)
![](34546866-F865-4660-8177-A64C5A14C002.png)
我们双击 "sink.csv",查看运行结果，如下：
![](0B58FCBF-4D98-41F9-98A3-13B31F939916.png)

到此我们Python Table API的开发环境搭建完成。

# 项目源代码
为了方便大家快速体验本篇内容，可以下载本篇涉及到的python项目，git地址如下：[https://github.com/sunjincheng121/enjoyment.code](https://github.com/sunjincheng121/enjoyment.code)

# 小结
本篇是为后续Flink Python Table API 的算子介绍做铺垫，介绍如何搭建开发环境，所以欲善其事必先利其器，最后以一个最简单的，也是最经典的WordCount具体示例来验证开发开发环境。

# 关于评论
如果你没有看到下面的评论区域，请进行翻墙！
![](comment.png)




