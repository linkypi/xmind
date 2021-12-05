## K8S 证书

[参考官网](https://kubernetes.io/zh/docs/setup/best-practices/certificates/)

首先查看k8s需要用到的证书：

```sh
kubeadm init phase certs all --cert-dir=/opt/kubernetes/server/bin/pki
```

通过tree命令查看

```
[root@vms1 bin]# tree pki
pki
├── apiserver.crt
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── etcd
│   ├── ca.crt
│   ├── ca.key
│   ├── healthcheck-client.crt
│   ├── healthcheck-client.key
│   ├── peer.crt
│   ├── peer.key
│   ├── server.crt
│   └── server.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
└── sa.pub
```

所有证书主要通过两部分区分：颁发者（Issuer: CN）与持有者（Subject: CN）。颁发者即由ca-csr.json根证书请求文件中指定的 CN，持有者由证书请求文件中的CN与O决定

### 1.  颁发者 kubernetes

|             证书             |    颁发者     |         验证方式         |                       持有者                        | 证书用途                                         |
| :--------------------------: | :-----------: | :----------------------: | :-------------------------------------------------: | ------------------------------------------------ |
|            ca.crt            | CN=kubernetes |            /             |                    CN=kubernetes                    |                                                  |
|        apiserver.crt         | CN=kubernetes |       server auth        |                  CN=kube-apiserver                  |                                                  |
| apiserver-kubelet-client.crt | CN=kubernetes |       client auth        | O=system:masters,  CN=kube-apiserver-kubelet-client | apiserver作为客户端连接kubelet使用               |
|          admin.crt           | CN=kubernetes |       client auth        |        O=system:masters, CN=kubernetes-admin        | kubectl 连接 apiserver使用，用于配置集群的管理员 |
|         kubelet.crt          | CN=kubernetes | client auth, server auth |     O=system:nodes, CN=system:node:`<nodeName>`     | 集群中的每个节点都需要一份                       |
|        scheduler.crt         | CN=kubernetes |       client auth        |              CN=system:kube-scheduler               | scheduler                                        |
|    controller-manager.crt    | CN=kubernetes |       client auth        |          CN=system:kube-controller-manager          |                                                  |



### 2. 颁发者 etcd

|           证书            |   颁发者   |         验证方式         |                      持有者                       |
| :-----------------------: | :--------: | :----------------------: | :-----------------------------------------------: |
|          ca.crt           | CN=etcd-ca |            /             |                   CN=kubernetes                   |
|  healthcheck-client.crt   | CN=etcd-ca |       client auth        | O=system:masters, CN=kube-etcd-healthcheck-client |
|         peer.crt          | CN=etcd-ca | client auth, server auth |                  CN=`<hostname>`                  |
|        server.crt         | CN=etcd-ca |       server auth        |                  CN=`<hostname>`                  |
| apiserver-etcd-client.crt | CN=etcd-ca |       client auth        |  O=system:masters, CN=kube-apiserver-etcd-client  |

### 3. 颁发者 front-proxy

|          证书          |      颁发者       |  验证方式   |        持有者         |
| :--------------------: | :---------------: | :---------: | :-------------------: |
|   front-proxy-ca.crt   | CN=front-proxy-ca |      /      |   CN=front-proxy-ca   |
| front-proxy-client.crt | CN=front-proxy-ca | client auth | CN=front-proxy-client |

### 4. 为颁发者 kubernetes 签发证书

#### 4.1 签发根证书

新建文件夹 pki 用于存放所有证书文件，并创建证书配置文件 ca-config.json 及根证书请求文件 ca-csr.json，证书有效期为 100 年

```sh
mkdir /opt/pki
cd /opt/pki

cat > /opt/pki/ca-config.json << "EOF"
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "peer": {
        "usages": [
	        "digital signature",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      },
      "server": {
        "usages": [
	        "digital signature",
            "key encipherment",
            "server auth"
        ],
        "expiry": "876000h"
      },
      "client": {
        "usages": [
	        "digital signature",
            "key encipherment",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF

cat > /opt/pki/ca-csr.json << "EOF"
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "ca": { 
      "expiry": "876000h"
  }
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

#### 4.2 签发 apiserver 证书

```sh
cd /opt/pki
cat > /opt/pki/apiserver-csr.json << "EOF"
{
  "CN": "kube-apiserver",
  "hosts": [
	"localhost",
	"127.0.0.1",
    "192.168.122.110",
    "192.168.122.120",
    "192.168.122.130",
    "192.168.122.140",
    "192.168.122.150",
    "192.168.122.160",
    "192.168.122.170",
    "192.168.122.180",
    "192.168.122.188",
    "192.168.122.190",
    "192.168.122.200",
	"kubernetes",
	"kubernetes.default",
	"kubernetes.default.svc",
	"kubernetes.default.svc.cluster",
	"kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server apiserver-csr.json | cfssljson -bare apiserver
```

#### 4.3 签发 apiserver-kubelet-client 证书

```sh
cat > /opt/pki/apiserver-kubelet-client-csr.json << "EOF"
{
  "CN": "kube-apiserver-kubelet-client",
  "hosts": [
    "192.168.122.110",
    "192.168.122.120",
    "192.168.122.130",
    "192.168.122.140",
    "192.168.122.150",
    "192.168.122.160",
    "192.168.122.170",
    "192.168.122.180",
    "192.168.122.188",
    "192.168.122.190"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client apiserver-kubelet-client-csr.json | cfssljson -bare apiserver-kubelet-client
```

#### 4.4 签发 admin 证书

```sh
cat > /opt/pki/admin-csr.json << "EOF"
{
  "CN": "kubernetes-admin",
  "hosts": [
    "192.168.122.110",
    "192.168.122.120",
    "192.168.122.130",
    "192.168.122.140",
    "192.168.122.150",
    "192.168.122.160",
    "192.168.122.188"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client admin-csr.json | cfssljson -bare admin
```

#### 4.5 签发 kubelet 证书

````sh
cat > /opt/pki/kubelet01-csr.json << "EOF"
{
  "CN": "system:node:node01",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "192.168.122.140"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:nodes"
    }
  ]
}
EOF
cat > /opt/pki/kubelet02-csr.json << "EOF"
{
  "CN": "system:node:node02",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "192.168.122.150"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:nodes"
    }
  ]
}
EOF
cat > /opt/pki/kubelet03-csr.json << "EOF"
{
  "CN": "system:node:node03",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "192.168.122.160"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:nodes"
    }
  ]
}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kubelet01-csr.json | cfssljson -bare kubelet-node01
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kubelet02-csr.json | cfssljson -bare kubelet-node02
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kubelet03-csr.json | cfssljson -bare kubelet-node03
````



#### 4.6 签发 scheduler 证书

````sh
cat > /opt/pki/scheduler-csr.json << "EOF"
{
  "CN": "system:kube-scheduler",
  "hosts": [
    "192.168.122.140",
    "192.168.122.150",
    "192.168.122.160"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client scheduler-csr.json | cfssljson -bare scheduler
````

#### 4.7 签发 controller-manager 证书

````sh
cat > /opt/pki/controller-manager-csr.json << "EOF"
{
  "CN": "system:kube-controller-manager",
  "hosts": [
        "192.168.122.140",
        "192.168.122.150",
        "192.168.122.160"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client controller-manager-csr.json | cfssljson -bare controller-manager
````



### 5. 为颁发者 etcd 签发证书

#### 5.1 签发根证书

在pki文件夹中创建etcd用于创建etcd使用的证书文件

```sh
mkdir -p /opt/pki/etcd && cd /opt/pki/etcd

cat > /opt/pki/etcd/ca-config.json << "EOF"
{
    "signing": {
        "profiles": {
           "peer": {
                "usages": [
                    "digital signature",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "876000h"
             },
            "server": {
                "expiry": "876000h",
                "usages": [
		            "digital signature",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "876000h",
                "usages": [
		            "digital signature",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}
EOF

cat > /opt/pki/etcd/ca-csr.json << "EOF"
{
  "CN": "etcd-ca",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

#### 5.2 签发 server 证书

```sh
cd /opt/pki/etcd
cat > /opt/pki/etcd/server-csr.json << "EOF"
{
  "CN": "kube-etcd",
  "hosts": [
     "localhost",
     "127.0.0.1",
     "192.168.122.110",
     "192.168.122.120",
     "192.168.122.130"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server-csr.json | cfssljson -bare etcd-server
```



#### 5.3 签发 apiserver-etcd-client 证书

#### 

```sh
cd /opt/pki/etcd
cat > /opt/pki/etcd/apiserver-etcd-client-csr.json << "EOF"
{
 "CN": "kube-apiserver-etcd-client",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client apiserver-etcd-client-csr.json | cfssljson -bare apiserver-etcd-client
```



### 6. 为颁发者 front-proxy 签发证书

#### 6.1 签发根证书

在pki文件夹中创建proxy用于创建front-proxy 使用的证书文件

```sh
mkdir /opt/pki/proxy && cd /opt/pki/proxy
cat > /opt/pki/proxy/ca-config.json << "EOF"
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "client": {
        "expiry": "876000h",
        "usages": [
           "digital signature",
           "key encipherment",
           "client auth"
        ]
      }
    }
  }
}
EOF

cat > /opt/pki/proxy/ca-csr.json << "EOF"
{
  "CN": "kubernetes-front-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "ca": { 
      "expiry": "876000h"
  }
}
EOF
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

#### 6.2 签发 proxy-client 证书

````sh
cat > /opt/pki/proxy/client-csr.json << "EOF"
{
  "CN": "front-proxy-client",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json | cfssljson -bare proxy-client
````

### 