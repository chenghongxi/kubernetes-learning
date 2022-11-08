# mongodb-operators

## Install:
```text
1. kubectl create -f https://operatorhub.io/install/mongodb-operator.yaml
```
![img](img/mongodb-operator.png)


```shell
2. kubectl get csv -n operators
```
![img](img/csv.png)


```shell
3. kubectl create -f storage-class.yml
```
![img](img/storage-class.png)


[storage-class.yml](yml/storage-class.yaml)

```shell
4. kubectl create -f local-pv.yml
```
![img](img/local-pv.png)


[local-pv.yml](yml/local-pv.yaml)


```shell
5. kubectl create -f mongodb-secret.yml 
```
![img](img/storage-class.png)


[mongodb-secret.yml](yml/mongodb-secret.yml)


```shell
6. kubectl create -f Mongodb.yaml
```
![img](img/create.png)


[Mongodb.yaml](yml/Mongodb.yaml)

## Validation

```shell
1. kubectl exec -it mongodb-standalone-0 -- /bin/bash
2. mongo
3. use pixiuDB
```
![img](img/conn_mongo.png)

