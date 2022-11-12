# Postgres-Operator

## Documentation
https://postgres-operator.readthedocs.io/en/latest/

## Github
https://github.com/zalando/postgres-operator

## Rely On
- `Kubernetes Cluster Version > 1.14.0`
- `Kubernetes Cluster Version < 1.21`

## Install:

```shell
1. kubectl create -f https://operatorhub.io/install/postgres-operator.yaml
```
![img](img/postgres-Operator.png)

[postgres-operator.yaml](yml/postgres-operator.yaml)

```shell
1. kubectl get csv -n operators
```
![img](img/operator.png)



