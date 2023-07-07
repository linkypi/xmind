## Hadoop3  Spark3 集群搭建

### 1. 机器组成

该集群由三台 Centos7 虚拟机构成，使用的Hadoop版本是3.3.2, spark版本是 3.3.2

设置主机名：

``` sh
cat >> /etc/hosts << EOF
192.168.127.11 node1
192.168.127.12 node2
192.168.127.13 node3
EOF

# 逐个机器设置主机名 
hostnamectl --static set-hostname  node1
```

### 2. 免密登录

``` sh
cd /root
ssh-keygen -t rsa

# 分别在三台机器执行，服务器自身也需要复制！！
for i in node1 node2 node3;\
do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done

#测试
ssh -p 4956 node1
```



### 3. 安装 JDK 并配置环境变量

到官网下载好hadoop安装包，以及所需JDK8安装包，并分别复制到三台机器备用。

``` sh
# 安装
tar -xzvf jdk-8u371-linux-x64.tar.gz -C /usr/local
```

配置环境变量 vim /etc/profile, 配置完成后执行 source /etc/profile

``` sh
JAVA_HOME=/usr/local/jdk1.8.0_371
CLASSPATH=.:%JAVA_HOME%/lib:%JAVA_HOME%/jre/lib
PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
export PATH CLASSPATH JAVA_HOME
```

### 4. 安装部署Hadoop3集群

首先在一台机器上完成所有配置以后 再将所有配置一次性复制到其他机器即可，减少配置操作！！！

``` sh
 mkdir /opt/hadoop-3.3.2
 tar -xzvf hadoop-3.3.2.tar.gz -C /opt
```

#### 4.1 配置  hadoop-env.sh

解压完成后，配置hadoop-env.sh文件中的 JAVA_HOME

``` sh
vim etc/hadoop/hadoop-env.sh

export JAVA_HOME=/usr/local/jdk1.8.0_371
```

#### 4.2 配置 core-site.xml 

vim etc/hadoop/core-site.xml 

``` xml
<configuration>
    <!-- 设置Hadoop的文件系统 --> 
     <property>
        <name>fs.defaultFS</name>
        <value>hdfs://node1:9000</value>
     </property>
    <!-- 配置Hadoop数据存储目录 -->
     <property>
       <name>hadoop.tmp.dir</name>
       <value>/opt/hadoop-3.3.2/data/tempdata</value>
    </property>
    <!--  缓冲区大小 -->
     <property>
       <name>io.file.buffer.size</name>
       <value>4096</value>
     </property>
    <!--  hdfs的垃圾桶机制，单位分钟 -->
     <property>
       <name>fs.trash.interval</name>
       <value>10080</value>
     </property>
</configuration>
```

#### 4.3 配置hdfs-site.xml

配置 hdfs-site.xml ,（hdfs 的核心配置文件），在 <configuration></configuration > 中配置以下内容，注意 secondaryNameNode 和 Namenode 不要放在同一台机器上

``` xml
<!-- SecondaryNameNode的主机和端口 -->
<property>
	<name>dfs.namenode.secondary.http-address</name>
	<value>node2:50090</value>
</property>
<!-- namenode的页面访问地址和端口 -->
<property>
	<name>dfs.namenode.http-address</name>
	<value>node1:50070</value>
</property>
<!-- namenode元数据的存放位置 -->
<property>
	<name>dfs.namenode.name.dir</name>
	<value>file:///opt/hadoop-3.3.2/data/nndata</value>
</property>
<!--  定义datanode数据存储的节点位置 -->
<property>
	<name>dfs.datanode.data.dir</name>
	<value>file:///opt/hadoop-3.3.2/data/dndata</value>
</property>	
<!-- namenode的edits文件存放路径 -->
<property>
	<name>dfs.namenode.edits.dir</name>
	<value>file:///opt/hadoop-3.3.2/data/nn/edits</value>
</property>
<!-- 检查点目录 -->
<property>
	<name>dfs.namenode.checkpoint.dir</name>
	<value>file:///opt/hadoop-3.3.2/data/snn/name</value>
</property>
 
<property>
	<name>dfs.namenode.checkpoint.edits.dir</name>
	<value>file:///opt/hadoop-3.3.2/data/dfs/snn/edits</value>
</property>
<!-- 文件切片的副本个数-->
<property>
	<name>dfs.replication</name>
	<value>3</value>
</property>
<!-- HDFS的文件权限-->
<property>
	<name>dfs.permissions</name>
	<value>true</value>
</property>
<!-- 设置一个文件切片的大小：128M-->
<property>
	<name>dfs.blocksize</name>
	<value>134217728</value>
</property>
```

#### 4.4 配置 mapred-site.xml

``` sh
vim etc/hadoop/mapred-site.xml
```

增加以下配置

``` xml
<!-- 分布式计算使用的框架 -->
<property>
	<name>mapreduce.framework.name</name>
	<value>yarn</value>
</property>
<!-- 开启MapReduce小任务模式 -->
<property>
	<name>mapreduce.job.ubertask.enable</name>
	<value>true</value>
</property>
<!-- 历史任务的主机和端口 -->
<property>
	<name>mapreduce.jobhistory.address</name>
	<value>node1:10020</value>
</property>
<!-- 网页访问历史任务的主机和端口 -->
<property>
	<name>mapreduce.jobhistory.webapp.address</name>
	<value>node1:19888</value>
</property>
```

配置 mapred-env.sh，指定 JAVA_HOME

``` sh
export JAVA_HOME=/usr/local/jdk1.8.0_371
```

#### 4.5 配置 yarn-site.xml

配置 yarn-site.xml（YARN 的核心配置文件） ，在 <configuration></configuration > 中配置以下内容

``` xml

<!-- yarn主节点的位置 -->
<property>
	<name>yarn.resourcemanager.hostname</name>
	<value>node1</value>
</property>
<property>
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
</property>
<!-- 开启日志聚合功能 -->
<property>
	<name>yarn.log-aggregation-enable</name>
	<value>true</value>
</property>
<!-- 设置聚合日志在hdfs上的保存时间 -->
<property>
	<name>yarn.log-aggregation.retain-seconds</name>
	<value>604800</value>
</property>
<!-- 设置yarn集群的内存分配方案 -->
<property>    
	<name>yarn.nodemanager.resource.memory-mb</name>    
	<value>2048</value>
</property>
<property>  
	<name>yarn.scheduler.minimum-allocation-mb</name>
	<value>2048</value>
</property>
<property>
	<name>yarn.nodemanager.vmem-pmem-ratio</name>
	<value>2.1</value>
</property>
```

#### 4.5 配置 workers

``` sh
vim etc/hadoop/workers
```

文件内容是各个节点名称

```
node1
node2
node3
```

#### 4.6 创建数据目录

``` sh
mkdir -p /opt/hadoop-3.3.2/data/tempdata
mkdir -p /opt/hadoop-3.3.2/data/nndata
mkdir -p /opt/hadoop-3.3.2/data/dndata
mkdir -p /opt/hadoop-3.3.2/data/nn/edits
mkdir -p /opt/hadoop-3.3.2/data/snn/name
mkdir -p /opt/hadoop-3.3.2/data/dfs/snn/edits
```

#### 4.7 将配置分发到其他机器

``` sh
scp -r hadoop-3.3.2/etc/hadoop node2:$PWD
scp -r hadoop-3.3.2/etc/hadoop node3:$PWD
```

#### 4.8 配置hadoop环境变量

``` sh
vim /etc/profile.d/customes_env.sh
```

输入

``` shell
# HADOOP_HOME
export HADOOP_HOME=/opt/hadoop-3.3.2
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

#必须配置这部分配置，否则集群无法启动
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
```

使得环境变量生效

```
source /etc/profile
```

#### 4.9. 启动 hadoop集群

首先格式化数据目录, 在 node1 执行

```
hadoop namenode -format
```

格式化完成后即可使用hadoop自带脚本启动集群

``` sh
# 在 node1 执行命令
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver

# 关闭集群服务
stop-dfs.sh
stop-yarn.sh
mr-jobhistory-daemon.sh stop historyserver
```

启动完成后即可通过地址访问

 http://192.168.127.11:50070/explorer.html#/

http://192.168.127.11:8088/cluster/nodes

通过webui 上传文件出现跨越问题，待解决

### 5. 安装部署 Spark3

下载 spark 和 scala 安装包: scala-2.12.18.zip, spark-3.3.2-bin-hadoop3.tgz 。并上传到服务器后解压

```sh
tar -xzvf spark-3.3.2-bin-hadoop3.tgz  -C /opt/
tar -xzvf scala-2.12.18.tgz -C /opt/
```

在文件 customes_env.sh 增加环境变量

```sh
vim /etc/profile.d/customes_env.sh

export HADOOP_CONF_DIR=/opt/hadoop-3.3.2/etc/hadoop/ ##hadoop 配置文件夹

export SPARK_MASTER_HOST=node1 ##spark主节点
export SPARK_LOCAL_DIRS=/opt/spark-3.3.2-bin-hadoop3
export SCALA_HOME=/opt/scala-2.12.18 
export SPARK_DIST_CLASSPATH=$(/opt/hadoop-3.3.2/bin/hadoop classpath)

```

配置spark集群节点，进入到spark目录下的conf文件夹配置

``` sh
cp conf/workers.template conf/workers
vim conf/workers

# workers 文件内容为 
node1
node2
node3
```

进入node1 节点启动 spark master节点

```sh
[root@node1 sbin]# ./start-master.sh
starting org.apache.spark.deploy.master.Master, logging to /opt/spark-3.3.2-bin-hadoop3/logs/spark-root-org.apache.spark.deploy.master.Master-1-node1.out
[root@node1 sbin]# vim /opt/spark-3.3.2-bin-hadoop3/logs/spark-root-org.apache.spark.deploy.master.Master-1-node1.out
```

启动完成后可以通过日志 Master-1-node1.out 查看启动日志，其中包含了控制台的 URL 地址 http://192.168.127.11:8080/

接着在 node1 启动worker工作节点

``` sh
[root@node1 sbin]# ./start-workers.sh 
node3: starting org.apache.spark.deploy.worker.Worker, logging to /opt/spark-3.3.2-bin-hadoop3/logs/spark-root-org.apache.spark.deploy.worker.Worker-1-node3.out
node2: starting org.apache.spark.deploy.worker.Worker, logging to /opt/spark-3.3.2-bin-hadoop3/logs/spark-root-org.apache.spark.deploy.worker.Worker-1-node2.out
node1: starting org.apache.spark.deploy.worker.Worker, logging to /opt/spark-3.3.2-bin-hadoop3/logs/spark-root-org.apache.spark.deploy.worker.Worker-1-node1.out
```

启动完成即可在管控台看到三个工作节点

### 6. 作业提交

#### 6.1 standalone 模式提交

``` sh
# standalone client 模式
/opt/spark-3.3.2-bin-hadoop3/bin/spark-submit \
--class com.spark.WordCountCluster \
--master spark://node1:7077 \
--num-executors 1 \ # 共有多少个executer运行spark app
--driver-memory 800m \
--executor-memory 600m \
--executor-cores 1 \
~/spark.test-1.0-SNAPSHOT.jar

# standalone cluster 模式
/opt/spark-3.3.2-bin-hadoop3/bin/spark-submit \
--class com.spark.WordCountCluster \
--master spark://node1:7077 \
--deploy-mode cluster \
--supervise \ # 指定spark监控driver节点，若driver挂掉则自动重启driver
--num-executors 1 \
--driver-memory 800m \
--executor-memory 600m \
--executor-cores 1 \
~/spark.test-1.0-SNAPSHOT.jar
```



#### 6.2 以yarn cluster模式提交

``` sh
/opt/spark-3.3.2-bin-hadoop3/bin/spark-submit \
--class com.spark.WordCountCluster \
--master yarn \
--deploy-mode cluster \
--num-executors 1 \
--driver-memory 800m \
--conf spark.shuffle.spill=false \ 
--conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:PrintGCTimeStamps" \
--executor-memory 600m \
--executor-cores 1 \
~/spark.test-1.0-SNAPSHOT.jar
```

注意在该方式下提交作业时不能在程序代码中设置 setMaster("...").

spark.shuffle.spill 默认值 true，用于指定 Shuffle 过程中如果内存中的数据超过阈值时是否需要将部分数据临时写入外部存储。此参数可以配置为 false。没有 spill，会极大提高 Spark 的处理效率，但前提需要确保集群内存够用







