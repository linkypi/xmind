### 1. 主机规划

| IP地址            | 机器名称      | 机器配置      | 机器角色   | 安装软件                                                                                                                                                       |
|:---------------:| --------- | --------- | ------ |:---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 192.168.127.110 | master110 | 2C 2G 30G | master | 1. master软件：<br/>kube-apiserver、<br/>kube-controller-manager、<br/>kube-scheduler、<br/>etcd、<br/>haproxy、keepalived<br/><br/>2. worker软件：kubelet、kube-proxy |
| 192.168.127.120 | master120 | 2C 2G 30G | master | 1. master软件：<br/>kube-apiserver、<br/>kube-controller-manager、<br/>kube-scheduler、<br/>etcd、<br/>haproxy、keepalived<br/>2. worker软件：kubelet、kube-proxy      |
| 192.168.127.130 | master130 | 2C 2G 30G | master | 1. master软件：kube-apiserver、kube-controller-manager、kube-scheduler、etcd、haproxy、keepalived<br/><br/>2. worker软件：kubelet、kube-proxy<br/><br/>3. cfssl        |
| 192.168.127.200 | /         | /         | 虚拟IP   | 由 HAProxy 和 keepalived 组成的 LB，由192.168.127.200统一路由到其他三个主节点                                                                                                 |

### 2. 软件版本

| 软件                                                                           | 版本                            |
| ---------------------------------------------------------------------------- | ----------------------------- |
| centos-7.9.2009                                                              | 内核 5.12.9-1.el7.elrepo.x86_64 |
| kube-apiserver、kube-controller-manager、<br>kube-scheduler、kubelet、kube-proxy | v 1.24.3                      |
| etcd                                                                         | v 3.5                         |
| coredns                                                                      |                               |
| haproxy                                                                      | v 1.5.18                      |
| keepalived                                                                   | v1.3.5-6-g6fa32f2             |

### 3. 网络分配

| 网段信息    | 配置            |
| ------- | ------------- |
| Pod     | 172.7.0.0/16  |
| Service | 10.244.0.0/16 |

### 4. 构建基础VM

#### 4.1 配置主机 IP 及源

```shell
# /etc/resolv.conf
nameserver 114.114.114.114

# /etc/sysconfig/network-scripts/ifcfg-ens33
ONBOOT=yes
IPADDR=192.168.127.110
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

```shell
$ uname -a
Linux k8s-master 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

$ cat /etc/redhat-release 
CentOS Linux release 7.3.1611 (Core) 
```

内核版本为 3.10 需要升级到 5

```shell
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

```shell
grub2-set-default "kernel-lt-5.4.151-1"
grub2-mkconfig -o /boot/grub2/grub.cfg
```

重启后即可发现最新内核放到了首位选项中。

#### 4.3 关闭 selinux 及防火墙

编辑文件 /etc/selinux/config ，将 SELINUX 设置为disabled。

再者关闭防火墙：

```shell
systemctl stop firewalld
systemctl disable firewalld
firewall-cmd --set-default-zone=trusted
```

#### 4.4 关闭虚拟内存交换 swap

```sh
swapoff -a ; sed -i '/swap/d' /etc/fstab
```

#### 4.5 系统时间同步

```sh
yum install ntpdate -y

# 制定同步定时任务
crontab -e
0 */1 * * * ntpdate time1.aliyun.com
```

#### 4.6 k8s内核优化

```shell
yum install ipvsadm ipset sysstat conntrack libseccomp -y

cat >> /etc/modules-load.d/ipvs.conf <<EOF
cat 
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
```

```shell
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

# 生效
sysctl --system

#所有节点配置完内核后，重启服务器，保证重启后内核依旧加载
reboot -h now

#重启后查看ipvs模块加载情况：
lsmod | grep --color=auto -e ip_vs -e nf_conntrack

#重启后查看 containerd 模块加载情况：
lsmod | egrep 'br_netfilter | overlay'
```

#### 4.7 创建 k8s 用户

```shell
#创建k8s用户
useradd -m k8s

#创建k8s用户密码 k8s
sh -c 'echo [k8s] |passwd k8s --stdin'

#修改visudo权限
visudo
#去掉%wheel ALL=(ALL) NOPASSWD: ALL这行的注释

#查看是否更改成功
grep '%wheel.*NOPASSWD: ALL' /etc/sudoers

#将k8s用户归到wheel组
gpasswd -a k8s wheel

#查看用户
id k8s

#在每台机器上添加 docker 账户，将 k8s 账户添加到 docker 组中，同时配置 dockerd 参数
useradd -m docker
gpasswd -a k8s docker
```

#### 4.8 安装后台进程管理器 supervisor

```shell
yum install -y epel-release
yum install -y supervisor
systemctl enable supervisord 
systemctl start supervisord 
```

至此，一个基础VM已构建完成，接下来就可以基于该VM直接复制出需要的VM即可。

### 5. 设置主机名

```shell
cat >> /etc/hosts << EOF
192.168.127.1110 master110
192.168.127.20 master120
192.168.127.130 master130
EOF

# 逐个机器设置主机名 
hostnamectl --static set-hostname  master110
```

### 6. 免密登录配置

```shell
cd /root
ssh-keygen -t rsa

for i in master110 master120 master130 ;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done

#测试
ssh -p 4956 master120
```

### 7. 准备根证书

由于证书签发是放在 master130 机器，所以需要进入 master130 机器进行操作

#### 7.1 安装 cfssl

由于cfssl是使用go语言开发，所以需要先安装go再编译源码

```shell
# 首先安装 epel 否则无法找到 go
yum install epel-release -y
yum install go -y
```

下载源码并编译

```shell
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

```shell
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

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

**配置 ca 证书策略**

```shell
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

master集群由三台主机 master110, master120, master130 组成，故需要在这三台机器部署。为了保证三台主机的apiserver高可用，所以需要使用 haproxy + keepalived 做负载均衡。访问方式是通过统一的虚拟IP 192.168.127.200:16443 进行访问，负载到其他三台主机的6443端口，若其中一台apiserver出问题就会自动切换到其他主机来保证高可用

#### 8.1 部署高可用软件 haproxy, keepalived

在三台master节点安装haproxy, keepalived

```shell
yum install keepalived haproxy -y
```

##### 8.1.1 配置 haproxy

```shell
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
 server  master01  192.168.127.110:6443 check
 server  master02  192.168.127.120:6443 check
 server  master03  192.168.127.130:6443 check

listen admin_stats              #web页面的listen
   stats   enable                  #开启统计功能
   bind    *:8888                  #监听的ip端口号
   mode    http
   stats   refresh 30s             #(可选)统计页面自动刷新时间
   stats   uri  /haproxy           #访问的uri   ip:8888/haproxy
   stats   realm haproxy_admin     #(可选)密码框提示文本
   stats   auth admin:admin        #(可选)认证用户名和密码
   stats   hide-version            #(可选)隐藏HAProxy的版本号
   stats   admin if TRUE           #管理功能，需要开启stats auth成功登陆后可通过webui管理节点
EOF
```

##### 8.1.2 配置 keepalived

注意三台主机需要设置不同的优先级 priority ，这样 keepalived 才会自动选择其中一台提供服务

```shell
#master110 配置：
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
   mcast_src_ip 192.168.127.110
   virtual_router_id 50
   priority 100
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass 123456
   }
   virtual_ipaddress {
       192.168.127.200
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master120 配置：
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
   mcast_src_ip 192.168.127.120
   virtual_router_id 51
   priority 99
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass 123456
   }
   virtual_ipaddress {
       192.168.127.200
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master130 配置：
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
   mcast_src_ip 192.168.127.130
   virtual_router_id 52
   priority 88
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass 123456
   }
   virtual_ipaddress {
       192.168.127.200
   }
    track_script {
      chk_apiserver
   }
EOF
```

##### 8.1.3 健康检查脚本

```shell
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

```shell
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
```

#### 8.2 搭建 etcd 集群

##### 8.2.1 签发 etcd 证书

首先准备etcd 证书请求文件 etcd-csr.json, etcd 仅部署在 110， 120 ，130 机器所以只需要三个IP:

```shell
cat > etcd-csr.json <<"EOF"
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.127.110",
    "192.168.127.120",
    "192.168.127.130"
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

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-csr.json | cfssljson -bare etcd-peer

```

##### 8.2.2 部署 etcd 集群

创建 etcd 用户：

```shell
useradd -s /sbin/nologin -M etcd
```

安装etcd :

```shell
cd /opt
get https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
tar -xvf etcd-v3.5.0-linux-amd64.tar.gz
mv etcd-v3.5.0-linux-amd64 etcd-v3.5.0

# 创建相关目录
mkdir -p /opt/etcd/cert /data/etcd /data/etcd/etcd-server /var/logs/etcd

# 复制二进制文件
cp /opt/etcd-v3.5.0/etcd* /opt/etcd

# 进入master130证书目录将 master130 生成的证书复制到 etcd 相关目录
for i in master110 master120 master130 ;do scp /opt/cert/ca*.pem $i:/opt/etcd/cert;done
for i in master110 master120 master130 ;do scp /opt/cert/etcd-peer*.pem $i:/opt/etcd/cert;done
```

首先设置etcd启动脚本：/opt/etcd/etcd-server-startup.sh

> 注意最后的 \ 后面不能有空格
> 
> --name 参数需要与--initial-cluster中的节点名称一致
> 
> --enable-v2 不能少，因为flannel插件使用的是 V2 接口，而etcd从 v3.4 开始默认关闭了 v2 接口

```bash
# master110 
cat > /opt/etcd/etcd-server-startup.sh <<"EOF"
#!/bin/sh
./etcd --name=etcd01 \
       --enable-v2 \
       --data-dir=/data/etcd/etcd-server \
       --listen-peer-urls=https://192.168.127.110:2380 \
       --listen-client-urls=https://192.168.127.110:2379,http://127.0.0.1:2379 \
       --listen-metrics-urls=http://127.0.0.1:2381 \
       --quota-backend-bytes=8000000000 \
       --initial-advertise-peer-urls=https://192.168.127.110:2380 \
       --advertise-client-urls=https://192.168.127.110:2379,http://127.0.0.1:2379 \
       --initial-cluster=etcd01=https://192.168.127.110:2380,etcd02=https://192.168.127.120:2380,etcd03=https://192.168.127.130:2380 \
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

# master120
cat > /opt/etcd/etcd-server-startup.sh <<"EOF"
#!/bin/sh
./etcd --name=etcd02 \
       --enable-v2 \
       --data-dir=/data/etcd/etcd-server \
       --listen-peer-urls=https://192.168.127.120:2380 \
       --listen-client-urls=https://192.168.127.120:2379,http://127.0.0.1:2379 \
       --listen-metrics-urls=http://127.0.0.1:2381 \
       --quota-backend-bytes=8000000000 \
       --initial-advertise-peer-urls=https://192.168.127.120:2380 \
       --advertise-client-urls=https://192.168.127.120:2379,http://127.0.0.1:2379 \
       --initial-cluster=etcd01=https://192.168.127.110:2380,etcd02=https://192.168.127.120:2380,etcd03=https://192.168.127.130:2380 \
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


# master130
cat > /opt/etcd/etcd-server-startup.sh <<"EOF"
#!/bin/sh
./etcd --name=etcd03 \
       --enable-v2 \
       --data-dir=/data/etcd/etcd-server \
       --listen-peer-urls=https://192.168.127.130:2380 \
       --listen-client-urls=https://192.168.127.130:2379,http://127.0.0.1:2379 \
       --listen-metrics-urls=http://127.0.0.1:2381 \
       --quota-backend-bytes=8000000000 \
       --initial-advertise-peer-urls=https://192.168.127.130:2380 \
       --advertise-client-urls=https://192.168.127.130:2379,http://127.0.0.1:2379 \
       --initial-cluster=etcd01=https://192.168.127.110:2380,etcd02=https://192.168.127.120:2380,etcd03=https://192.168.127.130:2380 \
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

```shell
chmod +x /opt/etcd/etcd-server-startup.sh
useradd -s /sbin/nologin -M etcd
chown -R etcd.etcd /opt/etcd-v3.5.0/
chown -R etcd.etcd /data/etcd/
chown -R etcd.etcd /var/logs/etcd/
```

创建启动service：

```shell
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
StandardOutput=append:/var/logs/etcd/out.log
StandardError=append:/var/logs/etcd/error.log
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
stdout_logfile=/var/logs/etcd/stdout.log
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

```shell
./etcdctl --cacert=cert/ca.pem --cert=cert/etcd-peer.pem \
--key=cert/etcd-peer-key.pem  \
--endpoints=https://192.168.127.110:2379,https://192.168.127.120:2379,https://192.168.127.130:2379 endpoint health

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
wget https://dl.k8s.io/v1.24.3/kubernetes-server-linux-amd64.tar.gz
tar xf kubernetes-server-linux-amd64.tar.gz -C /opt
cd /opt/
mv kubernetes kubernetes-v1.24.3
ln -s /opt/kubernetes-v1.24.3/ /opt/kubernetes
cd kubernetes-v1.24.3

# 删除源码包 , 删除无用镜像
rm -fr kubernetes-src.tar.gz
rm -fr server/bin/*.tar 
rm -fr server/bin/*_tag
```

##### 8.3.2 签发 apiserver 请求etcd使用的 client 证书

该证书是为apiserver与etcd集群通讯使用的证书， apiserver作为 client端， etcd作为server端。进入 master130 机器，创建生成证书签名请求的json文件 /opt/cert/etcd-client-csr.json：

```shell
cat > /opt/cert/etcd-client-csr.json << "EOF"
{
    "CN": "etcd-client",
    "hosts":[
       "192.168.127.110",
       "192.168.127.120",
       "192.168.127.130"
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

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
-config=ca-config.json -profile=client \
etcd-client-csr.json | cfssljson -bare etcd-client
```

##### 8.3.3 签发 apiserver 服务端证书

该证书是为其他客户端与apiserver通讯使用的证书， 其他客户端作为 client 端， apiserver 作为 server 端。进入 master130 机器，创建生成证书签名请求的json文件 /opt/cert/apiserver-server-csr.json：

```shell
cat > /opt/cert/apiserver-server-csr.json << "EOF"
{
    "CN": "kubernetes",
    "hosts":[
       "127.0.0.1",
       "kubernetes.default",
       "kubernetes.default.svc",
       "kubernetes.default.svc.cluster",
       "kubernetes.default.svc.cluster.local",
       "192.168.127.110",
       "192.168.127.120",
       "192.168.127.130",
       "192.168.127.200"
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

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
 -config=ca-config.json -profile=server \
apiserver-server-csr.json | cfssljson -bare apiserver-server
```

进入master130 将证书及私钥文件复制到 apiserver机器

```shell
cd /opt/kubernetes/server/bin/
mkdir config
mkdir cert && cd cert
scp master130:/opt/cert/ca*.pem .
scp master130:/opt/cert/etcd-client*.pem .
scp master130:/opt/cert/apiserver-server*.pem .
# 将证书复制到所有主节点
for i in master130 master120 master110; \ 
do scp /opt/cert/apiserver-server*.pem $i:/opt/kubernetes/server/bin/cert;done
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

```shell
echo "`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`,kubelet-bootstrap,10001,\"system:bootstrappers\"" > /opt/kubernetes/server/bin/config/token.csv
```

创建apiserver 启动脚本 /opt/kubernetes/server/bin/kube-apiserver.sh ：

> 注意最后的 \ 后面不能有空格
> 
> 如果同时提供了 `--client-ca-file` 和 `--requestheader-client-ca-file`， 则首先检查 `--requestheader-client-ca-file` CA，然后再检查 `--client-ca-file`。 通常，这些选项中的每一个都使用不同的 CA（根 CA 或中间 CA）。 常规客户端请求与 `--client-ca-file` 相匹配，而聚合请求要与 `--requestheader-client-ca-file` 相匹配。 但是，如果两者都使用同一个 CA，则通常会通过 `--client-ca-file` 传递的客户端请求将失败，因为 CA 将与 `--requestheader-client-ca-file` 中的 CA 匹配，但是通用名称 `CN=` 将不匹配 `--requestheader-allowed-names` 中可接受的通用名称之一。 这可能导致你的 kubelet 和其他控制平面组件以及最终用户无法向 Kubernetes apiserver 认证。

- `--requestheader-client-ca-file`：用于签名 `--proxy-client-cert-file` 和 `--proxy-client-key-file` 指定的证书；在启用了 metric aggregator 时使用；暂时去除
- requestheader-allowed-names
- kubelet-client 相关证书可与etcd-certfile共用同一套证书，该证书都是以apiserver作为客户端分布请求 kubelet 与 etcd 所使用的证书

```shell
mkdir /var/logs/kubernetes/kube-apiserver


cat > /opt/kubernetes/server/bin/kube-apiserver.sh << "EOF"
#!/bin/bash
./kube-apiserver \
 --apiserver-count=3 \
 --advertise-address=192.168.127.110 \
 --enable-aggregator-routing=true \
 --audit-log-path=/var/logs/kubernetes/kube-apiserver/audit.log \
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
 --etcd-servers=https://192.168.127.110:2379,https://192.168.127.120:2379,https://192.168.127.130:2379 \
 --service-account-issuer=k8s.cc \
 --service-account-key-file=./cert/ca-key.pem \
 --service-account-signing-key-file=./cert/ca-key.pem \
 --service-cluster-ip-range=192.168.0.0/16 \
 --service-node-port-range=3000-49999 \
 --kubelet-client-certificate=./cert/etcd-client.pem \
 --kubelet-client-key=./cert/etcd-client-key.pem \
 --log-dir=/var/logs/kubernetes/kube-apiserver \
 --tls-cert-file=./cert/apiserver-server.pem \
 --tls-private-key-file=./cert/apiserver-server-key.pem \
 --anonymous-auth=false \
 --allow-privileged=true \
 --v=2
 EOF
```

```
chmod +x kube-apiserver.sh 
mkdir -p /data/logs/kubernetes/kube-apiserver
```

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

# 查看集群状态
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
