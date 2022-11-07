# Helm

## Helm是什么？
Helm 是一个用于 Kubernetes 应用的包管理工具，主要用来管理 Charts。
有点类似于 Ubuntu 中的 APT 或 CentOS 中的 YUM。

## 组件:
- `Chart：Chart 是一个 Helm 的软件包，采用 tar 格式。包含了运行一个K8S应用程序所需的镜像、依赖关系和资源定义等，也就是包含了一个应用所需资源对象的YAML文件。`
- `Repository（仓库）：是 Helm 的软件仓库，Repository 本质上是一个 Web 服务器,该服务器保存了一系列的 Chart 软件包供用户下载，并且提供了该 Repository 的 Chart 包的清单文件便于查询。`
- `Config（配置数据）：部署时设置到 Chart 中的配置数据。`
- `Release：使用 helm install 命令在 k8s 集群中部署的 Chart 称为 Release 。是基于 Chart 和Config 部署到 K8S 集群中运行的一个实例。一个Chart可以被部署多次，每次的 Release 都不相同。`

## Install:
```shell
wget https://mirrors.huaweicloud.com/helm/v3.3.1/helm-v3.3.1-linux-amd64.tar.gz      
 
tar -zxvf helm-v3.3.1-linux-amd64.tar.gz
 
cp linux-amd64/helm /usr/local/bin/
 
helm version
```
