# Postgres-Operator

## Documentation
https://access.crunchydata.com/documentation/postgres-operator/v5/

## Github
https://github.com/CrunchyData/postgres-operator

## Rely On
- `Kubernetes Cluster Version > 1.19.0`

## Install:

```shell
1. kubectl create -f https://operatorhub.io/install/postgresql.yaml
```
![img](picture/postgres-Operator.png)


[postgres-operator.yaml](yml/postgres-operator.yaml)

```shell
2. kubectl get csv -n operators
```
![img](picture/operator.png)

```shell
3. kubectl create -f sc.yml
4. kubectl create -f sc1.yml
```
![img](picture/sc.png)

[sc.yml](yml/sc.yml)

[sc1.yml](yml/sc1.yml)

```shell
5. kubectl create -f local-pv.yml
6. kubectl create -f local-pv1.yml
```
![img](picture/pv.png)

[local-pv.yml](yml/local-pv.yml)

[local-pv1.yml](yml/local-pv1.yml)

```shell
7. kubectl create -f Postgres-cluster.yml
```
`hippo-instance1-gfb5-0` `pod` 中会创建几个容器，分别为:  `database`, `replication-cert-copy`, `pgbackrest`, `pgbackrest-config`,`postgres-startup (init)`, `nss-wrapper-init (init)`

`hippo-repo-host-0` `pod` 中会创建几个容器，分别为:`pgbackrest`, `pgbackrest-config`, `pgbackrest-log-dir (init)`, `nss-wrapper-init (init)`

![img](picture/po.png)

[Postgres-cluster.yml](yml/Postgres-cluster.yml)

## Validation
```text
1. kubectl exec -it hippo-instance1-gfb5-0 -- /bin/bash
2. psql
```

![img](picture/validation.png)







