K8S 创建 CRD

操作CRD通常可以直接使用 `client-go` 提供的 `dynamicClient` 来操作自定义资源对象, 而使用 [code-generator](https://github.com/kubernetes/code-generator) 可以简化其中很多操作. 首先clone代码,并切换到当前k8s集群所支持的分支. 然后根据需要开始安装相关代码生成工具. 注意下面的命令不可以在Windows系统的cmd下执行, 可以通过mingw64 的bash执行

```shell
go install ./cmd/{client-gen,deepcopy-gen,informer-gen,lister-gen}
```

执行完成后就会在 GOPATH的bin目录下生成相关命令, 可以将该目录放入到系统环境变量 PATH中

```
go env GOPATH
ls $GOPATH/bin
```

通常是通过 generater-group.sh 来创建, 不会使用单独命令创建.

```shell
go mod init github.com/operator-crd

/d/go/src/code-generator/code-generator/generate-groups.sh all /d/go/src/github.com/operator-crd/pkg/generated /d/go/src/github.com/operator-crd/pkg/apis crd.example.com:v1 --go-header-file=/d/go/src/github.com/operator-crd/hack/boilerplate.go.txt --output-base /

# centos7上执行 路径必须为绝对路径
generate-groups.sh all /root/go/src/github.com/operator-crd/pkg/generated /root/go/src/github.com/operator-crd/pkg/apis crd.example.com:v1 --go-header-file=/root/go/src/github.com/operator-crd/hack/boilerplate.go.txt --output-base /

```



## kubebuilder



```sh
# 创建项目路径
mkdir -p /root/go/src/github.com/linky/operator-example
cd /root/go/src/github.com/linky/operator-example

go mod init github.com/linky/operator-example

# kubebuilder 初始化项目
kubebuilder init --domain github.com --license apache2 --owner "linky"
```

