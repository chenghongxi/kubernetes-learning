# Kubelet 
## **概要**: 

kubelet 是运行在每个节点上的主要的“节点代理”，每个节点都会启动 kubelet进程，用来处理 Master 节点下发到本节点的任务，
按照 PodSpec 描述来管理Pod 和其中的容器（PodSpec 是用来描述一个 pod 的 YAML 或者 JSON 对象）。
kubelet 通过各种机制（主要通过 apiserver ）获取一组 PodSpec 并保证在这些 PodSpec 中描述的容器健康运行。
## Kubenetes version: 

## 源码:
- `cmd/kubelet/kubectl.go`    // cmd/kubelet/kubectl.go 为 main 函数入口，主要负责参数加载，运行框架等前期准备工作

- `pkg/kubelet/kubectl.go`    // pkg/kubelet/kubectl.go 主要负责功能上的逻辑代码

### 目录结构: 


  
