# Operator Lifecycle Manager(OLM)

- `Website`: https://docs.openshift.com/container-platform/3.11/install_config/installing-operator-framework.html
- `GitHub`: https://github.com/operator-framework/operator-lifecycle-manager.git
- `Operatorhub`: https://operatorhub.io/

## Rely on
- `kubectl v1.11.3+`
- `Kubernetes v1.11.3+ cluster`

## Environment version
1. 机器配置：4核，8G
2. Kubernetes Version: v1.23.6
3. 集群安装方法: https://github.com/caoyingjunz/kubez-ansible
## 组件介绍:
```text
OLM(Operator Lifecycle Manager) 作为 Operator Framework 的一部分，可以帮助用户进行 Operator 的自动安装，
升级及其生命周期的管理。同时 OLM 自身也是以 Operator 的形式进行安装部署，可以说它的工作方式是以 Operators 来管理 Operators，
而它面向 Operator 提供了声明式 (declarative) 的自动化管理能力也完全符合 Kubernetes 交互的设计理念。
```

## 组件原理:
```text
OLM 由两个 Operator 构成：OLM Operator 和 Catalog Operator，其分别管理以下几个基础 CRD 模型：
```

![img.png](img/1.jpg)

## 使用场景:
OLM可以帮助用户，安装，更新，和管理所有Operator的生命周期


### Install:
- `Scripted`
```shell
curl -L https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.19.1/install.sh -o install.sh
chmod +x install.sh
./install.sh v0.19.1
```

![img](img/install.png)

##### `安装过程解析`

[crds.yaml](yml/crds.yaml)

[olm.yml](yml/olm.yaml)

- `Operator-sdk`

请查看官方快速开始文档
```text
https://olm.operatorframework.io/docs/getting-started/
```

#### Validation:
```text
kubectl get ns
```
![img](img/ns.png)
```text
kubectl -n olm get deployments
```
![img](img/deploy.png)

## Application

[Redis-Operators](redis-operators/README.md)

[MongoDB-Operators](mongodb-operators/README.md)

[Rabbitmq-Operators](rabbitmq-operators/README.md)

[Postgres-Operators](postgres-Operators/README.md)

## UnInstall
```shell
1. kubectl delete -f olm.yaml
2. kubectl delete -f crds.yaml
```

