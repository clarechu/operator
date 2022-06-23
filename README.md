# Operator


* Operator 是由 CoreOS 开发的，用来扩展 Kubernetes API，特定的应用程序控制器，它用来创建、配置和管理复杂的有状态应用，如数据库、缓存和监控系统。Operator 基于 Kubernetes 的资源和控制器概念之上构建，但同时又包含了应用程序特定的领域知识。创建Operator 的关键是CRD（自定义资源）的设计。
* Kubernetes 1.7 版本以来就引入了自定义控制器的概念，该功能可以让开发人员扩展添加新功能，更新现有的功能，并且可以自动执行一些管理任务，这些自定义的控制器就像 Kubernetes 原生的组件一样，Operator 直接使用 Kubernetes API进行开发，也就是说他们可以根据这些控制器内部编写的自定义规则来监控集群、更改 Pods/Services、对正在运行的应用进行扩缩容。

前提条件:

* Kubernetes 1.20.0 版本
* Operator-sdk version: v1.22.0（这里有最新版就安装最新版的吧）
* go version go1.11.4 darwin/amd64

## 主要功能

Operator SDK 提供以下工作流来开发一个新的 Operator：

使用 SDK 创建一个新的 Operator 项目 通过添加自定义资源（CRD）定义新的资源 API 指定使用 SDK API 来 watch 的资源 
定义 Operator 的协调（reconcile）逻辑 使用 Operator SDK 构建并生成 Operator 部署清单文件

## QuickStart

安装 operator-sdk
operator sdk 安装方法非常多，我们可以直接在 github 上面下载需要使用的版本，然后放置到 PATH 环境下面即可，当然也可以将源码 clone 到本地手动编译安装即可，如果你是 Mac，当然还可以使用常用的 brew 工具进行安装： 我使用brew 安装的版本是v0.16.0

```bash

# macos
$ brew install operator-sdk

$ operator-sdk version
operator-sdk version: "v1.22.0", commit: "9e95050a94577d1f4ecbaeb6c2755a9d2c231289", kubernetes version: "v1.24.1", go version: "go1.18.3", GOOS: "darwin", GOARCH: "amd64"
```

### 创建项目

1. 添加一个项目名称为operator 的项目

```bash

# 创建文件夹

$ mkdir -p operator

# 创建go.mod

$ go mod init github.com/clarechu/operator

```

2. 创建脚手架工具

```bash
$   operator-sdk create api --group app --version v1beta1 --kind Student
```

3. 创建student crd资源

首先在`config/crd/bases`目录下面创建crd的yaml `<group>.<domain>_<kind>.yaml`

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.2.5
  creationTimestamp: null
  name: students.app.demo.com
spec:
  group: app.demo.com
  versions:
    - name: v1beta1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
  names:
    kind: Student
    listKind: StudentList
    plural: students
    singular: student
  scope: Namespaced
```

现在我们添加一个Student的crd资源

```bash
$ kustomize build config/crd |kubectl apply -f -
```

打开源文件api/v1beta1/student_types.go, 需要我们根据我们的需求去自定义结构体 AppServiceSpec,
我们最上面预定义的crd yaml中的属性，所有我们需要用到的属性都需要在这个结构体中进行定义：

```bash
// StudentSpec defines the desired state of Student
type StudentSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Student. Edit student_types.go to remove/update
	Foo string `json:"foo,omitempty"`
	
	Size int32 `json:"size,omitempty"`
	
	...
}
```

现在我们运行我们的main 方法

```log
Got a connection, launched process /private/var/folders/2w/m2f1gwjx2k972gxb6bdsp5tr0000gn/T/GoLand/___go_build_github_com_clarechu_operator (pid = 95641).
I0623 18:21:28.984280   95641 request.go:601] Waited for 1.046757348s due to client-side throttling, not priority and fairness, request: GET:https://lb.kubesphere.local:6443/apis/node.k8s.io/v1beta1?timeout=32s
1.655979689038683e+09   INFO    controller-runtime.metrics      Metrics server is starting to listen    {"addr": ":8080"}
1.655979689039343e+09   INFO    setup   starting manager
1.6559796890397902e+09  INFO    Starting server {"kind": "health probe", "addr": "[::]:8081"}
1.655979689039789e+09   INFO    Starting server {"path": "/metrics", "kind": "metrics", "addr": "[::]:8080"}
1.655979689040031e+09   INFO    Starting EventSource    {"controller": "student", "controllerGroup": "app.demo.com", "controllerKind": "Student", "source": "kind source: *v1beta1.Student"}
1.6559796890401092e+09  INFO    Starting Controller     {"controller": "student", "controllerGroup": "app.demo.com", "controllerKind": "Student"}
1.655979689143872e+09   INFO    Starting workers        {"controller": "student", "controllerGroup": "app.demo.com", "controllerKind": "Student", "worker count": 1}


```


最后我们创建一个student对象


```yaml
$ kubectl apply -f - << EOF
apiVersion: app.demo.com/v1beta1
kind: Student
metadata:
  name: xxx
  namespace: default
spec:
  size: 2
  foo: xx
EOF
```

可以看到运行日志

```log
1.655979689143872e+09   INFO    Starting workers        {"controller": "student", "controllerGroup": "app.demo.com", "controllerKind": "Student", "worker count": 1}
```

执行下面的命令构建 Operator 应用打包成 Docker 镜像：

```bash
$ make docker-build
```

