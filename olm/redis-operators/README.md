# redis-operators

## Supported redis versions:

- `=> v6`
- `=> v7`

## Documentation
https://ot-container-kit.github.io/redis-operator/

## GitHub
https://github.com/ot-container-kit/redis-operator

## Rely on
- `Kubernetes 1.18.0 cluster`

## Install:


```shell
1. kubectl create -f https://operatorhub.io/install/redis-operator.yaml
```
![img](img/redis-operators.png)


[redis-operator.yaml](https://operatorhub.io/install/redis-operator.yaml)

```shell
2. kubectl get csv -n operators
```
![img](img/csv.png)

```shell
3. kubectl apply -f https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/redis-operators/yml/create-redis-cluster.yaml
```
![img](img/create-redis-cluster.png)


[create-redis-cluster.yaml](https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/redis-operators/yml/create-redis-cluster.yaml)


## Validation
```shell
1. kubectl get sc,pv,pvc,po
```
![img](img/get-redis-cluster.png)
```text
2. kubectl exec -it redis-standalone-0 -- /bin/bash
3. redis-cli -c
4. set k1 v1
5. get k1
```
![img](img/exec-redis.png)

## UnInstall
```shell
kubectl delete -f https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/redis-operators/yml/create-redis-cluster.yaml
```



