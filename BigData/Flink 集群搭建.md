Flink 集群搭建

### 1. 机器分布

Flink集群沿用 spark3 , hadoop3 集群机器，都是三台 Centos7 机器，flink 使用的是 1.16.0， 支持  JDK 11，当前使用 JDK8.

Flink集群分为几种部分方式：

- **Local** 模式，JobManager 和 TaskManager 会公用一个 JVM，该模式将压缩包解压后，直接执行 bin 目录下的 start-cluster.sh 即可

-  **Standalone** 模式，高可用靠的是 ZK来完成

- **Yarn Cluster** 模式，

### 2. Standalone模式（无HA）

由于 node1 已经作为 hadoop 及 spark 的master节点，故此处将flink的master节点部署在 node2.  

#### 2.1 配置环境变量

首先将flink压缩包解压到 /opt 目录，接下来配置基于此前的 hadoop 集群配置文件 /etc/profile.d/customes_env.sh 来配置环境变量。

``` sh
# 在文件末尾追加
export FLINK_HOME=/opt/flink-1.16.0
export PATH=$PATH:$FLINK_HOME/bin

# 使得配置生效
source /etc/profile.d/customes_env.sh
```

#### 2.2 修改flink配置

紧接着配置flink配置文件 flink-conf.yaml， 进入到 /opt/flink-1.16.0/conf

``` sh
vim flink-conf.yaml
```

``` yaml
jobmanager.rpc.address: node2
jobmanager.bind-host: 0.0.0.0
# 每个 JobManager 的可用内存大小
jobmanager.memory.process.size: 2048m

# 每个 TaskManager 的可用内存大小，默认为 1024M
taskmanager.memory.process.size: 2048m
# 每台计算机可用的 CPU 数
taskmanager.numberOfTaskSlots: 2
# 集群中默认的并行度
parallelism.default: 8

rest.address: node2
rest.bind-address: 0.0.0.0
```

flink-conf.yaml 配置完成后继续配置conf下的主节点masters 及工作节点 workers
```sh
# masters 文件
node2:8081
```

``` sh
# workers 文件
node1
node2
node3
```

#### 2.3 复制配置到其他主机

环境变量 及 flink相关配置文件都配置好以后，就可以将这些配置都复制到其他两台机器

``` shell
scp /etc/profile.d/customes_env.sh node1:/etc/profile.d/customes_env.sh
scp /etc/profile.d/customes_env.sh node3:/etc/profile.d/customes_env.sh

 scp conf/* node1:/opt/flink-1.16.0/conf/
 scp conf/* node3:/opt/flink-1.16.0/conf/
```

#### 2.4 启动集群

在node2 启动集群

``` sh
[root@node2 bin]# ./start-cluster.sh
Starting cluster.
Starting standalonesession daemon on host node2.
Starting taskexecutor daemon on host node1.
Starting taskexecutor daemon on host node2.
Starting taskexecutor daemon on host node3.
```

查看机器进程启动情况

``` sh
[root@node2 bin]# jps
2801 StandaloneSessionClusterEntrypoint
3240 TaskManagerRunner
3370 Jps

[root@node3 ~]# jps
3011 TaskManagerRunner
3130 Jps

[root@node1 ~]# jps
3059 Jps
2900 TaskManagerRunner
```

最后通过  http://node2:8081/#/overview 访问控制台

### 3.  Yarn Cluster 模式

该模式的前提是必须先搭建好Yarn 集群。Flink 通过 Yarn 的接口实现了自己的 App Master。当在 Yarn 中部署了 Flink，Yarn 就会用自己的 Container 来启动 Flink 的 JobManager（即 App Master）和 TaskManager。启动流程如下：

1. 启动新的 Flink YARN 会话时，客户端首先检查所请求的资源（容器和内存）是否可用。之后，将包含 Flink 和配置的 jar 上传到 HDFS
2. YARN 资源服务器 RM 接收到请求后会分配一个节点来启动 *ApplicationMaster*（即 JobManager），AM启动后继续向 RM 申请资源来启动 TaskManager （即在NodeManager节点启动）
3. task启动后就会到 HDFS 下载相关配置及 jar 包来执行

#### 3.1 修改配置

首先设置hadoop配置文件路径的环境变量，已设置的可以忽略

``` sh
export HADOOP_CONF_DIR=/opt/hadoop-3.3.2/etc/hadoop 
```

#### 3.2 部署启动

进入到 flink 的bin目录，执行以下命令

``` sh
# -n : TaskManager的数量，相当于 executor 的数量
# -d 表示以后台程序方式运行
# -jm,--jobManagerMemory <arg>    JobManager的内存 [in MB]  
# -nm,--name                     在YARN上为一个自定义的应用设置一个名字  
# -q,--query                      显示yarn中可用的资源 (内存, cpu核数)  
# -qu,--queue <arg>               指定YARN队列.  
# -s,--slots <arg>                每个TaskManager使用的slots数量  
# -tm,--taskManagerMemory <arg>   每个TaskManager的内存 [in MB]  
# -z,--zookeeperNamespace <arg>   针对HA模式在zookeeper上创建NameSpace 
# -D <property=value>             use value for given property
# -id,--applicationId <yarnAppId> YARN集群上的任务id，附着到一个后台运行的yarn    session中
yarn-session.sh -d -s 2 -tm 1024 -n 2
```

启动报错

```sh
[root@node2 bin]# ./yarn-session.sh -d -s 2 -tm 1024 -n 2
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/yarn/exceptions/YarnException
        at java.lang.Class.getDeclaredMethods0(Native Method)
        at java.lang.Class.privateGetDeclaredMethods(Class.java:2701)
        at java.lang.Class.privateGetMethodRecursive(Class.java:3048)
        at java.lang.Class.getMethod0(Class.java:3018)
        at java.lang.Class.getMethod(Class.java:1784)
        at sun.launcher.LauncherHelper.validateMainClass(LauncherHelper.java:669)
        at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:651)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.yarn.exceptions.YarnException
        at java.net.URLClassLoader.findClass(URLClassLoader.java:387)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
```

原因是 flink lib文件夹下没有 [flink-shaded-hadoop-3-uber](https://mvnrepository.com/artifact/org.apache.flink/flink-shaded-hadoop-3-uber/3.1.1.7.2.9.0-173-9.0) 包，注意 hadoop 版本， 该jar 包可以到maven repo 下载。下载完成后将jar包放到 lib 目录下，然后继续执行以上命令即可看到日志：
``` sh
······
2023-06-24 11:01:48,664 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - Adding delegation tokens to the AM container.
2023-06-24 11:01:48,665 INFO  org.apache.flink.runtime.security.token.KerberosDelegationTokenManager [] - Loading delegation token providers
2023-06-24 11:01:48,667 INFO  org.apache.flink.runtime.security.token.KerberosDelegationTokenManager [] - Delegation token provider hadoopfs loaded and initialized
2023-06-24 11:01:48,667 INFO  org.apache.flink.runtime.security.token.KerberosDelegationTokenManager [] - Delegation token providers loaded successfully
2023-06-24 11:01:48,667 INFO  org.apache.flink.runtime.security.token.KerberosDelegationTokenManager [] - Obtaining delegation tokens
2023-06-24 11:01:48,668 INFO  org.apache.flink.runtime.security.token.KerberosDelegationTokenManager [] - Real user has no kerberos credentials so no tokens obtained
2023-06-24 11:01:48,669 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - Delegation tokens added to the AM container.
2023-06-24 11:01:48,676 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - Submitting application master application_1687617528132_0001
2023-06-24 11:01:49,157 INFO  org.apache.hadoop.yarn.client.api.impl.YarnClientImpl        [] - Submitted application application_1687617528132_0001
2023-06-24 11:01:49,157 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - Waiting for the cluster to be allocated
2023-06-24 11:01:49,159 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - Deploying cluster, current state ACCEPTED
2023-06-24 11:02:00,982 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - YARN application has been deployed successfully.
2023-06-24 11:02:00,982 INFO  org.apache.flink.yarn.YarnClusterDescriptor                  [] - Found Web Interface node1:37321 of application 'application_1687617528132_0001'.
JobManager Web Interface: http://node1:37321
2023-06-24 11:02:01,149 INFO  org.apache.flink.yarn.cli.FlinkYarnSessionCli                [] - The Flink YARN session cluster has been started in detached mode. In order to stop Flink gracefully, use the following command:
$ echo "stop" | ./bin/yarn-session.sh -id application_1687617528132_0001
If this should not be possible, then you can also kill Flink via YARN's web interface or via:
$ yarn application -kill application_1687617528132_0001
Note that killing Flink might not clean up all job artifacts and temporary files.
```

在日志可以看到集群已经启动完成，可以通过 http://node1:37321 访问flink 管控台，同时在 hadoop cluster http://192.168.127.11:8088/cluster/apps/RUNNING  页面可以看到已经启动的集群，对应的程序名称 Name 为 Flink session cluster。可以通过下面命令来停止集群或移除集群:

``` sh
$ echo "stop" | ./bin/yarn-session.sh -id application_1687617528132_0001
$ yarn application -kill application_1687617528132_0001
```

#### 3.3 提交任务

``` sh
bin/flink run -m yarn-cluster \
-yn 2 -yjm 1024 -ytm 1024 \
-c com/flink/BatchWordCount \
-yj ~/spark.test-1.0-SNAPSHOT.jar \
-yid application_1687617528132_0001 \
--input hdfs://node1:9000/words.txt \
--output hdfs://node1:9000/output1

./flink run -yjm 1024 -ytm 1024 \
-c com.flink.BatchWordCount \
-yid application_1687617528132_0001 \
~/spark.test-1.0-SNAPSHOT.jar
```

