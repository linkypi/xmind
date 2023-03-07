## 1. 各环境域名及地址规划

### 1.1 域名规划

开发环境：presit.edesop.com 

测试环境：sit.edesop.com 

产品验收环境：uat.edesop.com

### 1.2 地址规划

#### 1.2.1 管理系统：

/web 前端页面

/api 接口API

/doc 接口文档

#### 1.2.2 中台系统：

/cms 前端页面

/cms/api 接口API

/cms/doc 接口文档

#### 1.2.3 H5

/h5 前端页面

/h5/api 接口API

/h5/doc 接口文档

## 2. 命名空间 ns 及 Label标签设定

```
namespace: presit/sit/uat
tier: frontend/backend/middleware
env: presit/sit/uat
version: 
language: java
```

## 3. 网络策略配置

```yml
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: deny-dev-namesapce-to-test
spec:
  namespaceSelector: env == "presit"
  ingress:
  - action: Allow
  egress:
  - action: Deny
    destination:
      namespaceSelector: env == "test"
  - action: Allow
```

## 

## 4. 配置容器运行时 containerd 私有仓库地址

编辑 /etc/containerd/config.toml，在 plugins."io.containerd.grpc.v1.cri".registry.mirrors 下增加一个仓库域名 k8s.nexus，k8s.nexus 该域名需要在 /etc/hosts 配置为私有仓库所在主机对应的 IP

```
[plugins."io.containerd.grpc.v1.cri".registry.configs]
      [plugins."io.containerd.grpc.v1.cri".registry.configs."k8s.nexus".auth]
          username = "admin"
          password = "password"

 [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.nexus"]
         endpoint = ["http://192.168.57.31:8090"]
```

同时 plugins."io.containerd.grpc.v1.cri".registry.configs 下需要增加私有仓库对应的账户及密码。配置完成后重启 containerd即可

## 5. 部署基础中间件

### 5.1 部署nacos

首先下载部署nacos所用文件， 进入 master1 opt目录执行命令

```shell
git clone https://github.com/nacos-group/nacos-k8s.git
```

由于nacos日志等数据需要持久话，所以需要使用PVC

#### 5.1.1 部署nfs客户端

进入nacos-k8s目录

```shell
# 修改namespace, presit/sit/uat
sed -i'' "s/namespace:.*/namespace: presit/g" ./deploy/nfs/rbac.yaml
```

修改 deployment.yaml 需要共享的nfs服务器目录

```shell
vim deploy/nfs/deployment.yaml
```

修改如下：

```yaml
containers:
    - name: nfs-client-provisioner
      image: quay.io/external_storage/nfs-client-provisioner:latest
      volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
      env:
        - name: PROVISIONER_NAME
          value: fuseim.pri/ifs
        - name: NFS_SERVER
          value: 192.168.57.31  # nfs服务器地址
        - name: NFS_PATH
          value: /data/nfs/k8s  # nfs 共享目录
volumes:
    - name: nfs-client-root
      nfs:
        server: 192.168.57.31 # nfs服务器地址
        path: /data/nfs/k8s   # nfs 共享目录
```

最后执行命令：

```shell
kubectl create -f deploy/nfs/rbac.yaml
kubectl create -f deploy/nfs/deployment.yaml
kubectl create -f deploy/nfs/class.yaml



kubectl delete -f deploy/nfs/rbac.yaml
kubectl delete -f deploy/nfs/deployment.yaml
kubectl delete -f deploy/nfs/class.yaml
```

查看安装情况：

```shell
kubectl get pod -l app=nfs-client-provisioner
```

### 5.1.2 安装nacos

首先修改nacos连接的数据库地址及用户密码信息，即需要修改文件deploy/nacos/nacos-pvc-nfs.yaml

```shell
vim deploy/nacos/nacos-pvc-nfs.yaml
```

```yaml
data:
  mysql.master.db.name: "主库名称"
  mysql.master.port: "主库端口"
  mysql.slave.port: "从库端口"
  mysql.master.user: "主库用户名"
  mysql.master.password: "主库密码"
```

由于presit环境使用的是外接数据库，所以仍需要设置数据库访问地址。同样编辑 deploy/nacos/nacos-pvc-nfs.yaml 文件，找到 MYSQL_SERVICE_DB_NAME 所在的env, 然后按照 MYSQL_SERVICE_DB_NAME 在其后方增加一个  MYSQL_SERVICE_HOST。

然后在ConfigMap的data下增加mysql.host即可

```yaml
data:
  mysql.host: "192.168.57.30"


- name: MYSQL_SERVICE_DB_NAME
  valueFrom:
    configMapKeyRef:
      name: nacos-cm
      key: mysql.db.name
- name: MYSQL_SERVICE_HOST
  valueFrom:
    configMapKeyRef:
      name: nacos-cm
      key: mysql.host
```

## 6. 部署 dubbo

由于拉取镜像需要制定仓库及其账户，所以需要创建 secret

```shell
kubectl create secret generic nexussecret \
    --from-file=.dockerconfigjson=/root/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson

kubectl get secrets nexussecret \
--output="jsonpath={.data.\.dockerconfigjson}" | base64 -d
```

创建部署yml文件，注意统一namespace

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: dubbo
  namespace: test
  labels:
    name: dubbo
spec:
  replicas: 1
  selector:
    matchLabels:
      name: dubbo
      version: "4.8.0"
  template:
    metadata:
      labels:
        app: dubbo
        name: dubbo
        version: "4.8.0"
    spec:
      containers:
      - name: presit-esop-dubbo
        image: k8s.nexus/docker-release/presit-esop-dubbo:dev_4.8.0
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 20882
          protocol: TCP
        imagePullPolicy: IfNotPresent
        env:
        - name: JVM_OPTS
          value: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/java_heapdump.hprof -Xmx1024m -Xms512m
        - name: profile
          value: presit
        - name: LOG_PATH
          value: ./logs
        volumeMounts:
        - mountPath: ./logs    # 容器内路径
          name: log
      volumes:
      - name: log
        hostPath:
          path: /opt/cicd/dubbo/logs
      hostAliases:
      - ip: 127.0.0.1
        hostnames:
        - "presit-esop-dubbo"
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsUser: 0
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600

---
apiVersion: v1
kind: Service
metadata:
  name: dubbo
  namespace: test
spec:
  ports:
   - port: 20882
     targetPort: 20882
  selector:
    app: dubbo
```

使用configmap创建dubbo配置文件

```shell
kubectl create configmap dubbo-test-config --from-file=config.yml
```

## 7. 待办

1. deployment 增加健康检查

2. gitlab 对接 sonarqube ，完善代码审查

3. 使用gitlab 流水线替代 Jenkins

4. gitlab uat 流水线增加生成生产镜像的stage, 在产品验收通过后手动触发
