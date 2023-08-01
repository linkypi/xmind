## 使用 kubeasz 部署 k8s 集群
在生产环境下单纯靠手工操作各个组件来部署k8s集群是非常不可靠的。虽然可以很好的启动整个集群，但是后期的维护成本
将会异常繁琐，也非常容易出错。文档[基于三台机器搭建k8s集群](./基于三台机器搭建k8s集群.md)更适合于开发人员
自行学习k8s集群，更好的了解其内部各个组件的工作原理。

故为了可以快速部署及后期维护简单，使用 kubeasz 来完成集群自动化部署。
### 1. 前期准备

####  设置主机名并开启免密登录

```sh
cat >> /etc/hosts << EOF
192.168.127.200 k8s.node1
192.168.127.210 k8s.node2
192.168.127.220 k8s.node3
EOF

# 逐个机器设置主机名 
hostnamectl --static set-hostname k8s.node1
```

```sh
cd /root
ssh-keygen -t rsa
for i in k8s.node3 k8s.node2 k8s.node1 ;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done	
```

### 2. 下载脚本工具 ezdown

```sh
export release=3.6.1
wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
chmod +x ./ezdown
cp ezdown /usr/bin
rm -fr ezdown
```
> 注意！！！若此前已经安装有kubeasz，同时已经运行有通过该工具部署的 k8s 集群，则需要将集群所在目录 /etc/kubeasz/clusters 的配置文件做备份，否则下载新版本kubeasz后会把原有集群配置都覆盖。
然后使用 ezdown 下载安装集群所需要的基础镜像：

```
# 1. 下载kubeasz代码、二进制、默认容器镜像

# 国内环境
./ezdown -D
# 海外环境
#./ezdown -D -m standard

# 2. 下载额外镜像
./ezdown -X
```

上述脚本运行成功后，所有文件（kubeasz代码、二进制、离线镜像）均已整理好放入目录`/etc/kubeasz`, 若之前安装有久版本，则需要把 /etc/kubeasz/bin下的文件删除后再重新下载

### 3. 集群管理

#### 3.1 创建集群

```sh
# 容器化运行 kubeasz
./ezdown -S

# 创建新集群 k8s-01
docker exec -it kubeasz ezctl new k8s-01

# 精简命令
alias dk='docker exec -it kubeasz'
source ~/.bashrc
```

集群创建完成后会在宿主机目录 `/etc/kubeasz/clusters/` 下新建一个集群名称文件夹，用于存储集群相关配置文件：`hosts` 及 `config.yml`

#### 3.2 配置集群

首先通过 hosts 配置集群需要部署的机器，以及master节点及work节点分布情况等(当前只会列出特别需要注意及调整的配置)：

##### 3.2.1 配置 hosts

注意这些节点只能使用 IP ，不可使用 主机名，否则集群会启动失败

```sh
[etcd]
192.168.57.30
192.168.57.31
192.168.57.32

# master node(s)
[kube_master]
192.168.57.31
192.168.57.32

# work node(s)
[kube_node]
192.168.57.30
192.168.57.31
192.168.57.32

[all:vars]
# --------- Main Variables ---------------
# Secure port for apiservers
SECURE_PORT="26443"

# Cluster container-runtime supported: docker, containerd
# if k8s version >= 1.24, docker is not supported
CONTAINER_RUNTIME="containerd"

# NodePort Range
NODE_PORT_RANGE="8000-32767"
```

##### 3.2.2 配置 hosts
> 建议安装 kubernetes V1.27.0 及以上集群，因为1.27.0 以下的集群使用的 etcd版本 存在一个 Bug
> 两个相关 Issue: https://github.com/etcd-io/etcd/issues/14025, https://github.com/etcd-io/etcd/issues/15090
> 该Bug会导致Etcd集群异常时无法快速恢复(如磁盘空间被占满)！！！而 1.27.0 使用的Etcd版本是 3.5.7，
> 该版本已修复此问题。

修改完成后，接下来是修改 config.yml， 注意 nfs 服务需要先自行部署

```sh
# k8s version
K8S_VER: "1.27.0"

# nfs-provisioner 自动安装
nfs_provisioner_install: "yes"
nfs_provisioner_namespace: "kube-system"
nfs_provisioner_ver: "v4.0.2"
nfs_storage_class: "managed-nfs-storage"
nfs_server: "192.168.57.31"
nfs_path: "/data/nfs" # 注意实际目录
```

由于 node-local-dns 默认使用 8080 端口作为健康检查端口，可能会与平时开发使用的端口冲突，故需要做修改，进入集群文件夹 `/etc/kubeasz/clusters/k8s-01` 下方的 yml 文件夹，编辑文件 nodelocaldns.yaml。找到 8080 端口，修改为 6080 即可。

##### 3.2.3 配置 containerd 相关配置文件

为了是的pod启动时可以顺利下载私有仓库镜像，需要在containerd配置文件中设置私有仓库地址，并关闭 https 安全认证。

```shell
# 首先生成默认配置文件
containerd config default > /etc/containerd/config.toml

# 修改配置，增加私有仓库地址 以及 关闭https认证
# 在 config.toml 配置文件中找到 .registry.configs 及 .registry.mirrors，并增加以下配置
# 其中 easzlab.io.local:5000 是 kubeasz 自动构建的镜像仓库，k8s.nexus 是部署内部服务使用的镜像仓库
 [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."k8s.nexus".tls]
             insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."easzlab.io.local:5000".tls]
             insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."k8s.nexus".auth]
             username = "admin"
             password = "password"

 [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.nexus"]
            endpoint = ["http://192.168.57.31:8090"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."easzlab.io.local:5000"]
            endpoint = ["http://192.168.57.31:5000"]
```

修改完成配置以后还需要将配置放到启动程序中

``` sh
# 查看containerd服务所在位置
[root@k8s 200G]# systemctl status containerd
● containerd.service - containerd container runtime
   Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-06-26 19:07:39 HKT; 1min 58s ago
     Docs: https://containerd.io

# 编辑服务文件， 增加 /etc/containerd/config.toml
 vim /usr/lib/systemd/system/containerd.service
```

``` sh
[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/containerd --config /etc/containerd/config.toml
```
配置完成后重启 containerd
``` sh
systemctl daemon-reload
systemctl restart containerd

# 重启完成查看是否有配置到 
containerd config dump | grep nexus
```

##### 3.2.4 修改 node-dns-local 默认端口

node-dns-local默认端口为 8080 ，导致该端口与内部微服务的端口可能存在冲突，故需要修改该端口为其他不常使用的端口 6080

#### 3.3 启动集群

配置设置完成后就可以使用以下命令直接启动集群

````sh
dk ezctl setup k8s-01 all
````



#### 3.4 集群管理命令

```


[root@k8s k8s.infra]# dk ezctl 
Usage: ezctl COMMAND [args]
-------------------------------------------------------------------------------------
Cluster setups:
    list                             to list all of the managed clusters
    checkout    <cluster>            to switch default kubeconfig of the cluster
    new         <cluster>            to start a new k8s deploy with name 'cluster'
    setup       <cluster>  <step>    to setup a cluster, also supporting a step-by-step way
    start       <cluster>            to start all of the k8s services stopped by 'ezctl stop'
    stop        <cluster>            to stop all of the k8s services temporarily
    upgrade     <cluster>            to upgrade the k8s cluster
    destroy     <cluster>            to destroy the k8s cluster
    backup      <cluster>            to backup the cluster state (etcd snapshot)
    restore     <cluster>            to restore the cluster state from backups
    start-aio                        to quickly setup an all-in-one cluster with default settings

Cluster ops:
    add-etcd    <cluster>  <ip>      to add a etcd-node to the etcd cluster
    add-master  <cluster>  <ip>      to add a master node to the k8s cluster
    add-node    <cluster>  <ip>      to add a work node to the k8s cluster
    del-etcd    <cluster>  <ip>      to delete a etcd-node from the etcd cluster
    del-master  <cluster>  <ip>      to delete a master node from the k8s cluster
    del-node    <cluster>  <ip>      to delete a work node from the k8s cluster
    
    
[root@k8s k8s.infra]# dk ezctl setup
Usage: ezctl setup <cluster> <step>
available steps:
    01  prepare            to prepare CA/certs & kubeconfig & other system settings 
    02  etcd               to setup the etcd cluster
    03  container-runtime  to setup the container runtime(docker or containerd)
    04  kube-master        to setup the master nodes
    05  kube-node          to setup the worker nodes
    06  network            to setup the network plugin
    07  cluster-addon      to setup other useful plugins
    90  all                to run 01~07 all at once
    10  ex-lb              to install external loadbalance for accessing k8s from outside
    11  harbor             to install a new harbor server or to integrate with an existed one

examples: ./ezctl setup test-k8s 01  (or ./ezctl setup test-k8s prepare)
          ./ezctl setup test-k8s 02  (or ./ezctl setup test-k8s etcd)
          ./ezctl setup test-k8s all
          ./ezctl setup test-k8s 04 -t restart_master
```

### 4. 集群备份 ！！！
集群部署完成后即可安装argocd服务来一键启动整体服务，操作文档请查看[argocd readme](../argo-cd/readme.md)
待所有环境的所有服务都启动完成并可正常访问后，还有一件非常重要的事情需要处理，那就是--集群备份！！！

集群备份，备份的是当前k8s集群已正常运行的服务的所有元数据，以便在k8s集群出现异常（如机器磁盘爆满等异常）时可以快速恢复，减少宕机
恢复时间。k8s集群的数据默认放在 Etcd 集群中，故只需要备份这部分数据即可。由于 argocd 可以使用一键快速部署
整个集群服务，所以argocd的元数据可以不做备份。

虽然kubeasz也提供了集群备份功能，但尝试多次都是失败。并且其本身并未提供定时备份功能。所以为了实现该功能就需要自行
编写自动备份脚本
```shell
#!/bin/bash 
# 必须增加这一行导入当前用户环境变量，否则会提示 etcdctl command not found
source ~/.bash_profile

export ETCDCTL_API=3 
CLUSTER="esop"
ENDPOINT="https://k8s.node1:2379"
DIR="/etc/kubeasz/clusters/esop/backup"

targetDir=$DIR/`date +%Y%m`
[ -d $targetDir ] || mkdir -p $targetDir
cd $targetDir
# etcdctl --endpoints=$ENDPOINT snapshot save ${CLUSTER}_`date +%F`.db
etcdctl snapshot save ${CLUSTER}_`date +%F`.db
```
将上面的脚本保存到文件 /etc/kubeasz/clusters/esop/backup-etcd.sh ，然后编辑 crontab
```shell
vim /etc/crontab
```
在文件末尾增加一行
```shell
0 0 * * * root /etc/kubeasz/clusters/esop/backup-etcd.sh
```
这样就可以实现每天凌晨自动备份，还可以使用以下命令查看备份数据.
```shell
# snapshot备份
$ ETCDCTL_API=3 etcdctl snapshot save backup.db
# 查看备份
$ ETCDCTL_API=3 etcdctl --write-out=table snapshot status backup.db
```

待实际使用时只需执行以下命令恢复即可
```shell
# snapshot 恢复备份
$ ETCDCTL_API=3 etcdctl restore save backup.db
```
若使用定时任务导致文件备份失败，则可以通过查看执行日志来排查
``` sh
tail -300 /var/log/cron
tail -300 /var/spool/mail/root
```

### 5. Troubleshooting

#### 5.1 calicoctl 节点部署失败问题排查
```shell
[root@k8s k8s.infra]# calicoctl node status  --allow-version-mismat
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.57.32 | node-to-node mesh | up    | 01:30:45 | Established |
| 192.168.57.30 | node-to-node mesh | up    | 01:30:16 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

[root@k8s sit-redis]# calicoctl get nodes -o wide --allow-version-mismat 
NAME        ASN       IPV4               IPV6   
k8s.node1   (64512)   192.168.57.31/24          
k8s.node2   (64512)   192.168.57.32/24          
k8s.node3   (64512)   192.168.57.30/24 

```

#### 5.2 因磁盘空间不足导致 k8s etcd集群异常
> 注意： 在修复etcd集群过程中不可随便删除已有数据，因为etcd集群保存了所有k8s集群容器的元数据，若将etcd集群数据
> 删除则在恢复k8s集群后需要手动重新部署所有服务。

若尝试重启etcd都无法解决，则可以重新构建etcd服务，在构建服务之前请记得首先备份数据目录下的数据，一般数据目录位于
/var/lib/etcd/member。备份完成后即可使用 dk etcd 命令移除当前节点，然后再添加一个新的etcd服务。
```shell
Cluster ops:
    add-etcd    <cluster>  <ip>      to add a etcd-node to the etcd cluster
    del-etcd    <cluster>  <ip>      to delete a etcd-node from the etcd cluster
    
# 对应的执行命令为
dk ezctl del esop 192.168.57.xxx
dk ezctl add esop 192.168.57.xxx
```
若仍无效则可查看[官方文档](https://etcd.io/docs/v3.5/op-guide/runtime-configuration/#add-a-new-member)进行操作