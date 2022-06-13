#### 1. 安装ipvs管理工具及模块

> 注意: 仅为集群节点安装，负载均衡节点无需安装

```sh
yum install -y ipvsadm ipset sysstat conntrack libseccomp
```

配置 ipvs.conf，创建文件 /etc/modules-load.d/ipvs.conf ，并加入：

```sh
cat /etc/modules-load.d/ipvs.conf <<EOF
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

#### 2. 加载 containerd 内核模块

```sh
cat > /etc/modules-load.d/containerd.conf << EOF
overlay
br_netfilter
EOF

# 开机启动
systemctl enable --now systemd-modules-load.service
```

#### 3. etcd 配置

```sh
cat > /etcd/etcd.conf << EOF
#[Memeber]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.10.12:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.10.12:2379,https://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.10.12:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.10.12:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://192.168.10.12:2380,etcd2=https://192.168.10.13:2380,etcd3=https://192.168.10.14:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

