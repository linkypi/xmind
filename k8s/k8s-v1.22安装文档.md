### 1. 主机规划

|     IP地址      | 机器名称 |  机器配置   | 机器角色 |                           安装软件                           |
| :-------------: | :------: | :---------: | :------: | :----------------------------------------------------------: |
| 192.168.127.10  | master01 | 2C  2G  30G |  master  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、haproxy、keepalived、 |
| 192.168.127.20  | master02 | 2C  2G  30G |  master  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、haproxy、keepalived、 |
| 192.168.127.30  | master03 | 2C  2G  30G |  master  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、haproxy、keepalived、 |
| 192.168.127.40  |  node01  | 2C  2G  30G |   node   |                     kubelet、kube-proxy                      |
| 192.168.127.50  |  node02  | 2C  2G  30G |   node   |                     kubelet、kube-proxy                      |
| 192.168.127.60  |  node03  | 2C  2G  30G |   node   |                     kubelet、kube-proxy                      |
| 192.168.127.210 | proxy01  | 2C  2G  30G |  proxy   |                          keepalived                          |
| 192.168.127.220 | proxy02  | 2C  2G  30G |  proxy   |                          keepalived                          |
| 192.168.127.180 |  harbor  | 2C  2G  30G | 容器仓库 |     docker, harbor, cfssl, bind. 证书生成，域名解析服务      |
| 192.168.127.117 |    /     |      /      |  虚拟IP  | 由 HAProxy 和 keepalived 组成的 LB，由192.168.127.117统一路由到其他三个主节点 |

### 2. 软件版本

|                             软件                             |            版本            |
| :----------------------------------------------------------: | :------------------------: |
|                       centos-7.9.2009                        | 5.12.9-1.el7.elrepo.x86_64 |
| kube-apiserver、kube-controller-manager、<br />kube-scheduler、kubelet、kube-proxy |          v 1.22.2          |
|                             etcd                             |           v 3.5            |
|                           coredns                            |                            |
|                            docker                            |          v 20.18           |
|                          keepalived                          |          v 1.3.5           |

### 3. 网络分配

| 网段信息 |     配置     |
| :------: | :----------: |
|   Pod    | 172.7.0.0/16 |
| Service  | 10.96.0.0/16 |

### 4. 构建基础VM

#### 4.1 配置主机 IP 及源

```sh
# /etc/resolv.conf
nameserver 114.114.114.114

# /etc/sysconfig/network-scripts/ifcfg-ens33
ONBOOT=yes
IPADDR=192.168.127.10
GATEWAY=192.168.127.2
NETMASK=255.255.255.0

# 修改完成重启网络
systemctl restart network

#配置源
cd /etc/yum.repos.d/   
mv CentOS-Base.repo CentOS-Base.repo_bak
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache

# 安装常用工具
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git lrzsz -y
```

#### 4.2 升级 linux 内核到5.x

查看linux内核版本

```sh
$ uname -a
Linux k8s-master 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

$ cat /etc/redhat-release 
CentOS Linux release 7.3.1611 (Core) 
```

内核版本为 3.10 需要升级到 5

```sh
# 更新 yum 源仓库
yum -y update

# 启用 ELRepo 仓库
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

# 查看最新内核版本
[root@localhost yum.repos.d]# yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * elrepo-kernel: hkg.mirror.rackspace.com
..... 
Available Packages
.....
kernel-lt.x86_64       5.4.151-1.el7.elrepo  elrepo-kernel
.....
kernel-ml.x86_64       5.14.10-1.el7.elrepo  elrepo-kernel
.....

#这里使用稳定版本 kernel-lt，即 long term support 版本，而kernel-ml是主线版本较新
yum --enablerepo=elrepo-kernel install kernel-lt
```

内核安装完成后需要设置默认启动项/etc/default/grub 为其增加参数 GRUB_DEFAULT=0 ，即GRUB 初始化页面的第一个内核将作为默认内核。：

```
GRUB_TIMEOUT=5
GRUB_DEFAULT=0
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root rd.lvm.lv=centos/swap crashkernel=auto rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

接下来运行下面的命令来重新创建内核配置

```sh
grub2-set-default "kernel-lt-5.4.151-1"
grub2-mkconfig -o /boot/grub2/grub.cfg
```

重启后即可发现最新内核放到了首位选项中。

#### 4.3 关闭 selinux 及防火墙

编辑文件 /etc/selinux/config ，将 SELINUX 设置为disabled。

再者关闭防火墙：

```sh
systemctl stop firewalld
systemctl disable firewalld
firewall-cmd --set-default-zone=trusted
```

#### 4.4 关闭虚拟内存交换 swap

```sh
swapoff -a ; sed -i '/swap/d' /etc/fstab
```

#### 4.5 k8s内核优化

```sh
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 131072
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
sysctl --system

#所有节点配置完内核后，重启服务器，保证重启后内核依旧加载
reboot -h now

#重启后查看结果：
lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

#### 4.6 安装后台进程管理器 supervisor

```sh
yum install -y epel-release
yum install -y supervisor
systemctl enable supervisord 
systemctl start supervisord 
```

至此，一个基础VM已构建完成，接下来就可以基于该VM直接复制出需要的VM即可。

### 5. 设置主机名

```sh
cat >> /etc/hosts << EOF
192.168.127.10 master01
192.168.127.20 master02
192.168.127.30 master03
192.168.127.40 node01
192.168.127.50 node02
192.168.127.60 node03
192.168.127.210 proxy01
192.168.127.220 proxy02
192.168.127.180 harbor
EOF

# 逐个机器设置主机名 
hostnamectl --static set-hostname  master01
```

### 6. 免密登录配置

```sh
cd /root
ssh-keygen -t rsa

for i in master01 master02 master03 ;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done

for i in node01 node02 node03;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done

for i in master01 master02 master03 node01 node02 node03 harbor;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done

#测试
ssh -p 4956 master02
```

### 7. 准备根证书

由于证书签发是放在harbor 机器，所以需要进入harbor机器进行操作

#### 7.1 安装 cfssl

由于cfssl是使用go语言开发，所以需要先安装go再编译源码

```
# 首先安装 epel 否则无法找到 go
yum install epel-release
yum install go -y
```

下载源码并编译

```sh
git clone https://github.com/cloudflare/cfssl.git
cd cfssl
make

# 编译完成后的文件
❯ tree bin
bin
├── cfssl
├── cfssl-bundle
├── cfssl-certinfo
├── cfssljson
├── cfssl-newkey
├── cfssl-scan
├── mkbundle
├── multirootca
└── rice
# 将文件复制到 /usr/bin 后开启执行权限
cp bin/* /usr/bin
chmod +x /usr/bin/cfssl*

mkdir /opt/cert
cd /opt/cert/
```

#### 7.2 生成根证书

接下来生成根证书请求文件 ca-csr.json:

```sh
cat > ca-csr.json <<"EOF"
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "system"
    }
  ],
  "ca": {
     "expiry": "175200h"
  }
}
EOF
```

创建根证书：

```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

**配置 ca 证书策略**

```sh
cat > ca-config.json <<"EOF"
{
  "signing": {
      "default": {
          "expiry": "175200h"
       },
       "profiles": {
          "server": {
             "expiry": "175200h",
             "usages": [
                "signing",
                "key encipherment",
                "server auth"
            ]
          },
          "client": {
             "expiry": "175200h",
             "usages": [
                "signing",
                "key encipherment",
                "client auth"
            ]
          },
          "peer": {
             "expiry": "175200h",
             "usages": [
                "signing",
                "key encipherment",
                "server auth",
                "client auth"
            ]
          }
       }
    }
}
EOF
```

### 8. 部署master集群

master集群由三台主机 master01, master02, master03 组成，故需要在这三台机器部署。为了保证三台主机的apiserver高可用，所以需要使用 haproxy + keepalived 做负载均衡。访问方式是通过统一的虚拟IP 192.168.127.117:16443 进行访问，负载到其他三台主机的6443端口，若其中一台apiserver出问题就会自动切换到其他主机来保证高可用

#### 8.1 部署高可用软件 haproxy, keepalived

在三台master节点安装haproxy, keepalived

```sh
yum install keepalived haproxy -y
```

##### 8.1.1 配置 haproxy

```sh
cat >/etc/haproxy/haproxy.cfg<<"EOF"
global
 maxconn 2000
 ulimit-n 16384
 log 127.0.0.1 local0 err
 stats timeout 30s

defaults
 log global
 mode http
 option httplog
 timeout connect 5000
 timeout client 50000
 timeout server 50000
 timeout http-request 15s
 timeout http-keep-alive 15s

frontend monitor-in
 bind *:33305
 mode http
 option httplog
 monitor-uri /monitor

frontend k8s-master
 bind 0.0.0.0:16443
 bind 127.0.0.1:16443
 mode tcp
 option tcplog
 tcp-request inspect-delay 5s
 default_backend k8s-master

backend k8s-master
 mode tcp
 option tcplog
 option tcp-check
 balance roundrobin
 default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
 server  master01  192.168.127.10:6443 check
 server  master02  192.168.127.20:6443 check
 server  master03  192.168.127.30:6443 check
EOF
```

##### 8.1.2 配置 keepalived

注意三台主机需要设置不同的优先级 priority ，这样 keepalived 才会自动选择其中一台提供服务

```sh
#master01 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2 
   rise 1
}
vrrp_instance VI_1 {
   state MASTER
   interface ens33
   mcast_src_ip 192.168.127.10
   virtual_router_id 51
   priority 100
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass 123456
   }
   virtual_ipaddress {
       192.168.127.117
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master02 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
  interval 5
   weight -5
   fall 2 
   rise 1
}
vrrp_instance VI_1 {
   state BACKUP
   interface ens33
   mcast_src_ip 192.168.127.20
   virtual_router_id 51
   priority 99
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass 123456
   }
   virtual_ipaddress {
       192.168.127.117
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master03 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2 
   rise 1
}
vrrp_instance VI_1 {
   state BACKUP
   interface ens33
   mcast_src_ip 192.168.127.30
   virtual_router_id 51
   priority 88
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass 123456
   }
   virtual_ipaddress {
       192.168.127.117
   }
    track_script {
      chk_apiserver
   }
EOF
```

##### 8.1.3 健康检查脚本

```sh
cat > /etc/keepalived/check_apiserver.sh <<"EOF"
#!/bin/bash
err=0
for k in $(seq 1 3)
do
   check_code=$(pgrep haproxy)
   if [[ $check_code == "" ]]; then
       err=$(expr $err + 1)
       sleep 1
       continue
   else
       err=0
       break
   fi
done

if [[ $err != "0" ]]; then
   echo "systemctl stop keepalived"
   /usr/bin/systemctl stop keepalived
   exit 1
else
   exit 0
fi
EOF

chmod u+x /etc/keepalived/check_apiserver.sh
```

##### 8.1.4 启动服务

```
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
```

#### 8.2 搭建 etcd 集群

##### 8.2.1 签发 etcd 证书

首先准备etcd 证书请求文件 etcd-csr.json, etcd 仅部署在 10， 20 ，30 机器所以只需要三个IP:

```sh
cat > etcd-csr.json <<"EOF"
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.127.10",
    "192.168.127.20",
    "192.168.127.30"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Guangdong",
    "L": "ShenZhen",
    "O": "k8s",
    "OU": "system"
  }]
}
EOF
```

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-csr.json | cfssljson -bare etcd-peer

for i in master01 master02 master03;do scp /opt/cert/ca*.pem $i:/opt/etcd/cert;done
for i in master01 master02 master03;do scp /opt/cert/etcd-peer*.pem $i:/opt/etcd/cert;done
```

##### 8.2.2 部署 etcd 集群

创建 etcd 用户：

```
useradd -s /sbin/nologin -M etcd
```

安装etcd :

```sh
wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
tar -xvf etcd-v3.5.0-linux-amd64.tar.gz
mv etcd-v3.5.0-linux-amd64 /opt/
cd /opt/
mv etcd-v3.5.0-linux-amd64 etcd-v3.5.0
ln -s etcd-v3.5.0/ /opt/etcd
cd etcd
mkdir -p /opt/etcd/cert /data/etcd /data/etcd/etcd-server /data/logs/etcd-server
cd cert/

scp harbor:/opt/cert/ca.pem .
scp harbor:/opt/cert/etcd-peer.pem .
scp harbor:/opt/cert/etcd-peer-key.pem .
```

首先设置etcd启动脚本：/opt/etcd/etcd-server-startup.sh 

> 注意最后的 \ 后面不能有空格
>
> --name 参数需要与--initial-cluster中的节点名称一致
>
> --enable-v2 不能少，因为flannel插件使用的是 V2 接口，而etcd从 v3.4 开始默认关闭了 v2 接口

```bash
cat > /opt/etcd/etcd-server-startup.sh <<"EOF"
#!/bin/sh
./etcd --name=etcd01 \
       --enable-v2 \
       --data-dir=/data/etcd/etcd-server \
       --listen-peer-urls=https://192.168.127.10:2380 \
       --listen-client-urls=https://192.168.127.10:2379,http://127.0.0.1:2379 \
       --listen-metrics-urls=http://127.0.0.1:2381 \
       --quota-backend-bytes=8000000000 \
       --initial-advertise-peer-urls=https://192.168.127.10:2380 \
       --advertise-client-urls=https://192.168.127.10:2379,http://127.0.0.1:2379 \
       --initial-cluster=etcd01=https://192.168.127.10:2380,etcd02=https://192.168.127.20:2380,etcd03=https://192.168.127.30:2380 \
       --initial-cluster-state=new \
       --initial-cluster-token=ea8cfe2bfe85b7e6c66fe190f9225838 \
       --cert-file=./cert/etcd-peer.pem \
       --key-file=./cert/etcd-peer-key.pem \
       --client-cert-auth \
       --trusted-ca-file=./cert/ca.pem \
       --peer-cert-file=./cert/etcd-peer.pem \
       --peer-key-file=./cert/etcd-peer-key.pem \
       --peer-client-cert-auth \
       --peer-trusted-ca-file=./cert/ca.pem
EOF
```

修改文件权限及目录所属用户：

```sh
chmod +x etcd-server-startup.sh
useradd -s /sbin/nologin -M etcd
chown -R etcd.etcd /opt/etcd-v3.5.0/
chown -R etcd.etcd /data/etcd/
chown -R etcd.etcd /data/logs/etcd-server/
```

创建启动service：

```sh
cat > /usr/lib/systemd/system/etcd.service <<"EOF"
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=
WorkingDirectory=/opt/etcd/
ExecStart=/opt/etcd/etcd-server-startup.sh
StandardOutput=append:/data/logs/etcd-server/out.log
StandardError=append:/data/logs/etcd-server/error.log
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 同样可以通过 supervisor 启动
# 安装完成后设置配置文件： /etc/supervisord.d/etcd-server.ini
cat > /etc/supervisord.d/etcd-server.ini << "EOF"
[program:etcd-server1]
command=/opt/etcd/etcd-server-startup.sh
numprocs=1
directory=/opt/etcd
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=etcd
redirect_stderr=true
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF

# 接着执行以下命令启动etcd
supervisorctl update
supervisorctl status
```

查看集群状态，全部是 healthy：

```
./etcdctl --cacert=cert/ca.pem --cert=cert/etcd-peer.pem --key=cert/etcd-peer-key.pem  --endpoints=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 endpoint health

[root@master01 etcd]# ./etcdctl --cacert=cert/ca.pem --cert=cert/etcd-peer.pem --key=cert/etcd-peer-key.pem  --endpoints=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 endpoint health
https://192.168.127.10:2379 is healthy: successfully committed proposal: took = 15.296012ms
https://192.168.127.30:2379 is healthy: successfully committed proposal: took = 99.475887ms
https://192.168.127.20:2379 is healthy: successfully committed proposal: took = 169.215014ms

[root@master01 etcd]# ./etcdctl --cacert=cert/ca.pem --cert=cert/etcd-peer.pem --key=cert/etcd-peer-key.pem  --endpoints=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 endpoint  status --write-out=table
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.127.10:2379 | 1c7bb24a23bc3671 |   3.5.0 |   20 kB |      true |      false |      1679 |       3372 |               3372 |        |
| https://192.168.127.20:2379 | f37fd8ae634a8102 |   3.5.0 |   20 kB |     false |      false |      1679 |       3372 |               3372 |        |
| https://192.168.127.30:2379 | f049b8836150b9f8 |   3.5.0 |   20 kB |     false |      false |      1679 |       3372 |               3372 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

#### 8.3 部署 apiserver

##### 8.3.1 下载k8s二进制文件

首先进入 [k8s官网](https://github.com/kubernetes/kubernetes/releases) 找到需要下载的版本, 然后点击相应的 [the CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.22.md) 进入到页面找到 Server Binaries 对应的软件包下载。

```
wget https://dl.k8s.io/v1.21.2/kubernetes-server-linux-amd64.tar.gz
tar xf kubernetes-server-linux-amd64.tar.gz -C /opt
cd /opt/
mv kubernetes kubernetes-v1.22.2
ln -s /opt/kubernetes-v1.22.2/ /opt/kubernetes
cd kubernetes-v1.22.2

# 删除源码包 , 删除无用镜像
rm -fr kubernetes-src.tar.gz
rm -fr server/bin/*.tar 
rm -fr server/bin/*_tag
```



##### 8.3.2 签发 apiserver 请求etcd使用的 client 证书

该证书是为apiserver与etcd集群通讯使用的证书， apiserver作为 client端， etcd作为server端。进入harbor机器，创建生成证书签名请求的json文件 /opt/cert/etcd-client-csr.json：

```sh
cat > /opt/cert/etcd-client-csr.json << "EOF"
{
    "CN": "etcd-client",
    "hosts":[
       "192.168.127.10",
       "192.168.127.20",
       "192.168.127.30"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C":"CN",
           "ST":"GuangDong",
           "L": "ShenZhen",
           "O": "k8s",
           "OU": "system"
        }
    ]
}
EOF
```

执行命令：

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client etcd-client-csr.json | cfssljson -bare etcd-client
```

##### 8.3.3 签发 apiserver 服务端证书

该证书是为其他客户端与apiserver通讯使用的证书， 其他客户端作为 client 端， apiserver 作为 server 端。进入harbor机器，创建生成证书签名请求的json文件 /opt/cert/apiserver-server-csr.json：

```sh
cat > /opt/cert/apiserver-server-csr.json << "EOF"
{
    "CN": "kubernetes",
    "hosts":[
       "127.0.0.1",
       "kubernetes.default",
       "kubernetes.default.svc",
       "kubernetes.default.svc.cluster",
       "kubernetes.default.svc.cluster.local",
       "192.168.127.10",
       "192.168.127.20",
       "192.168.127.30",
       "192.168.127.40",
       "192.168.127.50",
       "192.168.127.60",
       "192.168.127.117",
       "192.168.127.180",
       "192.168.127.190",
       "192.168.127.200",
	   "192.168.127.210",
       "192.168.127.220"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C":"CN",
           "ST":"shenzhen",
           "L": "shenzhen",
           "O": "nanshan",
           "OU": "ops"
        }
    ]
}
EOF
```

执行命令

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server apiserver-server-csr.json | cfssljson -bare apiserver-server
```

进入master01将证书及私钥文件复制到 apiserver机器

```sh
cd server/bin/
mkdir config
mkdir cert && cd cert
scp harbor:/opt/cert/ca*.pem .
scp harbor:/opt/cert/etcd-client*.pem .
scp harbor:/opt/cert/apiserver-server*.pem .
# 将证书复制到所有主节点
for i in master01 master02 master03;do scp /opt/cert/apiserver-server*.pem $i:/opt/kubernetes/server/bin/cert;done
```

在 server/bin 下创建 config文件夹， 并创建审计配置文件 audit.yml

```yml
apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
```

##### 8.3.4 创建启动脚本

创建脚本前需要先重建 token ，用于kubelet 向apiserver申请证书，并通过 controller-manager颁发证书后才可以让kubelet与apiserver正常通信

```sh
echo "`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`,kubelet-bootstrap,10001,\"system:bootstrappers\"" > /opt/kubernetes/server/bin/config/token.csv
```

创建apiserver 启动脚本 /opt/kubernetes/server/bin/kube-apiserver.sh ：

> 注意最后的 \ 后面不能有空格
>
> 如果同时提供了 `--client-ca-file` 和 `--requestheader-client-ca-file`， 则首先检查 `--requestheader-client-ca-file` CA，然后再检查 `--client-ca-file`。 通常，这些选项中的每一个都使用不同的 CA（根 CA 或中间 CA）。 常规客户端请求与 `--client-ca-file` 相匹配，而聚合请求要与 `--requestheader-client-ca-file` 相匹配。 但是，如果两者都使用同一个 CA，则通常会通过 `--client-ca-file` 传递的客户端请求将失败，因为 CA 将与 `--requestheader-client-ca-file` 中的 CA 匹配，但是通用名称 `CN=` 将不匹配 `--requestheader-allowed-names` 中可接受的通用名称之一。 这可能导致你的 kubelet 和其他控制平面组件以及最终用户无法向 Kubernetes apiserver 认证。

- `--requestheader-client-ca-file`：用于签名 `--proxy-client-cert-file` 和 `--proxy-client-key-file` 指定的证书；在启用了 metric aggregator 时使用；暂时去除
- requestheader-allowed-names
- kubelet-client 相关证书可与etcd-certfile共用同一套证书，该证书都是以apiserver作为客户端分布请求 kubelet 与 etcd 所使用的证书

```bash
cat > /opt/kubernetes/server/bin/kube-apiserver.sh << "EOF"
#!/bin/bash
./kube-apiserver \
 --apiserver-count=3 \
 --advertise-address=192.168.127.10 \
 --enable-aggregator-routing=true \
 --audit-log-path=/data/logs/kubernetes/kube-apiserver/audit.log \
 --audit-policy-file=./config/audit.yml \
 --authorization-mode=RBAC,Node \
 --runtime-config=api/all=true \
 --enable-bootstrap-token-auth \
 --token-auth-file=config/token.csv \
 --requestheader-group-headers=X-Remote-Group \
 --requestheader-username-headers=X-Remote-User \
 --requestheader-extra-headers-prefix=X-Remote-Extra- \
 --client-ca-file=./cert/ca.pem \
 --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
 --etcd-cafile=./cert/ca.pem \
 --etcd-certfile=./cert/etcd-client.pem \
 --etcd-keyfile=./cert/etcd-client-key.pem \
 --etcd-servers=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 \
 --service-account-issuer=k8s.cc \
 --service-account-key-file=./cert/ca-key.pem \
 --service-account-signing-key-file=./cert/ca-key.pem \
 --service-cluster-ip-range=192.168.0.0/16 \
 --service-node-port-range=3000-49999 \
 --kubelet-client-certificate=./cert/etcd-client.pem \
 --kubelet-client-key=./cert/etcd-client-key.pem \
 --log-dir=/data/logs/kubernetes/kube-apiserver \
 --tls-cert-file=./cert/apiserver-server.pem \
 --tls-private-key-file=./cert/apiserver-server-key.pem \
 --anonymous-auth=false \
 --v=2
 EOF
```



````
chmod +x kube-apiserver.sh 
mkdir -p /data/logs/kubernetes/kube-apiserver
````

创建apiserver对应 的supervisor启动文件 /usr/lib/systemd/system/kube-apiserver.service :

```sh
# 通过 systemd 的方式启动
cat > /usr/lib/systemd/system/kube-apiserver.service <<"EOF"
[Unit]
Description=Kube Api Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=
WorkingDirectory=/opt/kubernetes/server/bin
ExecStart=/opt/kubernetes/server/bin/kube-apiserver.sh
StandardOutput=/data/logs/kubernetes/kube-apiserver/apiserver.log
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now kube-apiserver
systemctl status kube-apiserver

# 通过 supervisor 方式启动
# 创建apiserver对应 的supervisor启动文件 /etc/supervisord.d/kube-apiserver.ini :
cat > /etc/supervisord.d/kube-apiserver.ini << "EOF"
[program:kube-apiserver1]
command=/opt/kubernetes/server/bin/kube-apiserver.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF

# 创建完成后执行命令启动
supervisorctl update
```

##### 8.3.5 使用 kubectl 查看集群状态

为了使得用户具备管理权限，同样需要先签发证书，注意证书请求文件中的 names -> O 属性必须是"system:masters"

```sh
cat > kubectl-admin-csr.json << "EOF"
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangDong",
      "L": "ShenZhen",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
EOF
```

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kubectl-admin-csr.json | cfssljson -bare kubectl-admin

# 将证书复制到所有主节点
for i in master01 master02 master03;do scp /opt/cert/kubectl-admin*.pem $i:/opt/kubernetes/server/bin/cert;done
```

```sh
ln -s /opt/kubernetes-v1.22.2/server/bin/kubectl /usr/bin/kubectl

# 设置集群参数 注意此处使用的 server IP 是 192.168.127.117，高可用
kubectl config set-cluster kubernetes \
 --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
 --embed-certs=true \
 --server=https://192.168.127.117:16443 \
 --kubeconfig=/opt/kubernetes/server/bin/config/admin.config
 
 kubectl config set-credentials admin \
 --client-certificate=/opt/kubernetes/server/bin/cert/kubectl-admin.pem \
 --client-key=/opt/kubernetes/server/bin/cert/kubectl-admin-key.pem \
 --embed-certs=true \
 --kubeconfig=/opt/kubernetes/server/bin/config/admin.config
 
 kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=/opt/kubernetes/server/bin/config/admin.config

kubectl config use-context kubernetes --kubeconfig=/opt/kubernetes/server/bin/config/admin.config
 
mkdir ~/.kube
cp /opt/kubernetes/server/bin/config/admin.config ~/.kube/config

#绑定集群管理员角色
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user admin --kubeconfig=/root/.kube/config

export KUBECONFIG=$HOME/.kube/config

kubectl cluster-info
kubectl get componentstatuses
kubectl get all --all-namespaces
```

```
[root@localhost bin]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.127.117:16443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

[root@localhost bin]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                       ERROR
controller-manager   Unhealthy   Get "https://127.0.0.1:10257/healthz":  connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz":  connection refused    
etcd-2               Healthy     {"health":"true","reason":""}                                                               
etcd-1               Healthy     {"health":"true","reason":""}                                                               
etcd-0               Healthy     {"health":"true","reason":""}                                                                  
[root@localhost bin]# kubectl get all --all-namespaces
NAMESPACE   NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   192.168.0.1   <none>        443/TCP   3h7m
```

##### 8.3.6 配置kubectl子命令补全功能

```sh
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc'  
source $HOME/.bash_profile
```

#### 8.4 部署 controller-manager

##### 8.4.1 生成证书

```sh
cat > /opt/cert/kube-controller-manager-csr.json << "EOF"
{
    "CN": "system:kube-controller-manager",
    "hosts":[
       "127.0.0.1",
       "192.168.127.10",
       "192.168.127.20",
       "192.168.127.30",
       "192.168.127.40",
       "192.168.127.50",
       "192.168.127.60"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C":"CN",
           "ST":"GuangDong",
           "L": "ShenZhen",
           "O": "system:kube-controller-manager",
           "OU": "system"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

# 将证书复制到所有主节点
for i in master01 master02 master03;do scp /opt/cert/kube-controller-manager*.pem $i:/opt/kubernetes/server/bin/cert;done
```

>CN 为 system:kube-controller-manager , O 为 system:kube-controller-manager
>
>kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限

##### 8.4.2 生成配置文件

```sh
kubectl config set-cluster kubernetes --certificate-authority=cert/ca.pem --embed-certs=true --server=https://192.168.127.117:16443 --kubeconfig=config/kube-controller-manager.config

kubectl config set-credentials system:kube-controller-manager --client-certificate=cert/kube-controller-manager.pem --client-key=cert/kube-controller-manager-key.pem --embed-certs=true --kubeconfig=config/kube-controller-manager.config

kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=config/kube-controller-manager.config

kubectl config use-context system:kube-controller-manager --kubeconfig=config/kube-controller-manager.config
```

##### 8.4.3 创建启动脚本

>注意：参数 --cluster-signing-xxx 用于为 kubelet 颁发证书时使用，与apiserver 保持一致

```SH
cat > /opt/kubernetes/server/bin/kube-controller-manager.sh << "EOF"
#!/bin/sh
./kube-controller-manager \
  --allocate-node-cidrs=true \
  --cluster-cidr=172.7.0.0/16 \
  --cluster-name=kubernetes \
  --secure-port=10257 \
  --controllers=*,bootstrapsigner,tokencleaner \
  --authentication-kubeconfig=/opt/kubernetes/server/bin/config/kube-controller-manager.config \
  --authorization-kubeconfig=/opt/kubernetes/server/bin/config/kube-controller-manager.config \
  --kubeconfig=/opt/kubernetes/server/bin/config/kube-controller-manager.config \
  --leader-elect=true \
  --use-service-account-credentials=true \
  --log-dir=/data/logs/kubernetes/kube-controller-manager \
  --service-account-private-key-file=./cert/ca-key.pem \
  --service-cluster-ip-range=192.168.0.0/16 \
  --root-ca-file=./cert/ca.pem \
  --client-ca-file=./cert/ca.pem \
  --requestheader-client-ca-file=./cert/ca.pem \
  --tls-cert-file=./cert/kube-controller-manager.pem \
  --tls-private-key-file=./cert/kube-controller-manager-key.pem \
  --cluster-signing-cert-file=./cert/ca.pem \
  --cluster-signing-key-file=./cert/ca-key.pem \
  --v=2
EOF

mkdir -p /data/logs/kubernetes/kube-controller-manager
chmod +x /opt/kubernetes/server/bin/kube-controller-manager.sh
```

设置supervisor服务，/etc/supervisord.d/kube-controller-manager.ini

```SH
cat > /etc/supervisord.d/kube-controller-manager.ini << "EOF"
[program:kube-controller-manager1]
command=/opt/kubernetes/server/bin/kube-controller-manager.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF
```

启动 kube-controller-manager 并查看状态

```
supervisorctl update
supervisorctl status

# 查看controller集群状态
kubectl get cs
```

启动完成后继续在其他两台机器进行部署。

#### 8.5 部署 kube-schesuler

 ##### 8.5.1 生成证书

```sh
cat > /opt/cert/kube-scheduler-csr.json << "EOF"
{
    "CN": "system:kube-scheduler",
    "hosts":[
       "127.0.0.1",
       "192.168.127.10",
       "192.168.127.20",
       "192.168.127.30"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C":"CN",
           "ST":"GuangDong",
           "L": "ShenZhen",
           "O": "system:kube-scheduler",
           "OU": "system"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kube-scheduler-csr.json | cfssljson -bare kube-scheduler

# 将证书复制到所有主节点
for i in master01 master02 master03;do scp /opt/cert/kube-scheduler*.pem $i:/opt/kubernetes/server/bin/cert;done
```

##### 8.5.2 创建配置文件

```sh
cd /opt/kubernetes/server/bin

kubectl config set-cluster kubernetes --certificate-authority=cert/ca.pem --embed-certs=true --server=https://192.168.127.117:16443 --kubeconfig=config/kube-scheduler.config

kubectl config set-credentials system:kube-scheduler --client-certificate=cert/kube-scheduler.pem --client-key=cert/kube-scheduler-key.pem --embed-certs=true --kubeconfig=config/kube-scheduler.config

kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=config/kube-scheduler.config

kubectl config use-context system:kube-scheduler --kubeconfig=config/kube-scheduler.config
```

##### 8.5.3 创建启动脚本

```SH
cat > /opt/kubernetes/server/bin/kube-scheduler.sh << "EOF"
#!/bin/sh
./kube-scheduler \
  --kubeconfig=/opt/kubernetes/server/bin/config/kube-scheduler.config
  --leader-elect=true \
  --log-dir=/data/logs/kubernetes/kube-scheduler \
  --v=2
EOF

mkdir -p /data/logs/kubernetes/kube-scheduler
chmod +x /opt/kubernetes/server/bin/kube-scheduler.sh
```

设置supervisor服务，/etc/supervisord.d/kube-scheduler.ini

```SH
cat > /etc/supervisord.d/kube-scheduler.ini << "EOF"
[program:kube-scheduler1]
command=/opt/kubernetes/server/bin/kube-scheduler.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF
```

启动 kube-scheduler 并查看状态

```
supervisorctl update
supervisorctl status

# 查看controller集群状态
kubectl get cs

[root@master01 bin]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""}   
etcd-2               Healthy   {"health":"true","reason":""}   
etcd-1               Healthy   {"health":"true","reason":""}  
```

启动完成后继续在其他两台机器进行部署。

#### 8.6 总结

至此，由三台master主机组成的高可用集群已经部署完成，外部统一使用虚拟IP 192.168.127.117 进行访问

|     IP地址      | 机器名称 |  机器配置   | 机器角色 |                           安装软件                           |
| :-------------: | :------: | :---------: | :------: | :----------------------------------------------------------: |
| 192.168.127.10  | master01 | 2C  2G  30G |  master  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 192.168.127.20  | master02 | 2C  2G  30G |  master  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 192.168.127.30  | master03 | 2C  2G  30G |  master  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 192.168.127.117 |    /     |      /      |  虚拟IP  | 由 HAProxy 和 keepalived 组成的 LB，由192.168.127.117统一路由到其他三个主节点 |

### 9. 部署harbor 

为了将harbor服务独立，所以需要基于前面已经配置好的虚拟机再复制一个VM出来单独部署harbor，IP 为 168.127.200. 接下来到 harbor [官网](https://github.com/goharbor/harbor/releases) 下载安装包 离线安装包，然后在 /opt创建文件夹  /opt/harbor  并将离线包解压到此目录

```sh
tar xvf harbor-offline-installer-v2.3.3.tgz -C /opt
```

解压完成后进入文件夹harbor, 把 harbor.tml.tmpl 复制一份配置文件 harbor.yml, 并修改如下内容:

> 注意先创建文件夹 /data/harbor, /data/habor/logs。
>
> 通知注意去掉 https 部分

```sh
hostname: harbor.k8s.cc

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 1080

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: 123456

# The default data volume
data_volume: /data/harbor

log:
   location: /data/harbor/logs

```

由于harbor需要docker-compose支持，所以需要安装 docker-compose

```sh
yum install docker-compose -y

# 安装docker-compose 完成后开始安装 harbor
./install.sh 

# 安装完成后进入docker即可发现多了很多容器
[root@vms2 harbor]# docker ps
CONTAINER ID   IMAGE                         ...              PORTS                                       NAMES
af2789eb0afa   goharbor/harbor-jobservice:v2.3.3   ...                                                harbor-jobservice
d02f1bba3b0b   goharbor/nginx-photon:v2.3.3        ...   0.0.0.0:1080->8080/tcp, :::1080->8080/tcp    nginx
f2143cc630a5   goharbor/harbor-core:v2.3.3         ...                                                harbor-core
2e340c0f0ee1   goharbor/registry-photon:v2.3.3     ...                                                registry
90cd6577003f   goharbor/harbor-db:v2.3.3            ...                                               harbor-db
3a9aa0130786   goharbor/harbor-registryctl:v2.3.3   ...                                               registryctl
6b57d74dd9d6   goharbor/redis-photon:v2.3.3         ...                                               redis
03af85992be0   goharbor/harbor-portal:v2.3.3        ...                                               harbor-portal
3da5b3e90a38   goharbor/harbor-log:v2.3.3           ...  127.0.0.1:1514->10514/tcp                    harbor-log
```

由于集群需要从harbor拉取镜像，所以需要为harbor准备基础镜像 parse 

```sh
docker image pull kubernetes/pause
docker image tag kubernetes/pause:latest harbor.k8s.cc/public/pause:latest
docker login -u admin http://harbor.k8s.cc
docker image push harbor.k8s.cc/public/pause:latest
```

#### 9.1 部署域名解析服务

由于域名解析服务也是放在harbor 所以把域名部署服务放在此章节，保持操作连贯性

```
yum install bind -y
```

修改配置文件 vim /etc/named.conf， 即在 harbor 机器监听服务. forwarders 表示自身无法解析的域名会转发到其他服务去解析

```sh
listen-on port 53 { 192.168.127.180; };
allow-query     { any; };
forwarders      { 114.114.114.114;};

recursion yes;
dnssec-enable no;
dnssec-validation no;
```

设置完成后使用命令 named-checkconf 检查是否配置成功

##### 9.1.1 配置正反向解析文件 

配置正反向解析域，进入 /etc/named.rfc1912.zones

> 注意 反向解析中的 zone 名称一定要写成  127.168.192.in-addr.arpa ，其中前面部分是IP地址的反写。若没有设置IP地址反写则在解析时将不会有 answer ！！！ 至于 file 的命名可以任意命名。

```sh
cat >> /etc/named.rfc1912.zones << "EOF"
// 正向解析
zone "k8s.cc" IN {
     type  master;
     file  "k8s.cc.zone";
     allow-update { 192.168.127.180; };
};
// 反向解析
zone "127.168.192.in-addr.arpa" IN {
     type  master;
     file  "192.168.127.zone";
     allow-update { none; };
};
EOF
```

#### 2.1 配置正向解析文件

 进入 /var/named/k8s.cc.zone

```sh
cat > /var/named/k8s.cc.zone << "EOF"
$ORIGIN k8s.cc.
$TTL 600    ; 10 mins
@ IN SOA dns.k8s.cc. 182867664.qq.com. (
        2021101401  ; serial
        10800       ; refresh (3 hours)
        900         ; retry (15 mins)
        604800      ; expire (1 week)
        86400       ; minimum (1 day)
)
  IN   NS    dns.k8s.cc.
dns         A    192.168.127.180
master01    A    192.168.127.10
master02    A    192.168.127.20
master03    A    192.168.127.30
node01      A    192.168.127.40
node02      A    192.168.127.50
node03      A    192.168.127.60
harbor      A    192.168.127.180
EOF
```

#### 2.2 配置反向解析文件

 进入 /var/named/192.168.127.in-addr.arpa

```sh
cat > /var/named/192.168.127.in-addr.arpa << "EOF"
$TTL 600    ; 10 mins
@ IN SOA dns.k8s.cc. 182867664.qq.com. (
        2021101401  ; serial
        10800       ; refresh (3 hours)
        900         ; retry (15 mins)
        604800      ; expire (1 week)
        86400       ; minimum (1 day)
)
@    IN   NS    dns.k8s.cc.
10   IN   PTR   master01.k8s.cc.
20   IN   PTR   master02.k8s.cc.
30   IN   PTR   master03.k8s.cc.
40   IN   PTR   node01.k8s.cc.
50   IN   PTR   node02.k8s.cc.
60   IN   PTR   node03.k8s.cc.
180  IN   PTR   harbor.k8s.cc.
EOF
```

准备好上述三个文件后检查配置是否正确： named-checkconf， 然后再执行开启命令：

```sh
named-checkconf 
named-checkzone "k8s.cc" /var/named/k8s.cc.zone
named-checkzone "k8s.cc" /var/named/192.168.127.in-addr.arpa

# 设置文件权限
chown root:named /var/named/*.zone
chmod 640 /var/named/*.zone

systemctl start named
```

#### 2.3 正反向测试

首先安装 dig正向解析工具

```sh
 yum install bind-utils -y
```

```sh
cp /etc/sysconfig/network-scripts/ifcfg-ens33 /etc/sysconfig/network-scripts/ifcfg-ens33 .bak
cat >> /etc/sysconfig/network-scripts/ifcfg-ens33 << "EOF"

DNS1="192.168.127.180"
DNS2="8.8.8.8"
EOF

systemctl restart network
```



git

正向测试：

```sh
# 参数 @ 后面指定了DNS服务器，在已经指定DNS为bind服务的时候可以忽略
dig -t A master01.k8s.cc @192.168.127.10 +short

# 查询域名记录
dig -t NS k8s.cc @192.168.127.180

dig -t A master01.k8s.cc
dig -x 192.168.127.180
```

反向测试：

```
nslookup harbor.k8s.cc
nslookup master01.k8s.cc
```

### 

### 10 部署Node工作节点集群

Node节点是实际运行Pod的节点，故需要安装 docker。在xshell6 同时连接三个node机器执行下面操作

#### 10.1 安装docker

```sh
# 配置加速地址，防止安装太慢
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 

yum install -y docker-ce
systemctl start docker

cat > /etc/docker/daemon.json << "EOF"
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": ["https://jfixgq82.mirror.aliyuncs.com"],
  "insecure-registries": ["registry.access.redhat.com", "query.io", "harbor.k8s.cc"]
}
EOF

systemctl enable --now docker
systemctl restart docker
```

#### 10.2 部署 kubelet

为了让 kubelet 可以访问到master集群中的apiserver服务，kubelet需要知道连接参数及证书。所以首先需要生成证书文件。在生成证书前需要认识下 Kubernetes TLS bootstrapping。当集群开启了 TLS 认证后，每个节点的 kubelet 组件都要使用由 apiserver 使用的 CA 签发的有效证书才能与 apiserver 通讯；此时如果节点多起来，为每个节点单独签署证书将是一件非常繁琐的事情；TLS bootstrapping 功能就是让 kubelet 先使用一个预定的低权限用户连接到 apiserver，然后向 apiserver 申请证书，kubelet 的证书由 apiserver 动态签署；

##### 10.2.1 签发 kubelet证书

首先进入harbor机器 /opt/cert 创建证书请求文件 kubelet-csr.json

```sh
cat > kubelet-csr.json << "EOF"
{
    "CN": "system:node-bootstrapper",
    "hosts":[
       "192.168.127.10",
       "192.168.127.20",
       "192.168.127.30",
       "192.168.127.40",
       "192.168.127.50",
       "192.168.127.60",
       "192.168.127.117",
       "192.168.127.180",
       "192.168.127.190",
       "192.168.127.200",
	   "192.168.127.210",
       "192.168.127.220"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C":"CN",
           "ST": "GuangDong",
           "L": "ShenZhen",
           "O": "system:bootstrappers",   
           "OU": "ops"
        }
    ]
}
EOF
```

基于已有根证书开始签发 kubelet 证书

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server kubelet-csr.json | cfssljson -bare kubelet
```

生成证书后将证书拷贝到三个机器的目录 /opt/kubernetes/server/bin/cert 中

```sh
for i in master01 master02 master03;do scp /opt/cert/kubelet*.pem $i:/opt/kubernetes/server/bin/cert;done

for i in node01 node02 node03;do scp /opt/cert/kubelet*.pem $i:/opt/kubernetes/server/bin/cert;done
```

##### 10.2.2 在主节点生成配置文件及脚本

进入master集群中任一节点生成 kubelet-bootstrap.config， kubelet.json文件， 注意 BOOTSTRAP_TOKEN 参数不能少，连接apiserver时会出现证书异常的问题

```sh
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /opt/kubernetes/server/bin/config/token.csv)

kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
--embed-certs=true --server=https://192.168.127.117:16443 \
--kubeconfig=config/kubelet-bootstrap.config

kubectl config set-credentials kubelet-bootstrap \
--token=${BOOTSTRAP_TOKEN} \
--client-certificate=/opt/kubernetes/server/bin/cert/kubelet.pem \
--client-key=/opt/kubernetes/server/bin/cert/kubelet-key.pem \
--embed-certs=true \
--kubeconfig=config/kubelet-bootstrap.config

kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=config/kubelet-bootstrap.config

kubectl config use-context default --kubeconfig=config/kubelet-bootstrap.config


kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap --kubeconfig=config/kubelet-bootstrap.config
```



```sh
# 创建配置文件
cat > config/kubelet.json << "EOF"
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/opt/kubernetes/bin/cert/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",                    
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.12.0.2"]
}
EOF
```

创建启动脚本：

> 注意： 配置文件 config/kubelet.kubeconfig 会自动生成,  用于连接 apiserver

```SH
cat > /opt/kubernetes/server/bin/kubelet.sh << "EOF"
#!/bin/sh
./kubelet \
  --bootstrap-kubeconfig=config/kubelet-bootstrap.config \
  --kubeconfig=config/kubelet.kubeconfig \
  --hostname-override=node01 \
  --config=config/kubelet.json \
  --cert-dir=/opt/kubernetes/bin/cert \
  --pod-infra-container-image=harbor.k8s.cc/public/pause:latest \
  --rotate-certificates \
  --network-plugin=cni \
  --log-dir=/data/logs/kubernetes/kubelet \
  --v=2
EOF

mkdir -p /data/logs/kubernetes/kubelet
chmod +x /opt/kubernetes/server/bin/kubelet.sh
```

设置supervisor服务，/etc/supervisord.d/kubelet.ini

```SH
cat > /etc/supervisord.d/kubelet.ini << "EOF"
[program:kubelet1]
command=/opt/kubernetes/bin/kubelet.sh
numprocs=1
directory=/opt/kubernetes/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kubelet/kubelet.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF
```

```sh
# 在 master节点为用户授权请求签发证书
cat >> config/kubelet-bootstrap-clusterrolebinding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io  
EOF
kubectl apply -f config/kubelet-bootstrap-rbac.yaml
```

将配置及脚本复制到工作节点

```sh
# 将master节点的二进制文件复制到工作节点目录
for i in master01 master02 master03;do scp * $i:/opt/kubernetes/bin/;done

for i in node01 node02 node03;do scp config/kubelet-bootstrap.config $i:/opt/kubernetes/bin/config;done

for i in node01 node02 node03;do scp config/kubelet.json $i:/opt/kubernetes/bin/config;done

for i in node01 node02 node03;do scp kubelet.sh $i:/opt/kubernetes/bin/;done

for i in node01 node02 node03;do scp /etc/supervisord.d/kubelet.ini $i:/etc/supervisord.d/;done
```



##### 10.2.3 部署worker 节点

```bash
# 在工作节点创建目录
mkdir -p /data/logs/kubernetes/kubelet
mkdir -p /opt/kubernetes/server/bin/cert
mkdir -p /opt/kubernetes-v1.22.2/bin/
ln -s /opt/kubernetes-v1.22.2 /opt/kubernetes

# 启动 kebulet
supervisorctl update
```

若日志出现

>  "Failed to connect to apiserver" err="Get \"https://192.168.127.117:16443/healthz?timeout=1s\": x509: certificate signed by unknown authority (possibly because of \"crypto/rsa: verification error\" while trying to verify candidate authority certificate \"kubernetes\") 

则有可能是 token 没有在 kubelet-bootstrap.config 配置文件没有配置 token , 或者有部分证书没有同步。

```sh
# 重新生成证书
for i in master01 master02 master03;do scp ca*.pem $i:/opt/kubernetes/server/bin/cert;done
for i in master01 master02 master03;do scp apiserver-server*.pem $i:/opt/kubernetes/server/bin/cert;done
for i in master01 master02 master03;do scp etcd-client*.pem $i:/opt/kubernetes/server/bin/cert;done
```

待日志正常后进入master任一节点即可查看到工作节点的证书请求：

```sh
[root@mater01 bin]# kubectl get csr
NAME                    AGE     SIGNERNAME                   REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-BbHAG2ba...   9m52s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
node-csr-ttkk_kWj...   8m50s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
node-csr-ywB_XBQN...   6m48s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

##### 10.2.4 审批加入工作节点

```sh
# 单个审批
kubectl certificate approve node-csr-BbHAG2ba...

# 批量审批
kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

# 查看节点状态
[root@master03 bin]# kubectl get node
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   10m   v1.22.2
node02   NotReady   <none>   10m   v1.22.2
node03   NotReady   <none>   40s   v1.22.2
```

#### 10.3 部署 kube-proxy

##### 10.3.1 申请证书

```sh
cat > kube-proxy-csr.json << "EOF"
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hubei",
      "L": "Wuhan",
      "O": "k8s",
      "OU": "system"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kube-proxy-csr.json | cfssljson -bare kube-proxy

for i in node01 node02 node03;do scp /opt/cert/kube-proxy*.pem $i:/opt/kubernetes/bin/cert;done

for i in master01 master02 master03;do scp /opt/cert/kube-proxy*.pem $i:/opt/kubernetes/server/bin/cert;done
```

##### 10.3.2 生成配置文件

进入任一master节点生成 kube-proxy.config 配置文件 并复制到工作节点

```sh
cd /opt/kubernetes/server/bin
kubectl config set-cluster kubernetes --certificate-authority=cert/ca.pem --embed-certs=true --server=https://192.168.127.117:16443 --kubeconfig=config/kube-proxy.config

kubectl config set-credentials kube-proxy --client-certificate=cert/kube-proxy.pem --client-key=cert/kube-proxy-key.pem --embed-certs=true --kubeconfig=config/kube-proxy.config

kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=config/kube-proxy.config

kubectl config use-context default --kubeconfig=config/kube-proxy.config

for i in node01 node02 node03;do scp config/kube-proxy.config $i:/opt/kubernetes/bin/config;done
```

生成 kube-proxy.yml 配置,  注意 所有的 *bindAddress 都是工作节点 IP

```sh
cat > config/kube-proxy.yml << "EOF"
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /opt/kubernetes/bin/config/kube-proxy.config
clusterCIDR: 172.7.0.0/16
hostnameOverride: node01
healthzBindAddress: 0.0.0.0:10256
metricsBindAddress: 0.0.0.0:10249
mode: "ipvs"
EOF

for i in node01 node02 node03;do scp config/kube-proxy.yml $i:/opt/kubernetes/bin/config;done
```

##### 10.3.3 创建脚本

```SH
cat > /opt/kubernetes/server/bin/kube-proxy.sh << "EOF"
#!/bin/sh
./kube-proxy \
  --config=config/kube-proxy.yml \
  --log-dir=/data/logs/kubernetes/kube-proxy \
  --v=2
EOF

mkdir -p /data/logs/kubernetes/kube-proxy
chmod +x /opt/kubernetes/server/bin/kube-proxy.sh

for i in node01 node02 node03;do scp kube-proxy.sh $i:/opt/kubernetes/bin;done
```

设置supervisor服务，/etc/supervisord.d/kube-proxy.ini

```SH
cat > /etc/supervisord.d/kube-proxy.ini << "EOF"
[program:kube-proxy1]
command=/opt/kubernetes/bin/kube-proxy.sh
numprocs=1
directory=/opt/kubernetes/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF

for i in node01 node02 node03;do scp /etc/supervisord.d/kube-proxy.ini $i:/etc/supervisord.d/;done
```

```sh
# 启动 kube-proxy
supervisorctl update
```

### 11. 安装网络插件 flannel

到 flannel 官网 下载安装包后解压：

```sh
mkdir flannel-v0.14.0
tar -xvf flannel-v0.14.0-linux-amd64.tar.gz -C flannel-v0.14.0
ln -s /opt/flannel-v0.14.0 /opt/flannel
cd flannel
```

#### 11.1 将网络信息添加到 etcd

进入master任一节点设置

> - Network：用于指定 Flannel 地址池
> - SubnetLen：用于指定分配给单个宿主机的 docker0 的 ip 段的子网掩码的长度
> - SubnetMin：用于指定最小能够分配的 ip 段
> - SudbnetMax：用于指定最大能够分配的 ip 段，在上面的示例中，表示每个宿主机可以分配一个 24 位掩码长度的子网，可以分配的子网从 172.7.1.0/24 到 172.7.20.0/24，也就意味着在这个网段中，最多只能有 20 台宿主机
> - Backend：用于指定数据包以什么方式转发，默认为 udp 模式，host-gw 模式性能最好，但不能跨宿主机网络

```sh
etcdctl --cacert=/opt/etcd/cert/ca.pem --cert=/opt/etcd/cert/etcd-peer.pem --key=/opt/etcd/cert/etcd-peer-key.pem  --endpoints=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 put /coreos.com/network/config '{"Network": "172.7.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
```

#### 11.2 创建 flannel 启动脚本


> 由于使用的是systemd , 故 脚本使用到的文件必须是绝对路径，否则报错

```sh
mkdir /opt/flannel/cert
for i in master01 master02 master03;do scp $i:/opt/etcd/cert/ca.pem /opt/flannel/cert/;done
for i in master01 master02 master03;do scp $i:/opt/etcd/cert/etcd*.pem /opt/flannel/cert/;done

cat > /opt/flannel/flanneld.sh << EOF
#!/bin/sh
/opt/flannel/flanneld \\
  --etcd-endpoints=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 \\
  --etcd-keyfile=/opt/flannel/cert/etcd-client-key.pem \\
  --etcd-certfile=/opt/flannel/cert/etcd-client.pem \\
  --etcd-cafile=/opt/flannel/cert/ca.pem \\
  --iface=ens33
EOF

chmod +x  /opt/flannel/flanneld.sh
```

创建后台服务, 其中 参数 -d /run/flannel/docker 的作用是将环境变量写入到文件中，然后docker启动时会自动加载该配置，所以还需要再docker启动服务进行配置

```sh
cat > /usr/lib/systemd/system/flanneld.service << EOF
[Unit]
Description=Flannel
Documentation=https://github.com/coreos/flannel
After=network.target
After=network-online.target
Wants=network-online.target
Before=docker.service

[Service]
User=root
Type=simple
ExecStart=/opt/flannel/flanneld.sh
ExecStartPost=/opt/flannel/mk-docker-opts.sh  -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF

mkdir -p /run/flannel
touch /run/flannel/docker
```

启动flannel服务

```sh
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
```

启动失败，遇到错误：

> Couldn't fetch network config: client: response is invalid json. The endpoint is probably not valid etcd cluster endpoint

这是因为 flannel 使用的是 etcd 的 V2 接口，而etcd从v3.4开始默认关闭了v2接口，所以需要通过参数 --enable-v2 打开。故需要调整 etcd 服务，并重新设置flannel的网络参数：

```sh
ETCDCTL_API=2 etcdctl --ca-file=/opt/etcd/cert/ca.pem --cert-file=/opt/etcd/cert/etcd-peer.pem --key-file=/opt/etcd/cert/etcd-peer-key.pem  --endpoints=https://192.168.127.10:2379,https://192.168.127.20:2379,https://192.168.127.30:2379 set /coreos.com/network/config '{"Network": "172.7.0.0/16","Subnet": "172.7.21.1/24", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
```

再次重启即可，重启完成后发现服务已经正常，此时查看网卡多了 flannel.1

```
[root@node01 ~]# ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ec:b9:44 brd ff:ff:ff:ff:ff:ff
    inet 192.168.127.40/24 brd 192.168.127.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::7b75:80ad:659a:afc2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 6e:26:06:8a:ba:56 brd ff:ff:ff:ff:ff:ff
    inet 172.7.21.0/32 brd 172.7.21.0 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::6c26:6ff:fe8a:ba56/64 scope link 
       valid_lft forever preferred_lft forever
......
```

#### 11.3 调整docker服务

为了可以让docker0网桥可以顺利接入预先设定的网段，故需要将flannel生成的配置适用到 docker 中 。编辑文件 /usr/lib/systemd/system/docker.service, 增加 DOCKER_NETWORK_OPTIONS 及  EnvironmentFile=/run/flannel/docker

```sh
EnvironmentFile=/run/flannel/docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock $DOCKER_NETWORK_OPTIONS
```

```sh
systemctl daemon-reload
systemctl start flanneld
```

重启完成后查看三个工作节点 docker 网卡的网段，三个节点都使用了不同的网段

- 172.7.21.1
- 172.7.92.1
- 172.7.66.1

```sh
# node01
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:cb:ce:05:de brd ff:ff:ff:ff:ff:ff
    inet 172.7.21.1/24 brd 172.7.21.255 scope global docker0
       valid_lft forever preferred_lft forever
      
# node02 
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:0d:ad:87:bf brd ff:ff:ff:ff:ff:ff
    inet 172.7.92.1/24 brd 172.7.92.255 scope global docker0
       valid_lft forever preferred_lft forever
       
# node03 
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:6a:8c:d7:61 brd ff:ff:ff:ff:ff:ff
    inet 172.7.66.1/24 brd 172.7.66.255 scope global docker0
       valid_lft forever preferred_lft forever
```

此时使用busybody 测试不同节点间的连通性

```sh
docker run -it --name busybox busybox
```

进入容器后通过 ifconfig 查看不同容器IP ，然后在其中一个容器ping其他容器的IP即可发现可以联通

### 12 工作节点部署 kubectl

````sh
mkdir -p /root/.kube
for i in master01 master02 master03;do scp $i:/root/.kube/config /root/.kube/config;done
ln -s /opt/kubernetes/bin/kubectl /usr/bin/kubectl

# /etc/profile 设置环境变量
export KUBECONFIG=$HOME/.kube/config
````



### 13 工作节点配置 CNI

参考：

- [云计算 K8s 系列 ---- 网络 CNI](https://kingjcy.github.io/post/cloud/paas/base/kubernetes/k8s-network-cni/)

CNI（Container Network Interface）是 CNCF 旗下的一个项目，最早是由 CoreOS 发起的容器网络规范，由一组用于配置 Linux 容器的网络接口的规范和库组成。CNI的工作流程如下：

- 在集群中发起创建Pod请求，apiserver接收到请求到请求会交给scheduler将请求调度到具体节点
- kubelet监听到Pod创建时会为其先创建pause容器以生成ipc, utc, net等命名空间，Pod中的所有容器共用该pause容器
- 等到创建网络时时会先读取配置目录中的配置 xxx.conf，该配置文件指明了使用哪一个网络插件
- 然后执行具体的CNI插件，由该插件进入Pod的pause容器网络空间去配置Pod网络



下载 [cni 插件](https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz)，并把二进制文件放到 /opt/cni/bin 目录

```sh
mkdir /opt/cni/bin -p
tar -xzvf cni-plugins-linux-amd64-v1.0.1.tgz -C /opt/cni/bin
```

接着为集群安装 flannel 配置，配置文件为 kube-flannel.yml  (For Kubernetes v1.17+ ):

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

安装完成即可看到

```sh
   - /etc/kube-flannel/cni-conf.json
    - /etc/cni/net.d/10-flannel.conflist
    
[root@node03 ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                    READY   STATUS     RESTARTS   AGE
kube-system   kube-flannel-ds-4kh4m   0/1     Init:0/1   0          8m27s
kube-system   kube-flannel-ds-kq8rp   0/1     Init:0/1   0          8m27s
kube-system   kube-flannel-ds-zvsmp   0/1     Init:0/1   0          8m27s

# 查看Pod 日志报错，因为没有为用户 etcd-client 开通查询日志权限
[root@node01 ~]# kubectl logs kube-flannel-ds-4kh4m -n kube-system
Error from server (Forbidden): Forbidden (user=etcd-client, verb=get, resource=nodes, subresource=proxy) ( pods/log kube-flannel-ds-4kh4m)

# 为用户开通集群管理员权限，实际情况需要根据需要开通指定的权限
kubectl create clusterrolebinding cluster-admin-for-etcd-client --clusterrole=cluster-admin --user=etcd-client

# 再次查看日志，没有看到有用的日志
[root@node01 ~]# kubectl logs kube-flannel-ds-4kh4m -n kube-system
Error from server (BadRequest): container "kube-flannel" in pod "kube-flannel-ds-4kh4m" is waiting to start: PodInitializing

# 使用 describe 查看 pod，终于看到错误信息, 原来是域名解析出错导致无法拉取 pause
[root@node01 ~]# kubectl describe pod kube-flannel-ds-4kh4m -n kube-system
....Failed to create pod sandbox: rpc error: code = Unknown desc = failed pulling image "harbor.k8s.cc/public/pause:latest"..

# 配置完成域名解析服务后重新创建 Pod
kubectl replace --force -f kube-flannel.yml

```

重建后通过 describe 发现有warning， 






```sh
cat > /etc/kube-flannel/cni-conf.json << "EOF"
{
    "cniVersion": "0.4.0",
    "name": "mybridge",
    "type": "bridge",
    "bridge": "cni_bridge0",
    "isGateway": true,
    "ipMasq": true,
    "hairpinMode":true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.15.20.0/24",
        "routes": [
            { "dst": "0.0.0.0/0" },
            { "dst": "1.1.1.1/32", "gw":"10.15.20.1"}
        ]
    }
}
EOF

cat > /etc/cni/net.d/10-flannel.conflist << "EOF"
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
EOF
```

其中 delegate 的属性如下。 其中，ipam 字段里的信息，比如 10.244.1.0/24，读取自 Flannel 在宿主机上生成的 Flannel 配置文件，即：宿主机上的 /run/flannel/subnet.env 文件：

```json
{
    "hairpinMode":true,
    "ipMasq":false,
    "ipam":{
        "routes":[
            {
                "dst":"10.244.0.0/16"
            }
        ],
        "subnet":"10.244.1.0/24",
        "type":"host-local"
    },
    "isDefaultGateway":true,
    "isGateway":true,
    "mtu":1410,
    "name":"cbr0",
    "type":"bridge"
}
```



















































