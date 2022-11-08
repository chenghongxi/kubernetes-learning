# redis-operators

## Install:


```text
1. kubectl create -f https://operatorhub.io/install/redis-operator.yaml
```
![img](img/redis-operators.png)
[redis-operator.yaml](yml/redis-operator.yaml)

```text
2. kubectl get csv -n operators
```
![img](img/csv.png)

```text
3. kubectl create -f ./storage-class.yml
```
![img](img/storage.png)
![img](img/storage2.png)
[storage-class.yml](yml/storage-class.yml)

```shell
4. kubectl create -f local-pv.yml
```
![img](img/222.png)
[local-pv.yml](yml/local-pv.yaml)


```text
5. kubectl create -f redis-single.yml
```
![img](img/redis-po.png)
[redis-single](yml/redis-single.yml)

## Validation
```text
1. kubectl exec -it redis-standalone-0 -- /bin/bash
2. redis-cli -c
3. set k1 v1
4. get k1
```
![img](img/exec-redis.png)



