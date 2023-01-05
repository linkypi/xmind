## k8s生产化实践



### 1. 日志管理

1. kubelet 配置参数 container-log-max-size, container-log-max-files



### 2. 限制网络带宽

1. 可利用CNI社区插件 bandwidth插件，然后在代码指定
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
      annotation:
          kubernetes.io/ingress-bandwidth: 10MB
          kubernetes.io/egress-bandwidth: 10MB
   ```



### 3.  节点问题主动上报

对于硬件问题，内核问题，容器运行时等问题，k8s无法感知，所以可以使用  node-problem-detecor 守护进程收集节点问题并上报
