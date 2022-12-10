# Controller-Manager

## 介绍:
在`Kubernetes`中 `Controller-Manager` 作为集群内部的管理控制中心, 负责集群内的`Node`,`pod`,`Endpoint`,`service`,`namespace`等资源的管理,这些资源各有一个`controller`来负责。

每个`Controller`通过 `Api-server`提供的接口监控整个集群的每个资源对象的当前状态，当发生变化时，会尝试将状态恢复到`期望状态`。

## 

