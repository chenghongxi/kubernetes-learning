# Operator Lifecycle Manager(OLM)

官网：https://docs.openshift.com/container-platform/3.11/install_config/installing-operator-framework.html

## OLM
```text
OLM 扩展了Kubernetes，提供了一种陈述式的方式来安装，管理和升级Operator，以及它在集群中所依赖的资源。
它还对其管理的组件强制执行一些约束，以确保良好的用户体验。
该项目使用户能够执行以下操作：
```
- `1. 将应用程序定义为封装了需求和元数据的一个Kubernetes资源`
- `2. 使用依赖项解析方式自动地安装应用程序，或者使用kubectl/oc(OpenShift Client)手动安装应用程序`
- `3. 使用不同的批准策略自动升级应用程序`

