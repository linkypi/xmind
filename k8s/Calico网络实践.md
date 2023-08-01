

## Calico网络原理分析

Calico 是一种纯三层的实现，因此可以避免与二层方案相关的数据包封装的操作，中间没有任何的 NAT，没有任何的 overlay，所以它的转发效率可能是所有方案中最高的，因为它的包直接走原生 TCP/IP 的协议栈，它的隔离也因为这个栈而变得好做。因为 TCP/IP 的协议栈提供了一整套的防火墙的规则，所以它可以通过 IPTABLES 的规则达到比较复杂的隔离逻辑。

容器 calico 网络 veth pair 查看

```sh
# 查看某个 pod 的IP地址
[root@k8s esop]# kubectl get pod -owide -n sit
NAME                                    READY   STATUS        RESTARTS        AGE     IP             NODE            NOMINATED NODE   READINESS GATES
sit-web-backend-58f48df4c9-b2q28        1/1     Running       0               22h     10.22.151.11   192.168.57.32   <none>           <none>

# 到指定节点机器 192.168.57.32 查看 Pod IP 对应的网卡
[root@k8s esop]# route -n|grep "10.22.151.11"
10.22.151.11    0.0.0.0         255.255.255.255 UH    0      0        0 cali262cda17a90
# 查看网卡索引，对应的是 8608
[root@k8s esop]# ip a|grep cali262cda17a90
8608: cali262cda17a90@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 

# 最后到容器内部查看网卡索引 8608
root@sit-web-backend-58f48df4c9-b2q28:/# cat /sys/class/net/eth0/iflink
8608
```

从上面可以看到 sit-web-backend-xxx这个pod启动后 calico会为其创建一对 veth pair, 其中一端对应的是宿主机的网卡 calicoxxx，另外一端对应的是该容器内部的 eth0 网卡。假设其他节点的pod想要访问该容器，如前端pod想访问该后端pod， 而前端pod是部署在了其他机器节点，如192.168.57.31。那么首先进入192.168.57.31机器查看路由，看看通往后端 10.22.151.11 的路由是如何走：

```sh
[root@k8s esop]# ip route   # 通过 route -n 同样可以看到
default via 192.168.57.1 dev eth0 
blackhole 10.22.138.0/26 proto bird 
10.22.138.1 dev cali0953d667977 scope link 
...
10.22.151.0/26 via 192.168.57.32 dev tunl0 proto bird onlink 
```

可以看到请求后端地址 10.22.151.11 需要经过下一跳，即 192.168.57.32 这台机器，而网卡显示的是 tunl0 (即 IPIP模式)。**同个网段下可以不使用IPIP模式，但是不同网段下通讯就需要IPIP模式，calico为实现集群间部署也就需要用到该模式**

在 32 机器监听 tunl0 网卡，然后使用部署在 31 机器的pod来 ping 32机器上的 pod，结果可以看到其他节点请求该后端pod确实是通过 tunl0 网卡进行通讯。

```sh
# 32 机器操作
[root@k8s ~]# tcpdump -eni tunl0  -nnvv | grep 10.22.151.11
tcpdump: listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
    10.22.138.0 > 10.22.151.11: ICMP echo request, id 30387, seq 1, length 64
    10.22.151.11 > 10.22.138.0: ICMP echo reply, id 30387, seq 1, length 64
    10.22.138.0 > 10.22.151.11: ICMP echo request, id 30387, seq 2, length 64
    10.22.151.11 > 10.22.138.0: ICMP echo reply, id 30387, seq 2, length 64
    
# 31 机器操作
[root@k8s ~]# ping 10.22.151.11
```

因为Calico默认使用的是 IPIP模式，相关配置如下：

```yaml
 # Enable IPIP, value 为 Never 表示关闭
 - name: CALICO_IPV4POOL_IPIP
   value: "Always" # 还可以设置为 "CrossSubnet" 表示只有在跨子网的情况下才会使用 IPIP 模式
```

IPIP 模式下的 calico-node 在启动后会在当前宿主机创建一个 tunnel 虚拟网卡 tunl0。相关日志 log 记录在宿主机的 `/var/log/calico/cni` 目录下。通过以下命令来查看该网卡

```sh
[root@k8s esop]# ip a|grep tunl0
2: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 10.22.138.0/32 scope global tunl0
```

首先进入某个pod容器内部查看 IP，若容器内部没有相关命令则可以通过以下命令安装

```sh
# debian 或 Ubuntu系统
apt update
apt install iproute2
```

安装后查下

```sh
# 查看 IP 信息
root@sit-web-backend-58f48df4c9-b2q28:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if8608: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether f6:7f:37:81:fc:15 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.22.151.11/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f47f:37ff:fe81:fc15/64 scope link 
       valid_lft forever preferred_lft forever
```

通过容器内部IP及路由信息可以看到，容器的IP是 10.22.151.11/32 ，即这是32位主机地址，表示容器是一个单点局域网。其内部使用的 的网卡eth0对应的宿主机网卡索引是 8608，可以通过上方的 **eth0@if8608** 看出，同时还可以在容器内部通过以下命令查看

```sh
root@sit-web-backend-58f48df4c9-b2q28:/# cat /sys/class/net/eth0/iflink
8608
```

由于Calico使用虚拟路由代替虚拟交换，每一台虚拟路由通过 BGP 协议传播路由信息，所有的数据包都是通过路由的形式找到对应的主机和容器, 故必定与其路由有关，接下来再看看容器的路由信息

```sh
root@sit-web-backend-58f48df4c9-b2q28:/# ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0 scope link
```

结果发现有一个路由 IP ：169.254.1.1，当一个数据包的目的地址不是本机地址时，就会查询路由表，从路由表中查到网关后，它首先会通过 `ARP` 获得网关的 MAC 地址，然后在发出的网络数据包中将目标 MAC 改为网关的 MAC，而网关的 IP 地址不会出现在任何网络包头中。即无需关心该路由IP地址，只需可以正常找到MAC地址并响应 ARP 请求即可。此时继续深挖，从ARP缓存出发

```sh
# 查看容器ARP缓存
root@sit-web-backend-58f48df4c9-b2q28:/# ip neigh
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee REACHABLE
```

可以看到路由对应的MAC地址竟然无用的MAC地址 ee:ee:ee:ee:ee:ee。这是因为Calico使用了网卡的代理ARP。**代理 ARP 是 ARP 协议的一个变种，当 ARP 请求目标跨网段时，网关设备收到此 ARP 请求，会用自己的 MAC 地址返回给请求者，这便是代理 ARP（Proxy ARP）**

在主机上通过开启代理 ARP 功能来实现 ARP 应答，使得 ARP 广播被抑制在主机上，抑制了广播风暴，也不会有 ARP 表膨胀的问题。下面在宿主机看下对应的网卡是否已经开启代理ARP：

```sh
# 查看相关网卡是否已开启ARP代理
[root@k8s esop]# cat /proc/sys/net/ipv4/conf/cali262cda17a90/proxy_arp
1
```



```sh
# 在宿主机查看网卡信息，看到其MAC地址确实是 ee:ee:ee:ee:ee:ee
[root@k8s esop]# ip a|grep cali262cda17a90 -A 10
8608: cali262cda17a90@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 82
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever

```

从以上就可以看出，**Calico 通过一个巧妙的方法将workload 的所有流量引导到一个特殊的网关 169.254.1.1，从而引流到主机的 calixxx 网络设备上，最终将二三层流量全部转换成三层流量来转发**，这样也就实现了不同宿主机之间的网络连接。

在 32 机器监听 tunl0 网卡，然后使用部署在 31 机器的pod来 ping 32机器上的 pod，结果可以看到其他节点请求该后端pod确实是通过 tunl0 网卡进行通讯。

```sh
[root@k8s sit-seata]# tcpdump -eni tunl0  -nnvv | grep 10.22.151.11
tcpdump: listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
    10.22.138.0 > 10.22.151.11: ICMP echo request, id 30387, seq 1, length 64
    10.22.151.11 > 10.22.138.0: ICMP echo reply, id 30387, seq 1, length 64
    10.22.138.0 > 10.22.151.11: ICMP echo request, id 30387, seq 2, length 64
    10.22.151.11 > 10.22.138.0: ICMP echo reply, id 30387, seq 2, length 64
```



