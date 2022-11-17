# Postgres-Operator

## Documentation
https://access.crunchydata.com/documentation/postgres-operator/v5/

## Github
https://github.com/CrunchyData/postgres-operator

## Rely On
- `Kubernetes Cluster Version > 1.19.0`

## Install:

```shell
1. kubectl apply -f https://operatorhub.io/install/postgresql.yaml
```
![img](picture/postgres-Operator.png)


[postgresql.yaml](https://operatorhub.io/install/postgresql.yaml)

```shell
2. kubectl get csv -n operators
```
![img](picture/operator.png)

```shell
3. kubectl apply -f https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/postgres-Operators/yml/create-postgres-cluster.yaml
```
![img](picture/create-postgres-cluster.png)

[create-postgres-cluster.yaml](https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/postgres-Operators/yml/create-postgres-cluster.yaml)




## Validation
```shell
1. kubectl get sts,sc,pv,secret
```
`hippo-instance1-gfb5-0` `pod` 中会创建几个容器，分别为:  `database`, `replication-cert-copy`, `pgbackrest`, `pgbackrest-config`,`postgres-startup (init)`, `nss-wrapper-init (init)`

`hippo-repo-host-0` `pod` 中会创建几个容器，分别为:`pgbackrest`, `pgbackrest-config`, `pgbackrest-log-dir (init)`, `nss-wrapper-init (init)`

![img](picture/validation.png)
```text
1. kubectl exec -it hippo-instance1-gfb5-0 -- /bin/bash
2. psql
```

![img](picture/validation2.png)

## UnInstall
- `删除步骤 3 中的资源`
```shell
kubectl delete -f https://raw.githubusercontent.com/chenghongxi/kubernetes-learning/master/olm/postgres-Operators/yml/create-postgres-cluster.yaml
```
- `删除此 Operator`
```shell
1. kubectl delete subscription <subscription-name> -n operators
2. kubectl delete clusterserviceversion -n operators
```







