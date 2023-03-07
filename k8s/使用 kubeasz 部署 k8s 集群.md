## 使用 kops 部署 k8s 集群

### 1. 前期准备

####  设置主机名并开启免密登录

```sh
cat >> /etc/hosts << EOF
192.168.127.200 k8s.master1
192.168.127.210 k8s.master2
192.168.127.220 k8s.master3
EOF

# 逐个机器设置主机名 
hostnamectl --static set-hostname k8s.master1
```

```sh
cd /root
ssh-keygen -t rsa
for i in k8s.master1 k8s.master2 k8s.master3 ;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done	
```



### 2. 首先安装 kubectl

参照的是 k8s 官网安装，参见 https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

```shell
# 具体安装版本可自行决定，此处使用的是 v1.25.0
curl -LO "https://dl.k8s.io/release/v1.25.0/bin/linux/amd64/kubectl"
# 安装最新版本可以使用
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client
```

### 