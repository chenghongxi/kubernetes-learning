# Controller-Manager

## 介绍:

Github: https://github.com/kubernetes/kubernetes

在`Kubernetes`中 `Controller-Manager` 作为集群内部的管理控制中心, 负责集群内的`Node`,`pod`,`Endpoint`,`service`,`namespace`等资源的管理,这些资源各有一个`controller`来负责。

每个`Controller`通过 `Api-server`提供的接口监控整个集群的每个资源对象的当前状态，当发生变化时，会尝试将状态恢复到`期望状态`。

在源码中，具体可以看到有多少的 Controller:

- `具体链接如下:`
[Controller](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller)
- `从代码中`:
```go
        register("endpoint", startEndpointController)
	register("endpointslice", startEndpointSliceController)
	register("endpointslicemirroring", startEndpointSliceMirroringController)
        ## 副本 Controller
	register("replicationcontroller", startReplicationController)
	register("podgc", startPodGCController)
	register("resourcequota", startResourceQuotaController)
        ## 命名空间 Controller
	register("namespace", startNamespaceController 
	register("serviceaccount", startServiceAccountController)
	register("garbagecollector", startGarbageCollectorController)
	register("daemonset", startDaemonSetController)
	## job Controller
	register("job", startJobController)
	## 无状态 Controller
	register("deployment", startDeploymentController)
	register("replicaset", startR   eplicaSetController)
	register("horizontalpodautoscaling", startHPAController)
	register("disruption", startDisruptionController)
	register("statefulset", startStatefulSetController)
	register("cronjob", startCronJobController)
	register("csrsigning", startCSRSigningController)
	register("csrapproving", startCSRApprovingController)
	register("csrcleaner", startCSRCleanerController)
	register("ttl", startTTLController)
	register("bootstrapsigner", startBootstrapSignerController)
	## token Controller
	register("tokencleaner", startTokenCleanerController)
	register("nodeipam", startNodeIpamController)
	register("nodelifecycle", startNodeLifecycleController)
	if loopMode == IncludeCloudLoops {
		register("service", startServiceController)
		register("route", startRouteController)
		register("cloud-node-lifecycle", startCloudNodeLifecycleController)
		// TODO: volume controller into the IncludeCloudLoops only set.
	}
	register("persistentvolume-binder", startPersistentVolumeBinderController)
	register("attachdetach", startAttachDetachController)
	register("persistentvolume-expander", startVolumeExpandController)
	register("clusterrole-aggregation", startClusterRoleAggregrationController)
	register("pvc-protection", startPVCProtectionController)
	register("pv-protection", startPVProtectionController)
	register("ttl-after-finished", startTTLAfterFinishedController)
	register("root-ca-cert-publisher", startRootCACertPublisher)
	register("ephemeral-volume", startEphemeralVolumeController)
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) {
		register("storage-version-gc", startStorageVersionGCController)
	}
```




