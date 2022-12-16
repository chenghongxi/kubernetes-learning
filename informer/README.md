# Informer

## 什么是 Informer
```text
informer 是 client-go 中的核心工具包，informer 其实就是一个带有本地缓存和索引机制的，可以注册 EventHandler 的 client 本地缓存被称为 Store,
索引被称为 Index 。使用 informer 的目的是为了减轻 apiserver 数据交互的压力而抽象出来的一个 cache 层，客户端对 apiserver 数据的 读取和监听操作都通过本地 informer进行。
```

## 为什么需要 Informer 机制
```text
Kubernetes 各个组件都是通过 REST API 跟 API Server 交互通信的，而如果每次每一个组件都直接跟 API Server 交互去读取/写入到后端的etcd的话，
会对 API Server 以及etcd造成非常大的负担。 而 Informer 机制是为了保证各个组件之间通信的实时性、可靠性，并且减缓对 API Server和etcd 的负担。
```

## 对比:

### 代码对比:
- `1. 使用了 informer:`
    
    代码链接: https://github.com/chenghongxi/go-learning/blob/main/gin/gin-informer.go
```text
package main

import (
	"context"
	"log"
	"net/http"
	"path/filepath"

	"github.com/gin-gonic/gin"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

// 使用 informer 减轻 kubernetes apiserver 并发压力
// use two clients
// 一个 client 负责查资源
// 一个 client 负责资源的创建，更新，删除
func main() {
	// 从家目录读取config
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(homedir.HomeDir(), ".kube", "config"))
	if err != nil {
		panic(err)
	}
	// 使用config，返回clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// get all namespace sharedInformers
	sharedInformers := informers.NewSharedInformerFactory(clientset, 0)

	// get Resource
	gvrs := []schema.GroupVersionResource{
		{Group: "apps", Version: "v1", Resource: "deployments"},
		{Group: "", Version: "v1", Resource: "pods"},
	}

	for _, gvr := range gvrs {
		if _, err = sharedInformers.ForResource(gvr); err != nil {
			panic(err)
		}
	}

	// Start all informers
	sharedInformers.Start(ctx.Done())
	// Wait for all caches to sync
	sharedInformers.WaitForCacheSync(ctx.Done())

	log.Printf("all informers has been started")

	// 构造 pod Lister，用于 gin 的查询
	podLister := sharedInformers.Core().V1().Pods().Lister()
	// 启动 gin router
	// 仅作演示， 无封装， 无异常处理
	// 启动之后，curl 127.0.0.1:8080/pods
	r := gin.Default()
	r.GET("/pods", func(c *gin.Context) {
		pods, err := podLister.List(labels.Everything())
		if err != nil {
			panic(err)
			c.JSON(http.StatusBadRequest, gin.H{"message": err, "code": 400})
			return
		}
		c.JSON(http.StatusOK, gin.H{"message": "pong", "code": 200, "result": pods})
	})
	_ = r.Run(":8080")
}
```
- `2. 不使用 informer`

    代码链接: https://github.com/chenghongxi/go-learning/blob/main/gin/gin-no-informer.go
```text
package main

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"net/http"
	"path/filepath"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(homedir.HomeDir(), ".kube", "config"))
	if err != nil {
		panic(err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	r := gin.Default()
	r.GET("/pods", func(c *gin.Context) {
		list, err := clientset.CoreV1().Pods("").List(context.TODO(), metav1.ListOptions{})
		if err != nil {
			panic(err)
			c.JSON(http.StatusBadRequest, gin.H{"message": err, "code": 400})
		}
		c.JSON(http.StatusOK, gin.H{"message": 200, "result": list})
	})
	_ = r.Run(":8080")
}
```

### 测试结果对比:
-




