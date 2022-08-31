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

| 网段信息    | 配置               |
| ------- | ---------------- |
| Node    | 192.168.127.0/16 |
| Pod     | 172.7.0.0/16     |
| Service | 10.244.0.0/16    |

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
systemctl disable --now firewalld 
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager
setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
```

#### 4.4 关闭虚拟内存交换 swap

```shell
swapoff -a ; sed -i '/swap/d' /etc/fstab
```

#### 4.5 系统时间同步

```shell
yum install ntpdate -y

# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 制定同步定时任务
crontab -e
0 */1 * * * ntpdate time1.aliyun.com
```

#### 4.6 k8s内核优化

```shell
yum install ipvsadm ipset sysstat conntrack libseccomp -y

cat >/etc/modules-load.d/ipvs.conf <<EOF 
ip_vs 
ip_vs_lc 
ip_vs_wlc 
ip_vs_rr 
ip_vs_wrr 
ip_vs_lblc 
ip_vs_lblcr 
ip_vs_dh 
ip_vs_sh 
ip_vs_fo 
ip_vs_nq 
ip_vs_sed 
ip_vs_ftp 
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

systemctl restart systemd-modules-load.service

lsmod | grep -e ip_vs -e nf_conntrack
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

# 最好将 supervisor 配置文件中的进程异常自动重试设置为false
# 否则使用 supervisorctl 停止进程后会自动重启（重试三次），及其蛋疼
# 更重要的是重启完成后没有更新进程状态，通过status还是历史状态
vim /etc/supervisord.conf

autorestart=false              ; retstart at unexpected quit (default: true)
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

### 8. Master节点部署

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
 server  master110  192.168.127.110:6443 check
 server  master120  192.168.127.120:6443 check
 server  master130  192.168.127.130:6443 check

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
[program:etcd-server]
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
--endpoints=https://192.168.127.110:2379,https://192.168.127.120:2379,https://192.168.127.130:2379 \
endpoint health

[root@master01 etcd]# ./etcdctl --cacert=cert/ca.pem \
--cert=cert/etcd-peer.pem --key=cert/etcd-peer-key.pem \
 --endpoints=https://192.168.127.110:2379,https://192.168.127.120:2379,https://192.168.127.130:2379 \
endpoint health
https://192.168.127.10:2379 is healthy: successfully committed proposal: took = 15.296012ms
https://192.168.127.30:2379 is healthy: successfully committed proposal: took = 99.475887ms
https://192.168.127.20:2379 is healthy: successfully committed proposal: took = 169.215014ms

[root@master01 etcd]# ./etcdctl --cacert=cert/ca.pem \
--cert=cert/etcd-peer.pem --key=cert/etcd-peer-key.pem \
 --endpoints=https://192.168.127.110:2379,https://192.168.127.120:2379,https://192.168.127.130:2379 \
endpoint  status --write-out=table
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.127.10:2379 | 1c7bb24a23bc3671 |   3.5.0 |   20 kB |      true |      false |      1679 |       3372 |               3372 |        |
| https://192.168.127.20:2379 | f37fd8ae634a8102 |   3.5.0 |   20 kB |     false |      false |      1679 |       3372 |               3372 |        |
| https://192.168.127.30:2379 | f049b8836150b9f8 |   3.5.0 |   20 kB |     false |      false |      1679 |       3372 |               3372 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

##### 8.2.4 数据备份与恢复

由于etcd集群可能因为不可抗原因导致不可用的情况，故为了能把etcd集群快速恢复到正常状态就需要对其数据进行备份，以便快速恢复。

备份脚本 etcd_tool.sh：

```shell
# /bin/bash
CERT_PATH=/opt/etcd/cert
CACERT=$CERT_PATH/ca.pem
CERT=$CERT_PATH/etcd-peer.pem
KEY=$CERT_PATH/etcd-peer-key.pem
# 此处只能指定一个节点
ENDPOINTS=https://192.168.127.110:2379
ETCDCTL_API=3
BACKUP_FILE=/data/etcd/backup/etcd-snapshot-`date +%Y%m%d%H%M%S`.db
DATA_DIR=/data/etcd/etcd-server

if [[ $1 = 'backup' ]];
then
  mkdir -p /data/etcd/backup
  etcdctl --cacert="${CACERT}" \
  --cert="${CERT}" --key="${KEY}" \
  --endpoints=${ENDPOINTS} \
  snapshot save ${BACKUP_FILE}
elif [[ $1 = 'restore' ]];
then
  name='etcd0'$2
  # 此处只能使用 IP 的形式，因为证书已经指定IP，
  # 若此处需要指定域名则证书同样需要，否则会认证失败
  hostname='192.168.127.1'$2'0'
  host1=192.168.127.110
  host2=192.168.127.120
  host3=192.168.127.130

  etcdutl snapshot restore $3 \
  --name ${name} \
  --data-dir=${DATA_DIR} \
  --initial-cluster etcd01=https://${host1}:2380,etcd02=https://${host2}:2380,etcd03=https://${host3}:2380 \
  --initial-cluster-token ea8cfe2bfe85b7e6c66fe190f9225838 \
  --initial-advertise-peer-urls https://${hostname}:2380
else
  echo "command $1 not found"
fi
```

整个执行顺序：

1. 在任一节点备份 etcd（正常应该由 crontab来定时备份）

2. 停止 kube-apiserver

3. 停止 etcd

4. 恢复数据

5. 启动 etcd

6. 启动 kube-apiserver

```shell
# 恢复数据前先删除目录下的数据
rm -fr /data/etcd/etcd-server/*
# 在master110 将数据恢复到数据目录
./etcd_tool.sh restore 1 /data/etcd/backup/etcd-snapshot-20220830204356.db
# 在master120 将数据恢复到数据目录
./etcd_tool.sh restore 2 /data/etcd/backup/etcd-snapshot-20220830204356.db
# 在master130 将数据恢复到数据目录
./etcd_tool.sh restore 3 /data/etcd/backup/etcd-snapshot-20220830204356.db

# 在三台机器分别执行
chown etcd:etcd -R /data/etcd/etcd-server/
supervisorctl start etcd-server
```

etcd集群重启完成即可查看k8s集群pod情况

```shell
[root@master110 etcd]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS             RESTARTS         AGE
kube-system   calico-kube-controllers-5b97f5d8cf-tbmmt   0/1     CrashLoopBackOff   26 (3m39s ago)   23h
kube-system   calico-node-7wkx2                          1/1     Running            0                23h
kube-system   calico-node-dlv6q                          1/1     Running            0                23h
kube-system   calico-node-zt84m                          1/1     Running            0                23h
kube-system   coredns-754f9b4f7c-4j6j7                   1/1     Running            0                23h
kube-system   metrics-server-6d79d5d786-jg2jh            1/1     Running            0                23h
```

经过几分钟的等待后发现CrashLoopBackO的Pod也已正常Running。不过也有可能存在部分无法启动的问题，此时需要找到原有yaml资源文件重启即可。如此便能省去重新部署各个pod繁琐操作，达到快速恢复的目的。

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

```context
如果 hosts 字段不为空则需要指定授权使用该证书的 IP（含VIP） 或域名列表。
由于该证书被 集群使用，需要将节点的IP都填上，为了方便后期扩容可以多写几个预留的IP。
同时还需要填写 service 网络的首个IP(一般是 kube-apiserver 指定的 
service-cluster-ip-range 网段的第一个IP，如 10.244.0.1)。
```

```shell
cat > /opt/cert/apiserver-server-csr.json << "EOF"
{
    "CN": "kubernetes",
    "hosts":[
       "127.0.0.1",
       "kubernetes",
       "kubernetes.default",
       "kubernetes.default.svc",
       "kubernetes.default.svc.cluster",
       "kubernetes.default.svc.cluster.local",
       "10.244.0.1",
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
cd /opt/cert/
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
do scp /opt/cert/apiserver-server*.pem \
$i:/opt/kubernetes/server/bin/cert;done
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
# 在 master130 执行以下命令 生成 token.csv
echo "`head -c 16 /dev/urandom | od -An -t x | \
tr -d ' '`,kubelet-bootstrap,10001,\"system:bootstrappers\"" >\
/opt/kubernetes/server/bin/config/token.csv

# 将token.csv复制到其他主机
for i in master120 master110 master130; \
do scp /opt/kubernetes/server/bin/config/token.csv \
$i:/opt/kubernetes/server/bin/config;done
```

创建apiserver 启动脚本 /opt/kubernetes/server/bin/kube-apiserver.sh ：

> 注意最后的 \ 后面不能有空格
> 
> 如果同时提供了 `--client-ca-file` 和 `--requestheader-client-ca-file`， 则首先检查 `--requestheader-client-ca-file` CA，然后再检查 `--client-ca-file`。 通常，这些选项中的每一个都使用不同的 CA（根 CA 或中间 CA）。 常规客户端请求与 `--client-ca-file` 相匹配，而聚合请求要与 `--requestheader-client-ca-file` 相匹配。 但是，如果两者都使用同一个 CA，则通常会通过 `--client-ca-file` 传递的客户端请求将失败，因为 CA 将与 `--requestheader-client-ca-file` 中的 CA 匹配，但是通用名称 `CN=` 将不匹配 `--requestheader-allowed-names` 中可接受的通用名称之一。 这可能导致你的 kubelet 和其他控制平面组件以及最终用户无法向 Kubernetes apiserver 认证。

- `--requestheader-client-ca-file`：用于签名 `--proxy-client-cert-file` 和 `--proxy-client-key-file` 指定的证书；在启用了 metric aggregator 时使用；暂时去除
- requestheader-allowed-names
- kubelet-client 相关证书可与etcd-certfile共用同一套证书，该证书都是以apiserver作为客户端分布请求 kubelet 与 etcd 所使用的证书

```shell
# master110
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
 --service-cluster-ip-range=10.244.0.0/16 \
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



# master120
cat > /opt/kubernetes/server/bin/kube-apiserver.sh << "EOF"
#!/bin/bash
./kube-apiserver \
 --apiserver-count=3 \
 --advertise-address=192.168.127.120 \
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
 --service-cluster-ip-range=10.244.0.0/16 \
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



# master130
cat > /opt/kubernetes/server/bin/kube-apiserver.sh << "EOF"
#!/bin/bash
./kube-apiserver \
 --apiserver-count=3 \
 --advertise-address=192.168.127.130 \
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
 --service-cluster-ip-range=10.244.0.0/16 \
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
mkdir -p /var/logs/kubernetes/kube-apiserver
```

创建apiserver对应 的supervisor启动文件 /usr/lib/systemd/system/kube-apiserver.service :

```shell
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
StandardOutput=/var/logs/kubernetes/kube-apiserver/apiserver.log
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
[program:kube-apiserver]
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
stdout_logfile=/var/logs/kubernetes/kube-apiserver/apiserver.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF

# 创建完成后执行命令启动
supervisorctl update
```

##### 8.3.5 使用 kubectl 查看集群状态

为了使得用户具备管理权限，同样需要先签发证书，<mark>注意证书请求文件中的 names -> O 属性必须是"system:masters"</mark>

```shell
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

```
后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 
将 Group system:masters 与 Role cluster-admin 绑定，
该 Role 授予了调用kube-apiserver 的所有 API的权限；
O指定该证书的 Group 为 system:masters，kubelet 使用
该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，
同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
注：
这个admin 证书，是将来生成管理员用的kubeconfig 配置文件用的，
现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制，
kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group；
"O": "system:masters", 必须是system:masters，
否则后面kubectl create clusterrolebinding报错。
```

```shell
cd /opt/cert
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
-config=ca-config.json -profile=peer \
kubectl-admin-csr.json | cfssljson -bare kubectl-admin

# 将证书复制到所有主节点
for i in master110 master120 master130;\
do scp /opt/cert/kubectl-admin*.pem \
$i:/opt/kubernetes/server/bin/cert;done
```

```powershell
ln -s /opt/kubernetes/server/bin/kubectl \
/usr/bin/kubectl

# 设置集群参数 注意此处使用的 server IP 是 192.168.127.200，高可用
kubectl config set-cluster kubernetes \
 --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
 --embed-certs=true \
 --server=https://192.168.127.200:16443 \
 --kubeconfig=/opt/kubernetes/server/bin/config/admin.config

kubectl config set-credentials admin \
 --client-certificate=/opt/kubernetes/server/bin/cert/kubectl-admin.pem \
 --client-key=/opt/kubernetes/server/bin/cert/kubectl-admin-key.pem \
 --embed-certs=true \
 --kubeconfig=/opt/kubernetes/server/bin/config/admin.config

kubectl config set-context kubernetes --cluster=kubernetes \
--user=admin --kubeconfig=/opt/kubernetes/server/bin/config/admin.config

kubectl config use-context kubernetes \
--kubeconfig=/opt/kubernetes/server/bin/config/admin.config

mkdir ~/.kube
cp /opt/kubernetes/server/bin/config/admin.config \
~/.kube/config

#绑定集群管理员角色
kubectl create clusterrolebinding kube-apiserver:kubelet-apis \
--clusterrole=system:kubelet-api-admin --user admin \
--kubeconfig=/root/.kube/config

export KUBECONFIG=$HOME/.kube/config

# 查看集群状态
kubectl cluster-info
kubectl get componentstatuses
kubectl get all --all-namespaces
```

```
[root@localhost bin]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.127.200:16443

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

进入master130 机器 /opt/cert 目录生成相关证书

```
说明：hosts 列表包含所有 kube-controller-manager 节点 IP；
CN 为 system:kube-controller-manager;
O 为 system:kube-controller-manager，kubernetes 内置的 
ClusterRoleBindings system:kube-controller-manager 
赋予 kube-controller-manager 工作所需的权限
```

```shell
cat > /opt/cert/kube-controller-manager-csr.json << "EOF"
{
    "CN": "system:kube-controller-manager",
    "hosts":[
       "127.0.0.1",
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
           "O": "system:kube-controller-manager",
           "OU": "system"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
 -config=ca-config.json -profile=peer \
kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

# 将证书复制到所有master主节点
for i in master110 master120 master130;\
do scp /opt/cert/kube-controller-manager*.pem $i:/opt/kubernetes/server/bin/cert;done
```

> CN 为 system:kube-controller-manager , O 为 system:kube-controller-manager
> 
> kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限

##### 8.4.2 生成配置文件

进入目录 /opt/kubernetes/server/bin 生成配置文件

```shell
cd /opt/kubernetes/server/bin

kubectl config set-cluster kubernetes \
--certificate-authority=cert/ca.pem \
--embed-certs=true \
--server=https://192.168.127.200:16443 \
--kubeconfig=config/kube-controller-manager.config

kubectl config set-credentials \
system:kube-controller-manager \
--client-certificate=cert/kube-controller-manager.pem \
--client-key=cert/kube-controller-manager-key.pem \
--embed-certs=true --kubeconfig=config/kube-controller-manager.config

kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes --user=system:kube-controller-manager \
--kubeconfig=config/kube-controller-manager.config

kubectl config use-context system:kube-controller-manager \
--kubeconfig=config/kube-controller-manager.config
```

##### 8.4.3 创建启动脚本

> 注意：参数 --cluster-signing-xxx 用于为 kubelet 颁发证书时使用，与apiserver 保持一致

在三台机器上分别创建启动脚本

```shell
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
  --log-dir=/var/logs/kubernetes/kube-controller-manager \
  --service-account-private-key-file=./cert/ca-key.pem \
  --service-cluster-ip-range=10.244.0.0/16 \
  --root-ca-file=./cert/ca.pem \
  --client-ca-file=./cert/ca.pem \
  --requestheader-client-ca-file=./cert/ca.pem \
  --tls-cert-file=./cert/kube-controller-manager.pem \
  --tls-private-key-file=./cert/kube-controller-manager-key.pem \
  --cluster-signing-cert-file=./cert/ca.pem \
  --cluster-signing-key-file=./cert/ca-key.pem \
  --v=2
EOF

mkdir -p /var/logs/kubernetes/kube-controller-manager
chmod +x /opt/kubernetes/server/bin/kube-controller-manager.sh
```

设置supervisor服务，/etc/supervisord.d/kube-controller-manager.ini

```shell
cat > /etc/supervisord.d/kube-controller-manager.ini << "EOF"
[program:kube-controller-manager]
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
stdout_logfile=/var/logs/kubernetes/kube-controller-manager/controller.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF
```

启动 kube-controller-manager 并查看状态

```shell
supervisorctl update
supervisorctl status

# 查看controller集群状态
kubectl get cs
```

启动完成后继续在其他两台机器进行部署。

#### 8.5 部署 kube-schesuler

##### 8.5.1 生成证书

进入master130机器目录 /opt/cert

```shell
cd /opt/cert
cat > /opt/cert/kube-scheduler-csr.json << "EOF"
{
    "CN": "system:kube-scheduler",
    "hosts":[
       "127.0.0.1",
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
           "O": "system:kube-scheduler",
           "OU": "system"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
-config=ca-config.json -profile=peer \
kube-scheduler-csr.json | cfssljson -bare kube-scheduler

# 将证书复制到所有主节点
for i in master110 master120 master130;\
do scp /opt/cert/kube-scheduler*.pem $i:/opt/kubernetes/server/bin/cert;done
```

##### 8.5.2 创建配置文件

在三台机器对应bin目录生成 kube-scheduler.config 配置文件

```powershell
cd /opt/kubernetes/server/bin

kubectl config set-cluster kubernetes \
--certificate-authority=cert/ca.pem \
--embed-certs=true --server=https://192.168.127.200:16443 \
--kubeconfig=config/kube-scheduler.config

kubectl config set-credentials system:kube-scheduler \
--client-certificate=cert/kube-scheduler.pem \
--client-key=cert/kube-scheduler-key.pem \
--embed-certs=true --kubeconfig=config/kube-scheduler.config

kubectl config set-context system:kube-scheduler \
--cluster=kubernetes --user=system:kube-scheduler \
--kubeconfig=config/kube-scheduler.config

kubectl config use-context system:kube-scheduler \
--kubeconfig=config/kube-scheduler.config
```

##### 8.5.3 创建启动脚本

```shell
cat > /opt/kubernetes/server/bin/kube-scheduler.sh << "EOF"
#!/bin/sh
./kube-scheduler \
  --kubeconfig=/opt/kubernetes/server/bin/config/kube-scheduler.config
  --leader-elect=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/logs/kubernetes/kube-scheduler \
  --v=2
EOF

mkdir -p /var/logs/kubernetes/kube-scheduler
chmod +x /opt/kubernetes/server/bin/kube-scheduler.sh
```

设置supervisor服务，/etc/supervisord.d/kube-scheduler.ini

```shell
cat > /etc/supervisord.d/kube-scheduler.ini << "EOF"
[program:kube-scheduler]
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
stdout_logfile=/var/logs/kubernetes/kube-scheduler/scheduler.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF
```

启动 kube-scheduler 并查看状态

```shell
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

至此，由三台master主机组成的高可用集群已经部署完成，外部统一使用虚拟IP 192.168.127.200 进行访问。当前三台主机所安装的软件都是作为master主机应当安装的

| IP地址            | 机器名称      | 机器配置      | 机器角色   | 安装软件                                                       |
| --------------- | --------- | --------- | ------ | ---------------------------------------------------------- |
| 192.168.127.110 | master110 | 2C 2G 30G | master | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 192.168.127.120 | master120 | 2C 2G 30G | master | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 192.168.127.130 | master130 | 2C 2G 30G | master | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 192.168.127.200 | /         | /         | 虚拟IP   | 由 HAProxy 和 keepalived 组成的 LB，由192.168.127.200统一路由到其他三个主节点 |

### 9. Worker工作节点部署

由于机器限制，所以三台master节点也同时承担着工作节点的任务

#### 9.1 安装 containerd

##### 9.1.1 安装软件及配置

```shell
wget https://github.com/containerd/containerd/releases/download/v1.6.8/cri-containerd-cni-1.6.8-linux-amd64.tar.gz
cd /opt
# 解压后直接完成安装
tar -xf cri-containerd-cni-1.6.8-linux-amd64.tar.gz -C /

mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
```

修改 config.toml

```powershell
# 1. 修改 SystemdCgroup 配置，改为启用
# 2. 修改pause镜像远程仓库地址
sed -i 's@SystemdCgroup = false@SystemdCgroup = true@' /etc/containerd/config.toml
sed -i 's@k8s.gcr.io/pause:3.6@registry.aliyuncs.com/google_containers/pause:3.6@' /etc/containerd/config.toml

# 修改镜像仓库地址，增加 daocloud 及 docker.io 仓库地址
 [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
         [plugins."io.containerd.grpc.v1.cri".registry.mirrors."daocloud.io"]
              endpoint = ["http://f1361db2.m.daocloud.io"]
         [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
              endpoint = ["http://hub-mirror.c.163.com"]
```

##### 9.1.2 安装runc依赖

虽然containerd 软件包已经包含 runc ，但在命令行运行时会提示错误，因为还未安装runc的相关依赖:

```shell
[root@master130 opt]# runc
runc: symbol lookup error: runc: undefined symbol: seccomp_notify_respond
```

进入到 opt 目录下载 runc 并安装

```shell
cd /opt
wget https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64

chmod +x runc.amd64

#替换掉原软件包中的runc
mv runc.amd64 /usr/local/sbin/runc

[root@master130 opt]# runc -v
runc version 1.1.3
commit: v1.1.3-0-g6724737f
spec: 1.0.2-dev
go: go1.17.10
libseccomp: 2.5.4
```

配置完成后启动containerd:

```shell
 systemctl enable containerd
 systemctl start containerd
```

##### 9.1.3 查看容器

```shell
# ctr 命令必须带上命名空间
ctr -n k8s.io containers ls 

# 无需带上命名空间
crictl ps
```

#### 9.2 部署 kubelet

为了让 kubelet 可以访问到master集群中的apiserver服务，kubelet需要知道连接参数及证书。所以首先需要生成证书文件。在生成证书前需要认识下 Kubernetes TLS bootstrapping。当集群开启了 TLS 认证后，每个节点的 kubelet 组件都要使用由 apiserver 使用的 CA 签发的有效证书才能与 apiserver 通讯；此时如果节点多起来，为每个节点单独签署证书将是一件非常繁琐的事情；TLS bootstrapping 功能就是让 kubelet 先使用一个预定的低权限用户连接到 apiserver，然后向 apiserver 申请证书，kubelet 的证书由 apiserver 动态签署；

##### 9.2.1 签发 kubelet证书

首先进入 master130 机器 /opt/cert 创建证书请求文件 kubelet-csr.json

```shell
cd /opt/cert
cat > kubelet-csr.json << "EOF"
{
    "CN": "system:node-bootstrapper",
    "hosts":[
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
           "ST": "GuangDong",
           "L": "ShenZhen",
           "O": "system:masters",   
           "OU": "ops"
        }
    ]
}
EOF
```

基于已有根证书开始签发 kubelet 证书

```powershell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
-config=ca-config.json -profile=server \
kubelet-csr.json | cfssljson -bare kubelet
```

生成证书后将证书拷贝到三个机器的目录 /opt/kubernetes/server/bin/cert 中

```shell
for i in master110 master120 master130;\
do scp /opt/cert/kubelet*.pem $i:/opt/kubernetes/server/bin/cert;done

for i in master110 master120 master130;\
do scp /opt/cert/ca*.pem $i:/opt/kubernetes/server/bin/cert;done
```

##### 9.2.2 生成kubelet配置文件及脚本

进入master集群中任一节点生成 kubelet-bootstrap.config， kubelet.json文件， 注意 BOOTSTRAP_TOKEN 参数不能少，连接apiserver时会出现证书异常的问题

```powershell
cd /opt/kubernetes/server/bin

BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /opt/kubernetes/server/bin/config/token.csv)

kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
--embed-certs=true --server=https://192.168.127.200:16443 \
--kubeconfig=config/kubelet-bootstrap.config

kubectl config set-credentials kubelet-bootstrap \
--token=${BOOTSTRAP_TOKEN} \
--client-certificate=/opt/kubernetes/server/bin/cert/kubelet.pem \
--client-key=/opt/kubernetes/server/bin/cert/kubelet-key.pem \
--embed-certs=true \
--kubeconfig=config/kubelet-bootstrap.config

kubectl config set-context default --cluster=kubernetes \
--user=kubelet-bootstrap --kubeconfig=config/kubelet-bootstrap.config

kubectl config use-context default \
--kubeconfig=config/kubelet-bootstrap.config
```

config

```powershell
kubectl create clusterrolebinding cluster-system-anonymous \
--clusterrole=cluster-admin --user=kubelet-bootstrap

kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper --user=kubelet-bootstrap \
--kubeconfig=config/kubelet-bootstrap.config
```

```powershell
kubectl describe clusterrolebinding cluster-system-anonymous

kubectl describe clusterrolebinding kubelet-bootstrap
```

```shell
# 创建配置文件
cat > config/kubelet.json << "EOF"
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/opt/kubernetes/server/bin/cert/ca.pem"
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
  "address": "192.168.127.110",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",                    
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.244.0.2"]
}
EOF
```

创建启动脚本：

> 注意： 配置文件 config/kubelet.kubeconfig 会自动生成, 用于连接 apiserver

```shell
cat > /opt/kubernetes/server/bin/kubelet.sh << "EOF"
#!/bin/sh
./kubelet \
  --bootstrap-kubeconfig=config/kubelet-bootstrap.config \
  --kubeconfig=config/kubelet.kubeconfig \
  --hostname-override=node01 \
  --config=config/kubelet.json \
  --cert-dir=/opt/kubernetes/server/bin/cert \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.2 \
  --rotate-certificates \
  --root-dir=/etc/cni/net.d \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/logs/kubernetes/kubelet \
  --v=2
EOF

mkdir -p /var/logs/kubernetes/kubelet
chmod +x /opt/kubernetes/server/bin/kubelet.sh
```

设置supervisor服务，/etc/supervisord.d/kubelet.ini

```shell
cat > /etc/supervisord.d/kubelet.ini << "EOF"
[program:kubelet]
command=/opt/kubernetes/server/bin/kubelet.sh
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
stdout_logfile=/var/logs/kubernetes/kubelet/kubelet.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF
```

```shell
# 在 master节点为用户授权请求签发证书
cat >> config/kubelet-bootstrap-rbac.yaml << EOF
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

##### 9.2.3 审批加入工作节点

审批进入master任一节点即可查看到工作节点的证书请求：

```shell
[root@mater01 bin]# kubectl get csr
NAME                    AGE     SIGNERNAME                   REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-BbHAG2ba...   9m52s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
node-csr-ttkk_kWj...   8m50s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
node-csr-ywB_XBQN...   6m48s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

```shell
# 单个审批
kubectl certificate approve node-csr-BbHAG2ba...

# 批量审批
kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve
```

#### 9.3 部署 kube-proxy

##### 9.3.1 申请证书

```shell
cd /opt/cert

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

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
-config=ca-config.json -profile=peer \
kube-proxy-csr.json | cfssljson -bare kube-proxy

for i in master110 master120 master130;\
do scp /opt/cert/kube-proxy*.pem $i:/opt/kubernetes/server/bin/cert;done

for i in master110 master120 master130; \
do scp /opt/cert/kube-proxy*.pem $i:/opt/kubernetes/server/bin/cert;done
```

##### 9.3.2 生成配置文件

进入任一master节点生成 kube-proxy.config 配置文件 并复制到工作节点

```powershell
cd /opt/kubernetes/server/bin

kubectl config set-cluster kubernetes \
--certificate-authority=cert/ca.pem \
--embed-certs=true --server=https://192.168.127.200:16443 \
--kubeconfig=config/kube-proxy.config

kubectl config set-credentials kube-proxy \
--client-certificate=cert/kube-proxy.pem \
--client-key=cert/kube-proxy-key.pem \
--embed-certs=true --kubeconfig=config/kube-proxy.config

kubectl config set-context default --cluster=kubernetes \
--user=kube-proxy --kubeconfig=config/kube-proxy.config

kubectl config use-context default \
--kubeconfig=config/kube-proxy.config

for i in master110 master120 master130;\
do scp config/kube-proxy.config $i:/opt/kubernetes/server/bin/config;done
```

生成 kube-proxy.yml 配置, 注意 所有的 *bindAddress 都是工作节点 IP

```shell
cat > config/kube-proxy.yml << "EOF"
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /opt/kubernetes/server/bin/config/kube-proxy.config
clusterCIDR: 172.7.0.0/16
hostnameOverride: node01
healthzBindAddress: 0.0.0.0:10256
metricsBindAddress: 0.0.0.0:10249
mode: "ipvs"
EOF

for i in master110 master120 master130;\
do scp config/kube-proxy.yml $i:/opt/kubernetes/server/bin/config;done
```

##### 9.3.3 创建脚本

```shell
cat > /opt/kubernetes/server/bin/kube-proxy.sh << "EOF"
#!/bin/sh
./kube-proxy \
  --config=config/kube-proxy.yml \
  --log-dir=/data/logs/kubernetes/kube-proxy \
  --v=2
EOF

mkdir -p /var/logs/kubernetes/kube-proxy
chmod +x /opt/kubernetes/server/bin/kube-proxy.sh

for i in master110 master120 master130;\
do scp kube-proxy.sh $i:/opt/kubernetes/bin;done
```

设置supervisor服务，/etc/supervisord.d/kube-proxy.ini

```shell
cat > /etc/supervisord.d/kube-proxy.ini << "EOF"
[program:kube-proxy]
command=/opt/kubernetes/server/bin/kube-proxy.sh
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
stdout_logfile=/var/logs/kubernetes/kube-proxy/proxy.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
EOF

for i in master110 master120 master130;\
do scp /etc/supervisord.d/kube-proxy.ini $i:/etc/supervisord.d/;done
```

```sh
# 启动 kube-proxy
supervisorctl update
```

### 10 安装网络插件 Calico

#### 10.1 下载安装文件

在任意一个master节点执行以下命令，参考[Install Calico networking and network policy for on-premises deployments](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less)

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.0/manifests/calico.yaml -O
```

下载完成后, 首先修改配置文件。在文件中搜索并找到 CALICO_IPV4POOL_CIDR 的注释行，将前后两行注释打开，并将对应的value修改为此前规划好的Pod网段：

```yaml
3683    - name: CALICO_IPV4POOL_CIDR
3684      value: "172.7.0.0/16"
```

启动：

```shell
kubectl apply -f  calico.yaml
```

启动后遇到错误：

```shellsession
The DaemonSet "calico-node" is invalid: 
* spec.template.spec.containers[0].securityContext.privileged: Forbidden: disallowed by cluster policy
* spec.template.spec.initContainers[0].securityContext.privileged: Forbidden: disallowed by cluster policy
* spec.template.spec.initContainers[1].securityContext.privileged: Forbidden: disallowed by cluster policy
* spec.template.spec.initContainers[2].securityContext.privileged: Forbidden: disallowed by cluster policy
```

这是因为apiserver启动时没有增加参数 --allow-privileged=true。在kubet-apiserver脚本中配置该参数后重新启动即可。Calico启动完成后可以通过命令查看状态：

安装calico遇到镜像下载失败的问题，可以先在任意一台机器手动拉取镜像，再将所有用到的镜像导出，然后复制到其他机器：

```shell
# 镜像拉取失败
[root@master120 opt]# kubectl get pods -n kube-system
NAME                                       READY   STATUS                  RESTARTS      AGE
calico-kube-controllers-5b97f5d8cf-knw8r   0/1     ErrImagePull            0             17s
calico-node-9l7tf                          0/1     Init:CrashLoopBackOff   1 (12s ago)   18s
calico-node-xhsjz                          0/1     Init:Error              1 (15s ago)   18s
calico-node-xskjg                          0/1     Init:0/3                0             18s

# 查看calico所用到的镜像
[root@master120 opt]# cat calico.yaml|grep -E "image:"
          image: docker.io/calico/cni:v3.24.0
          image: docker.io/calico/cni:v3.24.0
          image: docker.io/calico/node:v3.24.0
          image: docker.io/calico/node:v3.24.0
          image: docker.io/calico/kube-controllers:v3.24.0

#手动拉取镜像
crictl pull docker.io/calico/cni:v3.24.0
crictl pull docker.io/calico/node:v3.24.0
crictl pull docker.io/calico/kube-controllers:v3.24.0

#打包镜像
ctr -n k8s.io image export k8s.1.24.3.tar\
docker.io/calico/cni:v3.24.0 \
docker.io/calico/node:v3.24.0 \
docker.io/calico/kube-controllers:v3.24.0

#复制到其他机器
for i in master120 master130;\
do scp k8s.1.24.3.tar $i:/opt;done

#到指定机器导入镜像
ctr -n k8s.io image import /opt/k8s.1.24.3.tar
```

#### 10.2 遇到的问题

<mark>最终发现时间原因是因为 config.toml 配置文件使用了旧版. 直接使用containerd生成默认配置文件后简单修改即可， 参考 9.1.1。故该解决方案可忽略</mark>

```shell
[root@master120 opt]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS                  RESTARTS      AGE
kube-system   calico-kube-controllers-5b97f5d8cf-ml55w   0/1     CrashLoopBackOff        2 (22s ago)   46s
kube-system   calico-node-72sfl                          0/1     Init:CrashLoopBackOff   2 (26s ago)   46s
kube-system   calico-node-kbll7                          0/1     Init:CrashLoopBackOff   2 (27s ago)   46s
kube-system   calico-node-q8c7b                          0/1     Init:CrashLoopBackOff   2 (22s ago)   46s

[root@master120 opt]# kubectl logs -f -n kube-system calico-kube-controllers-5b97f5d8cf-ml55w
2022-08-20 02:37:06.454 [INFO][1] main.go 103: Loaded configuration from environment config=&config.Config{LogLevel:"info", WorkloadEndpointWorkers:1, ProfileWorkers:1, PolicyWorkers:1, NodeWorkers:1, Kubeconfig:"", DatastoreType:"kubernetes"}
W0820 02:37:06.455820       1 client_config.go:617] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
2022-08-20 02:37:06.456 [INFO][1] main.go 124: Ensuring Calico datastore is initialized
2022-08-20 02:37:06.461 [ERROR][1] client.go 290: Error getting cluster information config ClusterInformation="default" error=Get "https://192.168.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": x509: certificate is valid for 127.0.0.1, 10.244.0.1, 192.168.127.110, 192.168.127.120, 192.168.127.130, 192.168.127.200, not 192.168.0.1
2022-08-20 02:37:06.461 [FATAL][1] main.go 129: Failed to initialize Calico datastore error=Get "https://192.168.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": x509: certificate is valid for 127.0.0.1, 10.244.0.1, 192.168.127.110, 192.168.127.120, 192.168.127.130, 192.168.127.200, not 192.168.0.1
```

![](C:\Users\troub\AppData\Roaming\marktext\images\2022-08-20-10-39-19-image.png)

<img src="file:///C:/Users/troub/AppData/Roaming/marktext/images/2022-08-20-10-38-37-image.png" title="" alt="" width="679">

```yml
     volumes:
        - name: kubeconfig
          hostPath:
            path: /root/.kube
        - name: cert
          hostPath:
            path: /opt/kubernetes/server/bin/cert/
```

### 11. 安装CoreDNS

进入任意一台及其执行以下脚本，其中配置文件需要指定 DNS 地址: 10.244.0.2。该地址为 kubelet.json 指定的clusterDNS 地址

```shell
cd ~
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/deploy.sh
chmod +x deploy.sh
./deploy.sh -i 10.244.0.2 >coredns.yml
kubectl apply -f coredns.yml
```

### 12. 安装Metrics-Sever

##### 12.1 安装

执行命令直接安装

```powershell
kubectl apply -f \
https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

若镜像拉取失败，则手动将配置文件下载后修改下镜像地址，将原有的 k8s.gcr.io/metrics-server/metrics-server:v0.6.1 改为下面的地址

```shell
image: bitnami/metrics-server:latest

# 重新执行
kubectl apply -f components.yaml
```

若还是无法拉取镜像，则手动拉取后再复制到其他机器

```shell
crictl pull bitnami/metrics-server:0.6.1

# 打包镜像
ctr -n k8s.io image export metrics.server.0.6.1.tar \
bitnami/metrics-server:0.6.1


# 复制到其他机器
for i in master120 master130;\
do scp metrics.server.0.6.1.tar $i:/opt;done

# 到指定机器导入镜像
ctr -n k8s.io image import /opt/metrics.server.0.6.1.tar
```

安装完成后状态已经为Running， 但是Ready却是 0/1. 查看日志：

```shell
[root@master130 ~]# kubectl logs -f  metrics-server-68655d67c8-6f9dg -n kube-system
E0827 03:07:25.461031       1 scraper.go:140] "Failed to scrape node" err="Get \"https://192.168.127.120:10250/metrics/resource\": x509: cannot validate certificate for 192.168.127.120 because it doesn't contain any IP SANs" node="node02"
E0827 03:07:25.468011       1 scraper.go:140] "Failed to scrape node" err="Get \"https://192.168.127.110:10250/metrics/resource\": x509: cannot validate certificate for 192.168.127.110 because it doesn't contain any IP SANs" node="node01"
E0827 03:07:25.468778       1 scraper.go:140] "Failed to scrape node" err="Get \"https://192.168.127.130:10250/metrics/resource\": x509: cannot validate certificate for 192.168.127.130 because it doesn't contain any IP SANs" node="node03"
I0827 03:07:28.983469       1 server.go:187] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
```

这是因为components.yaml配置文件没有关闭tls认证，回到配置文件修改：

```powershell
  - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  # 加上此行
        image: bitnami/metrics-server:latest
```

至此 k8s 集群终于安装完成，接下来就需要安装

##### 12.2 遇到的问题

<mark>安装kubeSphere遇到的metrics server 问题</mark>：

```shell
E0828 13:37:34.526744       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0828 13:37:53.483884       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0828 13:38:15.521319       1 configmap_cafile_content.go:242] kube-system/extension-apiserver-authentication failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0828 13:38:15.521512       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0828 13:38:53.478913       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0828 13:39:37.465360       1 configmap_cafile_content.go:242] kube-system/extension-apiserver-authentication failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0828 13:39:53.476422       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
```

经过查资料得知这是因为 kube-apiserver 没有开启 `API` 聚合功能。所以需要配置 `kube-apiserver` 参数，开启聚合功能即可。

> ###### 什么是 API 聚合

这里的 `API 聚合机制` 是 Kubernetes 1.7 版本引入的特性，能够将用户扩展的 `API` 注册到 `kube-apiserver` 上，仍然通过 `API Server` 的 `HTTP URL` 对新的 `API` 进行访问和操作。为了实现这个机制，Kubernetes 在 `kube-apiserver` 服务中引入了一个 `API 聚合层（API Aggregation Layer）`，用于将 `扩展 API` 的访问请求转发到用户服务的功能。为了能够将用户自定义的 API 注册到 `Master` 的 `API Server` 中，首先需要在 Master 节点所在服务器，配置 `kube-apiserver` 应用的启动参数来启用 `API 聚合` 功能，参数如下：

```shell
--runtime-config=api/all=true
--requestheader-allowed-names=aggregator
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-client-ca-file=/etc/kubernetes/pki/ca.pem
--proxy-client-cert-file=/etc/kubernetes/pki/proxy-client.pem
--proxy-client-key-file=/etc/kubernetes/pki/proxy-client-key.pem
```

如果 `kube-apiserver` 所在的主机上没有运行 `kube-proxy`，即无法通过服务的 `ClusterIP` 进行访问，那么还需要设置以下启动参数：

```shell
--enable-aggregator-routing=true
```

首先到master130下签发证书

```shell
cd /opt/cert

cat > proxy-client-csr.json << "EOF"
{
    "CN": "aggregator",
    "hosts":[],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C":"CN",
           "ST": "GuangDong",
           "L": "ShenZhen",
           "O": "system:masters",   
           "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
-config=ca-config.json -profile=client \
proxy-client-csr.json | cfssljson -bare proxy-client

# 将证书复制到三台master
for i in master110 master120 master130;\
do scp /opt/cert/proxy-client*.pem \
$i:/opt/kubernetes/server/bin/cert;done
```

接着修改api-server 脚本(已存在的可忽略), 修改完成后重启：

```shell
--runtime-config=api/all=true \
--requestheader-allowed-names=aggregator \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-client-ca-file=./cert/ca.pem \
--proxy-client-cert-file=./cert/proxy-client.pem \
--proxy-client-key-file=./cert/proxy-client-key.pem \
```

### 13. 安装 k8s 包管理工具 Helm

在三台机器分别安装Helm

```shell
wget https://get.helm.sh/helm-v3.9.3-linux-amd64.tar.gz
tar zxvf helm-v3.9.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/bin/


# 配置仓库地址，可以配置阿里云私有仓库地址
# 登录阿里云管控台，进入 https://repomanage.rdc.aliyun.com/my/helm-repos/namespaces
# 新建企业，然后新建namespace 即可看到. 具体用户名，密码需要自行确认，此处不展示
export NAMESPACE=139647-linky
helm repo add $NAMESPACE \
https://repomanage.rdc.aliyun.com/helm_repositories/$NAMESPACE \
--username=xxx--password=xxxx

# 可以使用共有仓库
#微软仓库, 该仓库chart较全
https://mirror.azure.cn/kubernetes/charts/
# 阿里云仓库
https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

#更新仓库
helm repo update
# 微软仓库地址
helm repo add stable https://mirror.azure.cn/kubernetes/charts
```

安装 Helm Push 插件， 用于将包推送到私有仓库中

```
$ helm plugin install https://github.com/chartmuseum/helm-push
```

发布 Chart 目录

```powershell
$ cat mychart/Chart.yaml
name: mychart 
version: 0.3.2
```

```shell
$ helm cm-push mychart/ $NAMESPACE
```

发布 Chart 压缩包

```
$ helm package mychart
```

```
$ helm cm-push mychart-0.3.2.tgz $NAMESPACE
```

更新本地索引

```
$ helm repo update
```

搜索 Chart

```
$ helm search $NAMESPACE/mychart
```

安装 Chart

```
$ helm install $NAMESPACE/mychart
```

### 13. 安装Web管理工具 - Kuboard

#### 13.1 首先安装 storageclass

##### 13.1.1 在 master110 上安装 nfs 服务端

```shell
yum -y install nfs-server nfs-utils

# 创建共享目录
mkdir /data/nfs/k8s -p

# 配置
#前面是共享目录，* 代表所有ip或主机，fsid、anonuid、anongid是给从节点写入权限，0代表root用户
cat >/etc/exports <<EOF 
/data/nfs/k8s 192.168.127.0/24(rw,sync,fsid=0,anonuid=0,anongid=0)
EOF

#配置生效
exportfs -r

#查看生效
[root@master110 ~]# exportfs
/data/nfs/k8s     192.168.127.0/24

#启动rpcbind、nfs服务
systemctl start nfs-server rpcbind
systemctl enable nfs-server rpcbind

# 查看 nfs 是否就绪
[root@master110 ~]# showmount -e
Export list for master110:
/data/nfs/k8s 192.168.127.0/24
```

##### 13.1.2 安装 nfs 客户端

在master120 , master130 上安装 nfs 客户端

```powershell
systemctl start nfs-server rpcbind
# 启动rpc
systemctl restart rpcbind && systemctl enable rpcbind

# 查看共享目录
[root@master130 ~]# showmount -e 192.168.127.110
Export list for 192.168.127.110:
/data/nfs/k8s 192.168.127.0/24

# 创建目录
mkdir /data/nfs/k8s -pr
# 挂载
mount -t nfs 192.168.127.110:/data/nfs/k8s /data/nfs/k8s
# 如果想要开机自动将共享目录挂载到本地,往/etc/fstab 中追加：
echo "192.168.127.110:/data/nfs/k8s /data/nfs/k8s nfs defaults 0 0" >> /etc/fstab
```

##### 13.1.3 安装 StorageClass

```shell
#下载
wget https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/releases/download/nfs-subdir-external-provisioner-4.0.16/nfs-subdir-external-provisioner-4.0.16.tgz
#解压
tar -xvf nfs-subdir-external-provisioner-4.0.16.tgz

cd nfs-subdir-external-provisioner

# 进入 values.yaml 修改 StorageClass 名称 name 为 nfs-storage
vim values.yaml

kubectl create ns nfs
helm install nfs-subdir-external-provisioner . -n nfs \
--set nfs.server=192.168.127.110 \
--set nfs.path=/data/nfs/k8s

# 执行结果 
[root@master130 nfs-subdir-external-provisioner]# helm install  nfs-subdir-external-provisioner . -n nfs
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Sat Aug 27 17:01:47 2022
NAMESPACE: nfs
STATUS: deployed
REVISION: 1
TEST SUITE: None

# 镜像拉取失败
[root@master130 nfs-subdir-external-provisioner]# helm install  nfs-subdir-external-provisioner . -n nfs \
--set nfs.server=192.168.127.110 \
--set nfs.path=/data/nfs/k8s
NAMESPACE     NAME                                               READY   STATUS             RESTARTS   AGE
kube-system   calico-kube-controllers-5b97f5d8cf-gpng8           1/1     Running            0          7h42m
kube-system   calico-node-j5hqp                                  1/1     Running            0          7h42m
kube-system   calico-node-xnz42                                  1/1     Running            0          7h42m
kube-system   calico-node-zn55p                                  1/1     Running            0          7h42m
kube-system   coredns-754f9b4f7c-rzgmh                           1/1     Running            0          7h2m
kube-system   metrics-server-79d59589b9-smss2                    1/1     Running            0          5h54m
nfs           nfs-subdir-external-provisioner-675dcb79b5-s6ng2   0/1     ImagePullBackOff   0          7m24s
```

自行构建镜像，首先到github 创建一个项目，项目中仅需要一个Dockerfile文件，内容如下：

```dockerfile
FROM k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
MAINTAINER linky 182867664@qq.com
```

然后到[阿里云管控台](https://cr.console.aliyun.com/cn-hangzhou/instance/repositories)使用刚刚创建的github地址构建镜像（最好将镜像仓库设置为公开）。构建好之后找到阿里云提供的镜像地址回到机器拉取镜像。执行最后将构建好的地址放到 values.yaml ，再重新执行命令

```powershell
# 拉取镜像，拉取成功后pod自动会变为 Running
crictl pull registry.cn-hangzhou.aliyuncs.com/linkyk8s/nfs-subdir-external-provisioner:v4.0.2

# 如果还是出错则可以先卸载后再install试试
helm uninstall nfs-subdir-external-provisioner -n nfs
```

> 若镜像还是无法拉取，则可以考虑到谷歌提供的[CloudShell](https://shell.cloud.google.com/?pli=1&show=ide%2Cterminal)去拉取，然后推送到自己的仓库。Cloud Shell 上面可以查询到k8s及gcr.io 中的所有镜像

最后设置默认的 StorageClass:

```shell
kubectl patch storageclass nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 检查集群中是否有默认 StorageClass
[root@master130 ~]# kubectl get sc
NAME                    PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   38s
```

#### 13.2 安装

使用 hostPath 提供持久化方式安装。但是结果却出现未知错误

```shell
kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3.yaml
```

所以考虑使用 StorageClass 提供持久化的方式安装

```shell
curl -o kuboard-v3.yaml https://addons.kuboard.cn/kuboard/kuboard-v3-storage-class.yaml
```

编辑 kuboard-v3.yaml 修改 KUBOARD_ENDPOINT 为某个工作节点地址，同时指定storageClassName为 nfs-storage，同时将etcd中的两个端口 2380， 2379 修改为 2480， 2479 . 最后启动

```shell
kubectl create -f kuboard-v3.yaml
```

遇到的问题：

- kubectl delete -f 删除卡死。解决方式是先删除namespace, 执行命令:
  
  ```powershell
  kubectl get ns kuboard -o json > kuboard.json
  
  # 编辑 kuboard.json 把 spec 中的 finalizers 删除
  vim kuboard.json
  ```
  
  最后在新开一个终端窗口启动监听 。
  
  ```shell
  kubectl proxy --port=8081
  ```
  
  然后执行命令将 namespace删除
  
  ```powershell
   curl -k -H "Content-Type:application/json" -X PUT --data-binary @kuboard.json http://127.0.0.1:8081/api/v1/namespaces/kuboard/finalize
  ```

### 14. 常用脚本

清空日志

```shell
echo "" > /var/logs/kubernetes/kube-controller-manager/controller.stdout.log 
echo "" > /var/logs/kubernetes/kube-proxy/proxy.stdout.log 
echo "" > /var/logs/kubernetes/kube-scheduler/scheduler.stdout.log 
echo "" > /var/logs/kubernetes/kube-apiserver/apiserver.stdout.log
```

根据端口杀死进程

```shell
# etcd 
netstat -ltunp|grep 2380|awk '{print $7}'|awk -F "/" '{system("echo "$1";kill -9 "$1)}'

# kube-apiserver
supervisorctl stop kube-apiserver
netstat -ltunp|grep 6443|awk '{print $7}'|awk -F "/" '{system("echo "$1";kill -9 "$1)}'

# kube-controller-manager
netstat -ltunp|grep 10257|awk '{print $7}'|awk -F "/" '{system("echo "$1";kill -9 "$1)}'
# kube-scheduler
netstat -ltunp|grep 10259|awk '{print $7}'|awk -F "/" '{system("echo "$1";kill -9 "$1)}'

# kubelet
netstat -ltunp|grep 10250|awk '{print $7}'|awk -F "/" '{system("echo "$1";kill -9 "$1)}'
netstat -ltunp|grep 10255|awk '{print $7}'|awk -F "/" '{system("echo "$1";kill -9 "$1)}'
```

**部署集群遇到的问题**

1. etcd 集群出错，删除etcd所有数据后重启(<mark>切记不能这么做</mark>， 可以先做备份)，导致apiserver也重启。重启完成后 kubectl get pods -A 无数据，此时需要重启kubelet

#### 15. 待办

1. 部署完成集群后备份etcd数据，然后再恢复，查看恢复情况
