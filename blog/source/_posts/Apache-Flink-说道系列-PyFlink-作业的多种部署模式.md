---
title: Apache Flink 说道系列 - PyFlink 作业的多种部署模式
date: 2020-01-02 17:15:18
categories: Apache Flink 说道
tags: [Flink, 道德经, 致虚极守静笃]
---

# 开篇说道
老子说：“致虚极，守静笃。万物并作，吾以观其复"。 字里行间是告诉人们要尽量使心灵达到虚寂的极致，持之以恒的坚守宁静。老子认为“虚”和“静”是心灵的本初的状态，也应该是一种常态，即，是一种没有心机，没有成见，没有外界名利，物质诱惑，而保持的状态。当达到这个状态之后，就可以静观万物的蓬勃发展，了解万物的循环往复的变化，通晓自然之理，体悟自然之道。

"致虚极，守静笃" 并不代表对任何事情无动于衷，而是通过 "致虚极，守静笃" 的方式了解万物变化的根源，就好比，看到新芽不惊，看到落叶不哀，因为我们知道，落叶归根，重归泥土之后，还会再破土而出，开始新的轮回。

在2020年1月1日零点我发了一条朋友圈，对自己的祝福是："㊗️ 2020 安"。
![](15776987959455/15779569378844.jpg)
当然这里我也祝福所有看到这篇博客的你: "㊗️ 2020 安"。 祝福大家 "安"，也即祝福大家 能够在新的一年，追求内心的修炼，摒去纷纷复杂的物欲干扰，拂去心灵的灰尘，保持自己的赤子之心！ 

# 道破 "天机"
前面一些博客大多介绍PyFlink的功能开发，比如，如何使用各种算子(Join/Window/AGG etc.)，如何使用各种Connector(Kafka, CSV, Socket etc.)，还有一些实际的案例。这些都停留在开发阶段，一旦开发完成，我们就面临激动人心的时刻，那就是 将我们精心设计开发的作业进行部署，那么问题来了，你知道怎样部署PyFlink的作业吗？这篇将为大家全面介绍部署PyFlink作业的各种模式。

# 组件栈回顾
![](15776987959455/15777810183663.jpg)


上面的组件栈除了PyFlink是第一次添加上去，其他部分大家应该非常熟悉了。目前PyFlink基于Java的Table API之上，同时在Runtime层面有Python的算子和执行容器。那么我们聚焦重点，看最底层的 Deploy部分，上图我们分成了三种部署模式，Local/Cluster/Cloud，其中Local模式还有2种不同方式，一是SingleJVM，也即是MiniCluster, 前面博客里面运行示例所使用的就是MiniCluster。二是SingleNode，也就是虽然是集群模式，但是所有角色都在一台机器上。下面我们简单介绍一下上面这几种部署模式的区别：
- Local-SingleJVM 模式- 该模式大多是开发测试阶段使用的方式，所有角色TM，JM等都在同一个JVM里面。
- Local-SingleNode 模式 - 意在所有角色都运行在同一台机器，直白一点就是从运行的架构上看，这种模式虽然是分布式的，但集群节点只有1个，该模式大多是测试和IoT设备上进行部署使用。
- Cluster 模式 - 也就是我们经常用于投产的分布式部署方式，上图根据对资源管理的方式不同又分为了多种，如：Standalone 是Flink自身进行资源管理，YARN，顾名思义就是利用资源管理框架Yarn来负责Flink运行资源的分配，还有结合Kubernetes等等。
- Cloud 模式- 该部署模式是结合其他云平台进行部署。

接下来我们看看PyFlink的作业可以进行怎样的模式部署？

# 环境依赖
- JDK 1.8+ (1.8.0_211)
- Maven 3.x (3.2.5)
- Scala 2.11+ (2.12.0)
- Python 3.5+ (3.7.6)
- Git 2.20+ (2.20.1)

# 源码构建及安装
在Apache Flink 1.10 发布之后，我们除了源码构建之外，还支持直接利用pip install 安装PyFlink。那么现在我们还是以源码构建的方式进行今天的介绍。
- 下载源码

```
git clone https://github.com/apache/flink.git
```
- 签出release-1.10分支(1.10版本是PyFlink的第二个版本）

```
git fetch origin release-1.10
git checkout -b release-1.10 origin/release-1.10
```
- 构建编译

```
mvn clean package -DskipTests 
```
如果一起顺利，你会最终看到如下信息：

```
...
...
[INFO] flink-walkthrough-table-scala ...................... SUCCESS [  0.070 s]
[INFO] flink-walkthrough-datastream-java .................. SUCCESS [  0.081 s]
[INFO] flink-walkthrough-datastream-scala ................. SUCCESS [  0.067 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  16:22 min
[INFO] Finished at: 2019-12-31T10:37:21+08:00
[INFO] ------------------------------------------------------------------------
```
- 构建PyFlink发布包

上面我们构建了Java的发布包，接下来我们构建PyFlink的发布包,如下：

```
cd flink-Python; Python setup.py sdist
```
最终输出如下信息，证明是成功的：

```
copying pyflink/util/exceptions.py -> apache-flink-1.10.dev0/pyflink/util
copying pyflink/util/utils.py -> apache-flink-1.10.dev0/pyflink/util
Writing apache-flink-1.10.dev0/setup.cfg
creating dist
Creating tar archive
removing 'apache-flink-1.10.dev0' (and everything under it)
```
在dist目录的apache-flink-1.10.dev0.tar.gz就是我们可以用于`pip install`的PyFlink包.

- 安装PyFlink

上面我们构建了PyFlink的发布包，接下来我们利用pip进行安装，检测是否之前已经安装过PyFlink，如下命令：

```
pip3 list|grep flink
...
flink                         1.0      
pyflink-demo-connector        0.1 
```
上面信息说明我本机已经安装过PyFlink，我们要先删除，如下：


```
pip3 uninstall flink
```
删除以前的安装之后，我们再安装新的如下：
```
pip3 install dist/*.tar.gz

...
Successfully built apache-flink
Installing collected packages: apache-flink
Successfully installed apache-flink-1.10.dev0
```
我们再用list命令检查一遍：


```
pip3 list|grep flink

...
apache-flink                  1.10.dev0
pyflink-demo-connector        0.1 
```
其中`pyflink-demo-connector` 是我以前做实验时候的安装，对本篇没有影响。

# 安装Apache Beam 依赖

我们需要使用Python3.5+ 版本，检验一下Python版本，如下：
```
jincheng.sunjc$ Python --version
Python 3.7.6
```

我本机是Python3.7.6，现在我们需要安装Apache Beam，如下：

```
python -m pip install apache-beam==2.15.0

...
Installing collected packages: apache-beam
Successfully installed apache-beam-2.15.0
```
如果顺利的出现上面信息，说明Apache-beam已经安装成功。

# PyFlink示例作业

接下来我们开发一个简单的PyFlink作业，源码如下：

```
import logging
import os
import shutil
import sys
import tempfile

from pyflink.table import BatchTableEnvironment, EnvironmentSettings
from pyflink.table.descriptors import FileSystem, OldCsv, Schema
from pyflink.table.types import DataTypes
from pyflink.table.udf import udf


def word_count():
   environment_settings = EnvironmentSettings.new_instance().in_batch_mode().use_blink_planner().build()
   t_env = BatchTableEnvironment.create(environment_settings=environment_settings)

   # register Results table in table environment
   tmp_dir = tempfile.gettempdir()
   result_path = tmp_dir + '/result'
   if os.path.exists(result_path):
       try:
           if os.path.isfile(result_path):
               os.remove(result_path)
           else:
               shutil.rmtree(result_path)
       except OSError as e:
           logging.error("Error removing directory: %s - %s.", e.filename, e.strerror)

   logging.info("Results directory: %s", result_path)

   # we should set the Python verison here if `Python` not point
   t_env.get_config().set_Python_executable("Python3")

   t_env.connect(FileSystem().path(result_path)) \
       .with_format(OldCsv()
                    .field_delimiter(',')
                    .field("city", DataTypes.STRING())
                    .field("sales_volume", DataTypes.BIGINT())
                    .field("sales", DataTypes.BIGINT())) \
       .with_schema(Schema()
                    .field("city", DataTypes.STRING())
                    .field("sales_volume", DataTypes.BIGINT())
                    .field("sales", DataTypes.BIGINT())) \
       .register_table_sink("Results")

   @udf(input_types=DataTypes.STRING(), result_type=DataTypes.ARRAY(DataTypes.STRING()))
   def split(input_str: str):
       return input_str.split(",")

   @udf(input_types=[DataTypes.ARRAY(DataTypes.STRING()), DataTypes.INT()], result_type=DataTypes.STRING())
   def get(arr, index):
       return arr[index]

   t_env.register_function("split", split)
   t_env.register_function("get", get)

   t_env.get_config().get_configuration().set_string("parallelism.default", "1")

   data = [("iPhone 11,30,5499,Beijing", ),
           ("iPhone 11 Pro,20,8699,Guangzhou", ),
           ("MacBook Pro,10,9999,Beijing", ),
           ("AirPods Pro,50,1999,Beijing", ),
           ("MacBook Pro,10,11499,Shanghai", ),
           ("iPhone 11,30,5999,Shanghai", ),
           ("iPhone 11 Pro,20,9999,Shenzhen", ),
           ("MacBook Pro,10,13899,Hangzhou", ),
           ("iPhone 11,10,6799,Beijing", ),
           ("MacBook Pro,10,18999,Beijing", ),
           ("iPhone 11 Pro,10,11799,Shenzhen", ),
           ("MacBook Pro,10,22199,Shanghai", ),
           ("AirPods Pro,40,1999,Shanghai", )]
   t_env.from_elements(data, ["line"]) \
       .select("split(line) as str_array") \
       .select("get(str_array, 3) as city, "
               "get(str_array, 1).cast(LONG) as count, "
               "get(str_array, 2).cast(LONG) as unit_price") \
       .select("city, count, count * unit_price as total_price") \
       .group_by("city") \
       .select("city, "
               "sum(count) as sales_volume, "
               "sum(total_price) as sales") \
       .insert_into("Results")

   t_env.execute("word_count")


if __name__ == '__main__':
   logging.basicConfig(stream=sys.stdout, level=logging.INFO, format="%(message)s")
   word_count()
```
接下来我们就介绍如何用不同部署模式运行PyFlink作业！

# Local-SingleJVM 模式部署
该模式多用于开发测试阶段，简单的利用`Python pyflink_job.py`命令，PyFlink就会默认启动一个Local-SingleJVM的Flink环境来执行作业，如下：

![](15776987959455/15777784863901.jpg)
首先确认你Python是3.5+，然后执行上面的PyFlink作业`Python deploy_demo.py`,结果写入到本地文件，然后`cat`计算结果，如果出现如图所示的结果，则说明准备工作已经就绪。:),如果有问题，欢迎留言！
这里运行时SingleJVM，在运行这个job时候大家可以查看java进程：

![](15776987959455/15777821361064.jpg)
我们发现只有一个JVM进程，里面包含了所有Flink所需角色。


# Local-SingleNode 模式部署
这种模式一般用在单机环境中进行部署，如IoT设备中，我们从0开始进行该模式的部署操作。我们进入到`flink/build-target`目录，执行如下命令(个人爱好，我把端口改成了8888)：

``` 
jincheng:build-target jincheng.sunjc$ bin/start-cluster.sh 
...
Starting cluster.
Starting standalonesession daemon on host jincheng.local.
```
查看一下Flink的进程：

![](15776987959455/15777817105172.jpg)

我们发现有TM和JM两个进程，虽然在一台机器(Local）但是也是一个集群的架构。
上面信息证明已经启动完成，我们可以查看web界面：[http://localhost:8888/](http://localhost:8888/)（我个人爱好端口是8888，默认是8080）, 如下：
![](15776987959455/15777816500228.jpg)

目前集群环境已经准备完成，我们看如果将作业部署到集群中，一条简单的命令，如下：

```
bin/flink run -m localhost:8888 -py ~/deploy_demo.py
```
这里如果你不更改端口可以不添加`-m` 选项。如果一切顺利，你会得到如下输出：

```
jincheng:build-target jincheng.sunjc$ bin/flink run -m localhost:8888 -py ~/deploy_demo.py 
Results directory: /var/folders/fp/s5wvp3md31j6v5gjkvqbkhrm0000gp/T/result
Job has been submitted with JobID 3ae7fb8fa0d1867daa8d65fd87ed3bc6
Program execution finished
Job with JobID 3ae7fb8fa0d1867daa8d65fd87ed3bc6 has finished.
Job Runtime: 5389 ms
```
其中`/var/folders/fp/s5wvp3md31j6v5gjkvqbkhrm0000gp/T/result` 目录是计算结果目录，我们可以产看一下，如下：

```
jincheng:build-target jincheng.sunjc$  cat /var/folders/fp/s5wvp3md31j6v5gjkvqbkhrm0000gp/T/result
Beijing,110,622890
Guangzhou,20,173980
Shanghai,90,596910
Shenzhen,30,317970
Hangzhou,10,138990
```
同时我们也可以在WebUI上面进行查看，在完成的job列表中，显示如下：
![](15776987959455/15777853568978.jpg)
到此，我们完成了在Local模式，其实也是只有一个节点的Standalone模式下完成PyFlink的部署。
最后我们为了继续下面的操作，请停止集群：

```
jincheng:build-target jincheng.sunjc$ bin/stop-cluster.sh
Stopping taskexecutor daemon (pid: 45714) on host jincheng.local.
Stopping standalonesession daemon (pid: 45459) on host jincheng.local.
```

# Cluster YARN 模式部署
这个模式部署，我们需要一个YARN环境，我们一切从简，以单机部署的方式准备YARN环境，然后再与Flink进行集成。
## 准备YARN环境
- 安装Hadoop

我本机是mac系统，所以我偷懒一下直接用brew进行安装：

```
jincheng:bin jincheng.sunjc$ brew install Hadoop
Updating Homebrew...
==> Auto-updated Homebrew!
Updated 2 taps (homebrew/core and homebrew/cask).
==> Updated Formulae
Python ✔        doxygen         minio           ntopng          typescript
certbot         libngspice      mitmproxy       ooniprobe
doitlive        minimal-racket  ngspice         openimageio

==> Downloading https://www.apache.org/dyn/closer.cgi?path=hadoop/common/hadoop-
==> Downloading from http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-3.2.1/
######################################################################## 100.0%

🍺  /usr/local/Cellar/Hadoop/3.2.1: 22,397 files, 815.6MB, built in 5 minutes 12 seconds
```
完成之后，检验一下hadoop版本：

```
jincheng:bin jincheng.sunjc$ hadoop version
Hadoop 3.2.1
```
超级顺利，hadoop被安装到了`/usr/local/Cellar/hadoop/3.2.1/`目录下，`brew`还是很能提高生产力啊～

- 配置免登(SSH)

Mac系统自带了ssh，我们可以简单配置一下即可，我们先打开远程登录。 `系统偏好设置` -> `共享` 中，左边勾选 `远程登录` ，右边选择 `仅这些用户` （选择`所有用户`更宽松），并添加当前用户。

```
jincheng:bin jincheng.sunjc$ whoami
jincheng.sunjc
```
我当前用户是 `jincheng.sunjc`。 配置图如下：

![](15776987959455/15777878055973.jpg)

然后生产证书，如下操作：

```
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
Generating public/private rsa key pair.
/Users/jincheng.sunjc/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Your identification has been saved in /Users/jincheng.sunjc/.ssh/id_rsa.
Your public key has been saved in /Users/jincheng.sunjc/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:IkjKkOjfMx1fxWlwtQYg8hThph7Xlm9kPutAYFmQR0A jincheng.sunjc@jincheng.local
The key's randomart image is:
+---[RSA 2048]----+
|       ..EB=.o.. |
|..      =.+.+ o .|
|+ .      B.  = o |
|+o .    + o + .  |
|.o. . .+S. * o   |
|  . ..o.= + =    |
|   . + o . . =   |
|      o     o o  |
|            .o   |
+----[SHA256]-----+
```
接下来将公钥追加到如下文件，并修改文件权限：

```
jincheng.sunjc$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
jincheng.sunjc$ chmod 0600 ~/.ssh/authorized_keys
```
利用ssh localhost验证，看到 Last login: 字样为ssh成功
 
```
jincheng:~ jincheng.sunjc$ ssh localhost
Password:
Last login: Tue Dec 31 18:26:48 2019 from ::1
```
- 设置环境变量

设置JAVA_HOME,HADOOP_HOME和HADOOP_CONF_DIR，`vi ~/.bashrc`:
  
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_211.jdk/Contents/Home

export HADOOP_HOME=/usr/local/Cellar/hadoop/3.2.1/libexec

export HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop
```
NOTE: 后续操作要确保的terminal环境变量是生效哦， 如果不生效可以执行 `source ~/.bashrc`。:)

- 修改配置

    1) 修改core-site.xml
     
```
<configuration>
   <property>
      <name>hadoop.tmp.dir</name>
      <value>/tmp</value>
   </property>
   <property>
      <name>fs.defaultFS</name>
      <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

    2) 修改hdfs-site.xml
    
```
<configuration>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/tmp/hadoop/name</value>
    </property>
    
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/tmp/hadoop/data</value>
    </property>
    
</configuration>
```

    3) 修改yarn-site.xml

配置 YARN 作为资源管理框架：
```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>    
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>  <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

简单的配置已经完成，我们执行一下简单命令启动环境：

- 格式化文档系统：

```
jincheng:libexec jincheng.sunjc$ hadoop namenode -format
...
...
2019-12-31 18:58:53,260 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at jincheng.local/127.0.0.1
************************************************************/

```

- 启动服务：

我们先启动hdf再启动yarn，如下图：
![](15776987959455/15778389671396.jpg)

Okay,一切顺利的话，我们会启动namenodes，datanodes，resourcemanager和nodemanagers。我们有几个web界面可以查看，如下：

 1). Overview 界面， [http://localhost:9870 ](http://localhost:9870 )如下：
![](15776987959455/15778373195478.jpg)


 2). NodeManager界面， [http://localhost:8042](http://localhost:8042)，如下：
![-w649](15776987959455/15778373437959.jpg)

 3). ResourceManager 管理界面 [http://localhost:8088/](http://localhost:8088/),如下：
![-w1054](15776987959455/15777972228895.jpg)

目前YARN的环境已经准备完成，我们接下来看如何与Flink进行集成。

## Flink集成Hadoop包
切换到编译结果目录下`flink/build-target`,并将haddop的JAR包放到`lib`目录。
在官网下载hadoop包，

```
cd lib;
curl https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.8.3-7.0/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar > flink-shaded-hadoop-2-uber-2.8.3-7.0.jar
```
下载后，lib目录下文件如下：
![](15776987959455/15777916683830.jpg)

到现在为止我们可以提交PyFlink的作业到由YARN进行资源分配的集群了。但为了确保集群上有正确的Python环境我们最好打包一个Python环境到集群上面。因为大部分情况下我们无法得知yarn集群上的Python版本是否符合我们的要求(Python 3.5+，装有apache-beam 2.15.0)，因此我们需要打包一个符合条件的Python环境，并随job文件提交到yarn集群上。

## 打包Python环境
再次检查一下当前Python的版本是否3.5+，如下：
```
jincheng:lib jincheng.sunjc$ Python
Python 3.7.6 (default, Dec 31 2019, 09:48:30) 
```
由于这个Python环境是用于集群的，所以打包时的系统需要和集群一致。如果不一致，比如集群是linux而本机是mac，我们需要在虚拟机或者docker中打包。以下列出两种情况的示范方法，读者根据需求选择一种即可。

### 本地打包（集群和本机操作系统一致时）
如果集群所在机器的操作系统和本地一致（都是mac或者都是linux），直接通过virtualenv打包一个符合条件的Python环境：
- 安装virtualenv
使用`python -m pip install virtualenv` 进行安装如下：

```
jincheng:tmp jincheng.sunjc$ python -m pip install virtualenv
Collecting virtualenv
  Downloading https://files.Pythonhosted.org/packages/05/f1/2e07e8ca50e047b9cc9ad56cf4291f4e041fa73207d000a095fe478abf84/virtualenv-16.7.9-py2.py3-none-any.whl (3.4MB)
     |████████████████████████████████| 3.4MB 2.0MB/s 
Installing collected packages: virtualenv
Successfully installed virtualenv-16.7.9
```
我本地环境已经成功安装。

- 创建Python环境
用virtualenv以always-copy方式建立一个全新的Python环境，名字随意，以venv为例，`virtualenv --always-copy venv`:

```
jincheng:tmp jincheng.sunjc$ virtualenv --always-copy venv
Using base prefix '/usr/local/Cellar/Python/3.7.6/Frameworks/Python.framework/Versions/3.7'
New Python executable in /Users/jincheng.sunjc/temp/hadoop/tmp/venv/bin/Python3.7
Also creating executable in /Users/jincheng.sunjc/temp/hadoop/tmp/venv/bin/Python
Installing setuptools, pip, wheel...
done.
```
- 在新环境中安装apache-beam 2.15.0
使用`venv/bin/pip install apache-beam==2.15.0`进行安装：

```
jincheng:tmp jincheng.sunjc$ venv/bin/pip install apache-beam==2.15.0
Collecting apache-beam==2.15.0
...
...
Successfully installed apache-beam-2.15.0 avro-Python3-1.9.1 certifi-2019.11.28 chardet-3.0.4 crcmod-1.7 dill-0.2.9 docopt-0.6.2 fastavro-0.21.24 future-0.18.2 grpcio-1.26.0 hdfs-2.5.8 httplib2-0.12.0 idna-2.8 mock-2.0.0 numpy-1.18.0 oauth2client-3.0.0 pbr-5.4.4 protobuf-3.11.2 pyarrow-0.14.1 pyasn1-0.4.8 pyasn1-modules-0.2.7 pydot-1.4.1 pymongo-3.10.0 pyparsing-2.4.6 pytz-2019.3 pyyaml-3.13 requests-2.22.0 rsa-4.0 six-1.13.0 urllib3-1.25.7
```
上面信息已经说明我们成功的在Python环境中安装了apache-beam==2.15.0。接下来我们打包Python环境。
- 打包Python环境
 我们将Python打包成zip文件，`zip -r venv.zip venv` 如下：
 
```
zip -r venv.zip venv
...
...
  adding: venv/lib/Python3.7/re.py (deflated 68%)
  adding: venv/lib/Python3.7/struct.py (deflated 46%)
  adding: venv/lib/Python3.7/sre_parse.py (deflated 80%)
  adding: venv/lib/Python3.7/abc.py (deflated 72%)
  adding: venv/lib/Python3.7/_bootlocale.py (deflated 63%)
```
查看一下zip大小：

```
jincheng:tmp jincheng.sunjc$ du -sh venv.zip 
 81M	venv.zip
```
这个大小实在太大了，核心问题是Beam的包非常大，后面我会持续在Beam社区提出优化建议。我们先忍一下:(。

### Docker中打包（比如集群为linux，本机为mac时）
我们选择在docker中打包，可以从以下链接下载最新版docker并安装：
[https://download.docker.com/mac/stable/Docker.dmg](https://download.docker.com/mac/stable/Docker.dmg)安装完毕后重启终端，执行`docker version`确认docker安装成功：

```
jincheng:tmp jincheng.sunjc$ docker version
Client: Docker Engine - Community
 Version:           19.03.4
 API version:       1.40
 Go version:        go1.12.10
 Git commit:        9013bf5
 Built:             Thu Oct 17 23:44:48 2019
 OS/Arch:           darwin/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.4
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.10
  Git commit:       9013bf5
  Built:            Thu Oct 17 23:50:38 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.2.10
  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
 runc:
  Version:          1.0.0-rc8+dev
  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

- 启动容器
我们启动一个Python 3.7版本的容器如果是第一次启动可能需要较长时间来拉取镜像：
`docker run -it Python:3.7 /bin/bash`, 如下：

```
jincheng:libexec jincheng.sunjc$  docker run -it Python:3.7 /bin/bash
Unable to find image 'Python:3.7' locally
3.7: Pulling from library/Python
8f0fdd3eaac0: Pull complete 
d918eaefd9de: Pull complete 
43bf3e3107f5: Pull complete 
27622921edb2: Pull complete 
dcfa0aa1ae2c: Pull complete 
bf6840af9e70: Pull complete 
167665d59281: Pull complete 
ffc544588c7f: Pull complete 
4ebe99df65fe: Pull complete 
Digest: sha256:40d615d7617f0f3b54614fd228d41a891949b988ae2b452c0aaac5bee924888d
Status: Downloaded newer image for Python:3.7
```

- 容器中安装virtualenv
我们在刚才启动的容器中安装virtualenv， `pip install virtualenv`,如下：
```
root@1b48d2b526ae:/# pip install virtualenv
Collecting virtualenv
  Downloading https://files.Pythonhosted.org/packages/05/f1/2e07e8ca50e047b9cc9ad56cf4291f4e041fa73207d000a095fe478abf84/virtualenv-16.7.9-py2.py3-none-any.whl (3.4MB)
     |████████████████████████████████| 3.4MB 2.0MB/s 
Installing collected packages: virtualenv
Successfully installed virtualenv-16.7.9
root@1b48d2b526ae:/# 
```

- 创建Python环境
以always copy方式建立一个全新的Python环境，名字随意，以venv为例，`virtualenv --always-copy venv`, 如下：


```
root@1b48d2b526ae:/# virtualenv --always-copy venv
Using base prefix '/usr/local'
New Python executable in /venv/bin/Python
Installing setuptools, pip, wheel...
done.
root@1b48d2b526ae:/# 
```

- 安装Apache Beam
 在新的Python环境中安装apache-beam 2.15.0，`venv/bin/pip install apache-beam==2.15.0`,如下：
 
```
root@1b48d2b526ae:/# venv/bin/pip install apache-beam==2.15.0
Collecting apache-beam==2.15.0
...
...
Successfully installed apache-beam-2.15.0 avro-Python3-1.9.1 certifi-2019.11.28 chardet-3.0.4 crcmod-1.7 dill-0.2.9 docopt-0.6.2 fastavro-0.21.24 future-0.18.2 grpcio-1.26.0 hdfs-2.5.8 httplib2-0.12.0 idna-2.8 mock-2.0.0 numpy-1.18.0 oauth2client-3.0.0 pbr-5.4.4 protobuf-3.11.2 pyarrow-0.14.1 pyasn1-0.4.8 pyasn1-modules-0.2.7 pydot-1.4.1 pymongo-3.10.0 pyparsing-2.4.6 pytz-2019.3 pyyaml-3.13 requests-2.22.0 rsa-4.0 six-1.13.0 urllib3-1.25.7
```
- 查看docker中的Python环境
用`exit`命令退出容器，用`docker ps -a`找到docker容器的id，用于拷贝文件,如下：

```
root@1b48d2b526ae:/# exit
exit
jincheng:libexec jincheng.sunjc$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
1b48d2b526ae        Python:3.7          "/bin/bash"         7 minutes ago       Exited (0) 8 seconds ago                       elated_visvesvaraya
```
由于刚刚结束，一般来说是列表中的第一条，可以根据容器的镜像名Python:3.7来分辨。我们记下最左边的容器ID。如上是`1b48d2b526ae`。

- 打包Python环境
 从将容器中的Python环境拷贝出来，我们切换到`flink/build-target`录下，拷贝 `docker cp 1b48d2b526ae:/venv ./`并打包`zip -r venv.zip venv`。
 最终`flink/build-target`录下生成`venv.zip`。
 

## 部署作业
终于到部署作业的环节了:), flink on yarn支持两种模式，per-job和session。per-job模式在提交job时会为每个job单独起一个flink集群，session模式先在yarn上起一个flink集群，之后提交job都提交到这个flink集群。

### Pre-Job 模式部署作业
执行以下命令，以Pre-Job模式部署PyFlink作业：
`bin/flink run -m yarn-cluster -pyarch venv.zip -pyexec venv.zip/venv/bin/Python -py deploy_demo.py`,如下：

```
jincheng:build-target jincheng.sunjc$ bin/flink run -m yarn-cluster -pyarch venv.zip -pyexec venv.zip/venv/bin/Python -py deploy_demo.py
2020-01-02 13:04:52,889 WARN  org.apache.flink.yarn.cli.FlinkYarnSessionCli                 - The configuration directory ('/Users/jincheng.sunjc/blog/demo_dev/flink/flink-dist/target/flink-1.10-SNAPSHOT-bin/flink-1.10-SNAPSHOT/conf') already contains a LOG4J config file.If you want to use logback, then please delete or rename the log configuration file.
2020-01-02 13:04:52,889 WARN  org.apache.flink.yarn.cli.FlinkYarnSessionCli                 - The configuration directory ('/Users/jincheng.sunjc/blog/demo_dev/flink/flink-dist/target/flink-1.10-SNAPSHOT-bin/flink-1.10-SNAPSHOT/conf') already contains a LOG4J config file.If you want to use logback, then please delete or rename the log configuration file.
Results directory: /var/folders/fp/s5wvp3md31j6v5gjkvqbkhrm0000gp/T/result
2020-01-02 13:04:55,945 INFO  org.apache.hadoop.yarn.client.RMProxy                         - Connecting to ResourceManager at /0.0.0.0:8032
2020-01-02 13:04:56,049 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - No path for the flink jar passed. Using the location of class org.apache.flink.yarn.YarnClusterDescriptor to locate the jar
2020-01-02 13:05:01,153 WARN  org.apache.flink.yarn.YarnClusterDescriptor                   - Neither the HADOOP_CONF_DIR nor the YARN_CONF_DIR environment variable is set. The Flink YARN Client needs one of these to be set to properly load the Hadoop configuration for accessing YARN.
2020-01-02 13:05:01,177 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Cluster specification: ClusterSpecification{masterMemoryMB=1024, taskManagerMemoryMB=1024, numberTaskManagers=1, slotsPerTaskManager=1}
2020-01-02 13:05:01,294 WARN  org.apache.flink.yarn.YarnClusterDescriptor                   - The file system scheme is 'file'. This indicates that the specified Hadoop configuration path is wrong and the system is using the default Hadoop configuration values.The Flink YARN client needs to store its files in a distributed file system
2020-01-02 13:05:02,600 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Submitting application master application_1577936885434_0004
2020-01-02 13:05:02,971 INFO  org.apache.hadoop.yarn.client.api.impl.YarnClientImpl         - Submitted application application_1577936885434_0004
2020-01-02 13:05:02,972 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Waiting for the cluster to be allocated
2020-01-02 13:05:02,975 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Deploying cluster, current state ACCEPTED
2020-01-02 13:05:23,138 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - YARN application has been deployed successfully.
2020-01-02 13:05:23,140 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Found Web Interface localhost:61616 of application 'application_1577936885434_0004'.
Job has been submitted with JobID a41d82194a500809fd715da8f29894a0
Program execution finished
Job with JobID a41d82194a500809fd715da8f29894a0 has finished.
Job Runtime: 35576 ms
```
上面信息已经显示运行完成，在Web界面可以看到作业状态：
![](15776987959455/15779417223687.jpg)

我们再检验一下计算结果`cat /var/folders/fp/s5wvp3md31j6v5gjkvqbkhrm0000gp/T/result`：
![](15776987959455/15779417681405.jpg)

到这里，我们以Pre-Job的方式成功部署了PyFlink的作业！相比提交到本地Standalone集群，多了三个参数，我们简单说明如下：

| 参数                               | 说明                                                  |
|----------------------------------|-----------------------------------------------------|
| -m yarn-cluster                  | 以Per-Job模式部署到yarn集群                                 |
| -pyarch venv.zip                 | 将当前目录下的venv.zip上传到yarn集群                            |
| -pyexec venv.zip/venv/bin/Python | 指定venv.zip中的Python解释器来执行Python UDF，路径需要和zip包内部结构一致。 |

### Session 模式部署作业
以Session模式部署作业也非常简单，我们实际操作一下：

```
jincheng:build-target jincheng.sunjc$ bin/yarn-session.sh 
2020-01-02 13:58:53,049 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.address, localhost
2020-01-02 13:58:53,050 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.port, 6123
2020-01-02 13:58:53,050 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.heap.size, 1024m
2020-01-02 13:58:53,050 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.memory.process.size, 1024m
2020-01-02 13:58:53,050 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.numberOfTaskSlots, 1
2020-01-02 13:58:53,050 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: parallelism.default, 1
2020-01-02 13:58:53,051 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.execution.failover-strategy, region
2020-01-02 13:58:53,413 WARN  org.apache.hadoop.util.NativeCodeLoader                       - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
2020-01-02 13:58:53,476 INFO  org.apache.flink.runtime.security.modules.HadoopModule        - Hadoop user set to jincheng.sunjc (auth:SIMPLE)
2020-01-02 13:58:53,509 INFO  org.apache.flink.runtime.security.modules.JaasModule          - Jaas file will be created as /var/folders/fp/s5wvp3md31j6v5gjkvqbkhrm0000gp/T/jaas-3848984206030141476.conf.
2020-01-02 13:58:53,521 WARN  org.apache.flink.yarn.cli.FlinkYarnSessionCli                 - The configuration directory ('/Users/jincheng.sunjc/blog/demo_dev/flink/flink-dist/target/flink-1.10-SNAPSHOT-bin/flink-1.10-SNAPSHOT/conf') already contains a LOG4J config file.If you want to use logback, then please delete or rename the log configuration file.
2020-01-02 13:58:53,562 INFO  org.apache.hadoop.yarn.client.RMProxy                         - Connecting to ResourceManager at /0.0.0.0:8032
2020-01-02 13:58:58,803 WARN  org.apache.flink.yarn.YarnClusterDescriptor                   - Neither the HADOOP_CONF_DIR nor the YARN_CONF_DIR environment variable is set. The Flink YARN Client needs one of these to be set to properly load the Hadoop configuration for accessing YARN.
2020-01-02 13:58:58,824 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Cluster specification: ClusterSpecification{masterMemoryMB=1024, taskManagerMemoryMB=1024, numberTaskManagers=1, slotsPerTaskManager=1}
2020-01-02 13:59:03,975 WARN  org.apache.flink.yarn.YarnClusterDescriptor                   - The file system scheme is 'file'. This indicates that the specified Hadoop configuration path is wrong and the system is using the default Hadoop configuration values.The Flink YARN client needs to store its files in a distributed file system
2020-01-02 13:59:04,779 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Submitting application master application_1577936885434_0005
2020-01-02 13:59:04,799 INFO  org.apache.hadoop.yarn.client.api.impl.YarnClientImpl         - Submitted application application_1577936885434_0005
2020-01-02 13:59:04,799 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Waiting for the cluster to be allocated
2020-01-02 13:59:04,801 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Deploying cluster, current state ACCEPTED
2020-01-02 13:59:24,711 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - YARN application has been deployed successfully.
2020-01-02 13:59:24,713 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Found Web Interface localhost:62247 of application 'application_1577936885434_0005'.
JobManager Web Interface: http://localhost:62247

```
执行成功后不会返回，但会启动一个JoBManager Web，地址如上`http://localhost:62247`，可复制到浏览器查看:
![](15776987959455/15779449909178.jpg)


我们可以修改conf/flink-conf.yaml中的配置参数。如果要更改某些内容，请参考[官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html)。接下来我们提交作业，首先按组合键Ctrl+Z将yarn-session.sh进程切换到后台，并执行bg指令让其在后台继续执行, 然后执行以下命令，即可向Session模式的flink集群提交job `bin/flink run -m yarn-cluster -pyarch venv.zip -pyexec venv.zip/venv/bin/Python -py deploy_demo.py`：


```
jincheng:build-target jincheng.sunjc$ bin/flink run -pyarch venv.zip -pyexec venv.zip/venv/bin/Python -py deploy_demo.py

2020-01-02 14:10:48,285 INFO  org.apache.flink.yarn.YarnClusterDescriptor                   - Found Web Interface localhost:62247 of application 'application_1577936885434_0005'.
Job has been submitted with JobID bea33b7aa07c0f62153ab5f6e134b6bf
Program execution finished
Job with JobID bea33b7aa07c0f62153ab5f6e134b6bf has finished.
Job Runtime: 34405 ms
```
如果在打印`finished`之前查看之前的web页面，我们会发现Session集群会有一个正确运行的作业，如下：
![](15776987959455/15779454827330.jpg)
如果已经运行完成，那么我们应该会看到状态也变成结束：
![](15776987959455/15779456315383.jpg)

相比per job模式提交，少了”-m”参数。因为之前已经启动了yarn-session.sh，所以flink默认会向yarn-session.sh启动的集群上提交job。执行完毕后，别忘了关闭yarn-session.sh（session 模式）：先将yarn-session.sh调到前台，执行`fg`,然后在再按Ctrl+C结束进程或者执行`stop`，结束时yarn上的集群也会被关闭。

# Docker 模式部署
我们还可以将Flink Python job打包成docker镜像，然后使用docker-compose或者Kubernetes部署执行，由于现在的docker镜像打包工具并没有完美支持运行Python UDF，因此我们需要往里面添加一些额外的文件。首先是一个仅包含PythonDriver类的jar包. 我们在`build-target`目录下执行如下命令：

```
jincheng:build-target jincheng.sunjc$ mkdir temp
jincheng:build-target jincheng.sunjc$ cd temp
jincheng:temp jincheng.sunjc$ unzip ../opt/flink-Python_2.11-1.10-SNAPSHOT.jar org/apache/flink/client/Python/PythonDriver.class
Archive:  ../opt/flink-Python_2.11-1.10-SNAPSHOT.jar
  inflating: org/apache/flink/client/Python/PythonDriver.class 
```
解压之后，我们在进行压缩打包：

```
jincheng:temp jincheng.sunjc$ zip Python-driver.jar org/apache/flink/client/Python/PythonDriver.class
  adding: org/apache/flink/client/Python/PythonDriver.class (deflated 56%)
```
![](15776987959455/15779465411786.jpg)
我们得到`Python-driver.jar`。然后下载一个pyArrow的安装文件(我准备了一个大家下载直接使用即可 [pyarrow-0.12.0a0-cp36-cp36m-linux_x86_64.whl](https://github.com/sunjincheng121/enjoyment.code/blob/master/share_resource/pyarrow-0.12.0a0-cp36-cp36m-linux_x86_64.whl)。执行以下命令构建Docker镜像，需要作为artifacts引入的文件有作业文件，Python-driver的jar包和pyarrow安装文件,`./build.sh --job-artifacts ~/deploy_demo.py,Python-driver.jar,pyarrow-0.12.0a0-cp36-cp36m-linux_x86_64.whl --with-Python3 --from-local-dist`(进入`flink/flink-container/docker`目录)

```
jincheng:docker jincheng.sunjc$ ./build.sh --job-artifacts ~/deploy_demo.py,Python-driver.jar,pyarrow-0.12.0a0-cp36-cp36m-linux_x86_64.whl --with-Python3 --from-local-dist
Using flink dist: ../../flink-dist/target/flink-*-bin
a .
a ./flink-1.10-SNAPSHOT
a ./flink-1.10-SNAPSHOT/temp
...
...
Removing intermediate container a0558bbcbdd1
 ---> 00ecda6117b7
Successfully built 00ecda6117b7
Successfully tagged flink-job:latest
```
构建Docker镜像需要较长时间，请耐心等待。构建完毕之后，可以输入docker images命令在镜像列表中找到构建结果`docker images
`：
![-w765](15776987959455/15779503938890.jpg)
然后我们在构建好的镜像基础上安装好Python udf所需依赖，并删除过程中产生的临时文件：
- 启动docker容器
`docker run -it --user root --entrypoint /bin/bash --name flink-job-container flink-job`
- 安装一些依赖
`apk add --no-cache g++ Python3-dev musl-dev`
- 安装PyArrow
`python -m pip3 install /opt/artifacts/pyarrow-0.12.0a0-cp36-cp36m-linux_x86_64.whl`

- 安装Apache Beam
`python -m pip3 install apache-beam==2.15.0`

- 删除临时文件
`rm -rf /root/.cache/pip`

执行完如上命令我可以执行`exit`退出容器了，然后把这个容器提交为新的flink-job镜像`docker commit -c 'CMD ["--help"]' -c "USER flink" -c 'ENTRYPOINT ["/docker-entrypoint.sh"]' flink-job-container flink-job:latest 
`：

```
jincheng:docker jincheng.sunjc$ docker commit -c 'CMD ["--help"]' -c "USER flink" -c 'ENTRYPOINT ["/docker-entrypoint.sh"]' flink-job-container flink-job:latest 
sha256:0740a635e2b0342ddf776f33692df263ebf0437d6373f156821f4dd044ad648b
```
到这里包含Python UDF 作业的Docker镜像就制作好了，这个Docker镜像既可以以docker-compose使用，也可以结合kubernetes中使用。

我们以使用docker-compose执行为例，mac版docker自带docker-compose，用户可以直接使用，在`flink/flink-container/docker`目录下，使用以下命令启动作业,`FLINK_JOB=org.apache.flink.client.Python.PythonDriver FLINK_JOB_ARGUMENTS="-py /opt/artifacts/deploy_demo.py" docker-compose up`:


```
jincheng:docker jincheng.sunjc$ FLINK_JOB=org.apache.flink.client.Python.PythonDriver FLINK_JOB_ARGUMENTS="-py /opt/artifacts/deploy_demo.py" docker-compose up
WARNING: The SAVEPOINT_OPTIONS variable is not set. Defaulting to a blank string.
Recreating docker_job-cluster_1 ... done
Starting docker_taskmanager_1   ... done
Attaching to docker_taskmanager_1, docker_job-cluster_1
taskmanager_1  | Starting the task-manager
job-cluster_1  | Starting the job-cluster
...
...
job-cluster_1  | 2020-01-02 08:35:03,796 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         - Terminating cluster entrypoint process StandaloneJobClusterEntryPoint with exit code 0.
docker_job-cluster_1 exited with code 0
```
在log中出现“docker_job-cluster_1 exited with code 0”表示job已执行成功，JobManager已经退出。TaskManager还需要较长的时间等待超时后才会退出，我们可以直接按快捷键Ctrl+C提前退出。

查看执行结果，可以从TaskManager的容器中将结果文件拷贝出来查看,执行 `docker cp docker_taskmanager_1:/tmp/result ./; cat result `
![](15776987959455/15779542617667.jpg)

Okay, 到这里本篇要与大家分享的内容已经接近尾声了，如果你期间也很顺利的成功了，可以 Cheers 了:)

# 小结
本篇核心向大家分享了如何以多种方式部署PyFlink作业。期望在PyFlink1.10发布之后，大家能有一个顺利快速体验的快感！在开篇说道部分，为大家分享了老子倡导大家的 “致虚极，守静笃。万物并作，吾以观其复”的大道，同时也给大家带来了2020的祝福，祝福大家 "2020 安！"。











