# Rabbitmq-Operators

部署和管理RabbitMQ集群 , 为 RabbitMQ 集群的生命周期（创建、升级、正常关闭）设计的自定义控制器和自定义资源定义 ( CRD )

## Documentation
https://www.rabbitmq.com/kubernetes/operator/operator-overview.html

## GitHub
https://github.com/rabbitmq/cluster-operator


## Rely on
- `Kubernetes 1.19.0 cluster`
- 集群已安装 `OLM` 组件。安装手册参考 [OLM安装](../README.md)

## Install:


```shell
1. kubectl apply -f https://operatorhub.io/install/rabbitmq-cluster-operator.yaml
```
![img](picture/rabbitmq-cluster-operator.png)

#### `安装过程解析`
- 原理同 `redis-operator`

- [redis-operator](https://github.com/chenghongxi/kubernetes-learning/blob/master/olm/redis-operators/README.md#%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B%E8%A7%A3%E6%9E%90)





```shell
2. kubectl get csv -n operators
```
![img](picture/csv.png)

```shell
3. kubectl create -f https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/rabbitmq-operators/yml/create-rabbitmq-cluster.yaml
```
![img](picture/create-rabbitmq-cluster.png)

[create-rabbitmq-cluster.yaml](https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/rabbitmq-operators/yml/create-rabbitmq-cluster.yaml)

## Validation
```shell
1. kubectl get po,sc,pv,pvc,secret
```
![img](picture/validation.png)
## UnInstall
- `删除步骤 3 中的资源`
```shell
kubectl delete -f https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/rabbitmq-operators/yml/create-rabbitmq-cluster.yaml
```
- `删除此 Operator`
```shell
1. kubectl delete subscription <subscription-name> -n operators
2. kubectl delete clusterserviceversion -n operators
```