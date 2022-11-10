# apiserver源码分析(启动流程)

## 介绍:

github: https://github.com/kubernetes/kubernetes

Apiserver是Kubernetes的一个重要组件，在所有组件中是唯一一个对接Etcd,Kubernetes中各种资源的增删改查服务都是使用http服务的形式

## 组件:

Apiserver组件中有3个server,apiserver依靠这3个server来处理不同的请求内容,具体如下:

- `KubeAPIServer: 主要负责处理k8s内置资源的请求，此外还会包括通用处理，认证、鉴权等`
- `APIExtensionServer: 主要负责处理CustomResourceDefination（CRD）方面的请求`
- `AggregratorServer: 主要负责aggregrate方面的处理，它充当一个代理服务器，将请求转发到聚合进来的k8s service中`

apiserver组件与其他组件(如:kubelet等)使用了corbra命令行框架处理启动命令,它从命令行的RunE回调函数-->Run函数，开始执行启动流程。

RunE回调函数: https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L133

```go
func NewAPIServerCommand() *cobra.Command {
    s := options.NewServerRunOptions()
    cmd := &cobra.Command{
    Use: "kube-apiserver",
    Long: `The Kubernetes API server validates and configures data
for the api objects which include pods, services, replicationcontrollers, and
others. The API Server services REST operations and provides the frontend to the
cluster's shared state through which all other components interact.`,

    // stop printing usage when the command errors
    SilenceUsage: true,
    PersistentPreRunE: func (*cobra.Command, []string) error {
    // silence client-go warnings.
    // kube-apiserver loopback clients should not log self-issued warnings.
        rest.SetDefaultWarningHandler(rest.NoWarnings{})
        return nil
	},
    RunE: func (cmd *cobra.Command, args []string) error {
        verflag.PrintAndExitIfRequested()
        fs := cmd.Flags()

         // Activate logging as soon as possible, after that
         // show flags with the final logging configuration.
        if err := logsapi.ValidateAndApply(s.Logs, utilfeature.DefaultFeatureGate); err != nil {
            return err  
		}
        cliflag.PrintFlags(fs)

        // set default options 
		completedOptions, err := Complete(s)
		if err != nil {
        return err
        }

        // validate options
        if errs := completedOptions.Validate(); len(errs) != 0 {
            return utilerrors.NewAggregate(errs)
         }
         // add feature enablement metrics
         utilfeature.DefaultMutableFeatureGate.AddMetrics()
		    return Run(completedOptions, genericapiserver.SetupSignalHandler())
         },
    Args: func (cmd *cobra.Command, args []string) error {
            for _, arg := range args {
                if len(arg) > 0 {
                    return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
                }
            }
            return nil
	},
}

    fs := cmd.Flags()
    namedFlagSets := s.Flags()
    verflag.AddFlags(namedFlagSets.FlagSet("global"))
    globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name(), logs.SkipLoggingConfigurationFlags())
    options.AddCustomGlobalFlags(namedFlagSets.FlagSet("generic"))
    for _, f := range namedFlagSets.FlagSets {
        fs.AddFlagSet(f)
    }

    cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
    cliflag.SetUsageAndHelpFunc(cmd, namedFlagSets, cols)

    return cmd
}
```

Run函数: https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L161
        

```go
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
    // To help debugging, immediately log version
    klog.Infof("Version: %+v", version.Get())

    klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

    // 注册了三个server,分别为:
    // 1. KubeAPIServer
    // 2. APIExtensionServer
    // 3. AggregratorServer
    // CreateServerChain函数为具体的注册
    server, err := CreateServerChain(completeOptions)
    if err != nil {
    return err
        }

    // 为每个server注册健康检查,就绪,存活探针
    prepared, err := server.PrepareRun()
    if err != nil {
        return err
    }

    return prepared.Run(stopCh)
}
```
三个server的创建流程: server的创建都有相对应的config，每个config都需要依赖GenericConfig,函数结束后返回一个APIAggregator(api聚合器)
- `KubeAPIServer`
- `APIExtensionServer`
- `AggregratorServer`

CreateServerChain函数: https://github.com/kubernetes/kubernetes/blob/f87231003aeb43bf884cf9dc10b1247b8ae5cbb8/cmd/kube-apiserver/app/server.go#L181

```go
func CreateServerChain(completedOptions completedServerRunOptions) (*aggregatorapiserver.APIAggregator, error) {
    kubeAPIServerConfig, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions)
    if err != nil {
        return nil, err
	}

    // If additional API servers are added, they should be gated.
    apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
    serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(kubeAPIServerConfig.ExtraConfig.ProxyTransport, kubeAPIServerConfig.GenericConfig.EgressSelector, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig, kubeAPIServerConfig.GenericConfig.TracerProvider))
    if err != nil {
        return nil, err
    }

    notFoundHandler := notfoundhandler.New(kubeAPIServerConfig.GenericConfig.Serializer, genericapifilters.NoMuxAndDiscoveryIncompleteKey)
	// 创建APIExtensionsServer并注册路由
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
    if err != nil {
        return nil, err
	}
    // 创建KubeAPIServer并注册路由
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
    if err != nil {
        return nil, err
    }

    // aggregator comes last in the chain
    aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, kubeAPIServerConfig.ExtraConfig.ProxyTransport, pluginInitializer)
    if err != nil {
        return nil, err
    }
    // 创建aggregatorServer并注册路由
    aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
    if err != nil {
    // we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
        return nil, err
    }

    return aggregatorServer, nil
}
```
CreateServerChain函数解析:

- `1. 通过 CreateKubeAPIServerConfig函数 返回 kubeAPIServerConfig`

- `2. 每个server在创建时需要对应的config,函数结束时，返回APIAggregator`

- `3. 完成server注册`







