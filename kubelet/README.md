# Kubelet--启动篇
## **概要**: 

kubelet 是运行在每个节点上的主要的“节点代理”，每个节点都会启动 kubelet进程，用来处理 Master 节点下发到本节点的任务，
按照 PodSpec 描述来管理Pod 和其中的容器（PodSpec 是用来描述一个 pod 的 YAML 或者 JSON 对象）。
kubelet 通过各种机制（主要通过 apiserver ）获取一组 PodSpec 并保证在这些 PodSpec 中描述的容器健康运行。

## Kubenetes version: 

## 源码:
- `cmd/kubelet/kubectl.go`    // cmd/kubelet/kubectl.go 为 main 函数入口，主要负责参数加载，运行框架等前期准备工作

- `pkg/kubelet/kubectl.go`    // pkg/kubelet/kubectl.go 主要负责功能上的逻辑代码

### 目录结构: 
![img](picture/cmd-kubelet.png)
![img](picture/pkg-kubelet.png)

### cmd/kubelet/kubectl.go 源码分析:

1. kubelet 程序主入口, 使用了 corbra 命令行框架处理启动命令,它从命令行的`RunE`回调函数-->`Run`函数，开始执行启动流程。

   //  cobra使用问题请移步: https://github.com/chenghongxi/go-learning/tree/main/cobra


代码: https://github.com/kubernetes/kubernetes/blob/master/cmd/kubelet/kubelet.go#L34
```text
func main() {
	command := app.NewKubeletCommand()
	code := cli.Run(command)
	os.Exit(code)
}
```

2. `NewKubeletCommand` 函数主要作用，从命令行的`RunE`回调函数-->`Run`函数，开始启动流程流程,从下面的代码中可以看到,`NewKubeletCommand` 函数是做前期的工作准备，
比如：参数解析，校验，构建依赖，构建server,通过这些参数运行Kubelet,进入Run函数:

代码: https://github.com/kubernetes/kubernetes/blob/cf12a74b18b66efc577ec819c78a0c68f6d49225/cmd/kubelet/app/server.go#L124
```text
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags()

	kubeletConfig, err := options.NewKubeletConfiguration()
	// 错误
	if err != nil {
		klog.ErrorS(err, "Failed to create a new kubelet configuration")
		os.Exit(1)
	}

	cmd := &cobra.Command{
		Use: componentKubelet,
		Long: `The kubelet is the primary "node agent" that runs on each
node. It can register the node with the apiserver using one of: the hostname; a flag to
override the hostname; or specific logic for a cloud provider.
The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object
that describes a pod. The kubelet takes a set of PodSpecs that are provided through
various mechanisms (primarily through the apiserver) and ensures that the containers
described in those PodSpecs are running and healthy. The kubelet doesn't manage
containers which were not created by Kubernetes.
Other than from an PodSpec from the apiserver, there are two ways that a container
manifest can be provided to the Kubelet.
File: Path passed as a flag on the command line. Files under this path will be monitored
periodically for updates. The monitoring period is 20s by default and is configurable
via a flag.
HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).`,
		DisableFlagParsing: true,
		SilenceUsage:       true,
		RunE: func(cmd *cobra.Command, args []string) error {
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				return fmt.Errorf("failed to parse kubelet flag: %w", err)
			}


			// 检查是否有命令未传入参数，若是，则返回错误和使用指引
			cmds := cleanFlagSet.Args()
			if len(cmds) > 0 {
				return fmt.Errorf("unknown command %+s", cmds[0])
			}

			// short-circuit on help
			// 检查是否设置了帮助指引
			help, err := cleanFlagSet.GetBool("help")
			if err != nil {
				return errors.New(`"help" flag is non-bool, programmer error, please correct`)
			}
			if help {
				return cmd.Help()
			}


			// 若是查看版本信息，则返回版本信息及退出
			verflag.PrintAndExitIfRequested()

			// set feature gates from initial flags-based config
			if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
				return fmt.Errorf("failed to set feature gates from initial flags-based config: %w", err)
			}


			// 校验kubelet flag相关参数传入是否合法，如果合法则初始化到对应变量
			if err := options.ValidateKubeletFlags(kubeletFlags); err != nil {
				return fmt.Errorf("failed to validate kubelet flags: %w", err)
			}

			if cleanFlagSet.Changed("pod-infra-container-image") {
				klog.InfoS("--pod-infra-container-image will not be pruned by the image garbage collector in kubelet and should also be set in the remote runtime")
			}

	
			// 检测并解析kubelet配置文件
			if configFile := kubeletFlags.KubeletConfigFile; len(configFile) > 0 {
				kubeletConfig, err = loadConfigFile(configFile)
				if err != nil {
					return fmt.Errorf("failed to load kubelet config file, error: %w, path: %s", err, configFile)
				}

				if err := kubeletConfigFlagPrecedence(kubeletConfig, args); err != nil {
					return fmt.Errorf("failed to precedence kubeletConfigFlag: %w", err)
				}

				if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
					return fmt.Errorf("failed to set feature gates from initial flags-based config: %w", err)
				}
			}

			// 初始化日志
			logs.InitLogs()
			if err := logsapi.ValidateAndApplyAsField(&kubeletConfig.Logging, utilfeature.DefaultFeatureGate, field.NewPath("logging")); err != nil {
				return fmt.Errorf("initialize logging: %v", err)
			}
			//若是查看参数使用，则输出所有参数说明
			cliflag.PrintFlags(cleanFlagSet)

			// 本地命令行+配置文件验证
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig, utilfeature.DefaultFeatureGate); err != nil {
				return fmt.Errorf("failed to validate kubelet configuration, error: %w, path: %s", err, kubeletConfig)
			}

			if (kubeletConfig.KubeletCgroups != "" && kubeletConfig.KubeReservedCgroup != "") && (strings.Index(kubeletConfig.KubeletCgroups, kubeletConfig.KubeReservedCgroup) != 0) {
				klog.InfoS("unsupported configuration:KubeletCgroups is not within KubeReservedCgroup")
			}

			// construct a KubeletServer from kubeletFlags and kubeletConfig
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}


			// 通过kubelet server和DefaultFeatureGate，构建kubeletDeps，
			// kubeletDeps主要集成了kubelet的运行依赖如：认证信息、第三方云厂商信息、事件记录、挂载、网络/卷插件、OOM管理等依赖组件调用链操作句柄
			kubeletDeps, err := UnsecuredDependencies(kubeletServer, utilfeature.DefaultFeatureGate)
			if err != nil {
				return fmt.Errorf("failed to construct kubelet dependencies: %w", err)
			}

			if err := checkPermissions(); err != nil {
				klog.ErrorS(err, "kubelet running with insufficient permissions")
			}

			config := kubeletServer.KubeletConfiguration.DeepCopy()
			for k := range config.StaticPodURLHeader {
				config.StaticPodURLHeader[k] = []string{"<masked>"}
			}
			// log the kubelet's config for inspection
			klog.V(5).InfoS("KubeletConfiguration", "configuration", config)

			// 设置kubelet关闭的信号环境
			ctx := genericapiserver.SetupSignalContext()

			utilfeature.DefaultMutableFeatureGate.AddMetrics()
			
			// 启动kubelet，把ctx、kubeletServer、kubeletDeps、utilfeature.DefaultFeatureGate这些准备好的参数传入，
			// 执行内层核心Run函数
			return Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate)
		},
	}

	kubeletFlags.AddFlags(cleanFlagSet)
	options.AddKubeletConfigFlags(cleanFlagSet, kubeletConfig)
	options.AddGlobalFlags(cleanFlagSet)
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}
```

3. `Run` 函数主要工作: 
- 打印kubelet
- golang设置
- 初始化OS
- 真正run kubelet
代码: https://github.com/kubernetes/kubernetes/blob/cf12a74b18b66efc577ec819c78a0c68f6d49225/cmd/kubelet/app/server.go#L410
```text
func Run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) error {
	// 为了帮助调试，记录版本
	klog.InfoS("Kubelet version", "kubeletVersion", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	if err := initForOS(s.KubeletFlags.WindowsService, s.KubeletFlags.WindowsPriorityClass); err != nil {
		return fmt.Errorf("failed OS init: %w", err)
	}
	// 
	if err := run(ctx, s, kubeDeps, featureGate); err != nil {
		return fmt.Errorf("failed to run Kubelet: %w", err)
	}
	return nil
}
```
4. `run` 函数主要流程：
- 设置全局kubeletServer
- 验证初始化的 kubeletServer
- 将当前配置注册到config
- 获取客户端，检测 standaloneMode
- 如果处于独立模式，则将所有客户端设置为nil
- 设置 event 记录器
- 通知systemd,kubelet已经启动

代码: https://github.com/kubernetes/kubernetes/blob/cf12a74b18b66efc577ec819c78a0c68f6d49225/cmd/kubelet/app/server.go#L489
```text
func run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) (err error) {
	// 设置全局kubeletServer
	err = utilfeature.DefaultMutableFeatureGate.SetFromMap(s.KubeletConfiguration.FeatureGates)
	if err != nil {
		return err
	}
	// 开始验证初始化的 kubeletServer
	if err := options.ValidateKubeletServer(s); err != nil {
		return err
	}

	// 判断 cgroups v1是否启动了 MemoryQoS
	if utilfeature.DefaultFeatureGate.Enabled(features.MemoryQoS) &&
		!isCgroup2UnifiedMode() {
		klog.InfoS("Warning: MemoryQoS feature only works with cgroups v2 on Linux, but enabled with cgroups v1")
	}
	// 获取 kubelet 锁文件
	if s.ExitOnLockContention && s.LockFilePath == "" {
		return errors.New("cannot exit on lock file contention: no lock file specified")
	}
	done := make(chan struct{})
	if s.LockFilePath != "" {
		klog.InfoS("Acquiring file lock", "path", s.LockFilePath)
		if err := flock.Acquire(s.LockFilePath); err != nil {
			return fmt.Errorf("unable to acquire file lock on %q: %w", s.LockFilePath, err)
		}
		if s.ExitOnLockContention {
			klog.InfoS("Watching for inotify events", "path", s.LockFilePath)
			if err := watchForLockfileContention(s.LockFilePath, done); err != nil {
				return err
			}
		}
	}

	// 将当前配置注册到 /configz endpoint
	err = initConfigz(&s.KubeletConfiguration)
	if err != nil {
		klog.ErrorS(err, "Failed to register kubelet configuration with configz")
	}

	if len(s.ShowHiddenMetricsForVersion) > 0 {
		metrics.SetShowHidden()
	}

	// 获取客户端，检测 standaloneMode
	standaloneMode := true
	if len(s.KubeConfig) > 0 {
		standaloneMode = false
	}

	if kubeDeps == nil {
		kubeDeps, err = UnsecuredDependencies(s, featureGate)
		if err != nil {
			return err
		}
	}

	if kubeDeps.Cloud == nil {
		if !cloudprovider.IsExternal(s.CloudProvider) {
			cloudprovider.DeprecationWarningForProvider(s.CloudProvider)
			cloud, err := cloudprovider.InitCloudProvider(s.CloudProvider, s.CloudConfigFile)
			if err != nil {
				return err
			}
			if cloud != nil {
				klog.V(2).InfoS("Successfully initialized cloud provider", "cloudProvider", s.CloudProvider, "cloudConfigFile", s.CloudConfigFile)
			}
			kubeDeps.Cloud = cloud
		}
	}

	hostName, err := nodeutil.GetHostname(s.HostnameOverride)
	if err != nil {
		return err
	}
	nodeName, err := getNodeName(kubeDeps.Cloud, hostName)
	if err != nil {
		return err
	}

	// 如果处于独立模式，则将所有客户端设置为nil
	switch {
	case standaloneMode:
		kubeDeps.KubeClient = nil
		kubeDeps.EventClient = nil
		kubeDeps.HeartbeatClient = nil
		klog.InfoS("Standalone mode, no API client")

	case kubeDeps.KubeClient == nil, kubeDeps.EventClient == nil, kubeDeps.HeartbeatClient == nil:
		clientConfig, onHeartbeatFailure, err := buildKubeletClientConfig(ctx, s, kubeDeps.TracerProvider, nodeName)
		if err != nil {
			return err
		}
		if onHeartbeatFailure == nil {
			return errors.New("onHeartbeatFailure must be a valid function other than nil")
		}
		kubeDeps.OnHeartbeatFailure = onHeartbeatFailure

		kubeDeps.KubeClient, err = clientset.NewForConfig(clientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet client: %w", err)
		}

		// 为事件创建单独的客户端
		eventClientConfig := *clientConfig
		eventClientConfig.QPS = float32(s.EventRecordQPS)
		eventClientConfig.Burst = int(s.EventBurst)
		kubeDeps.EventClient, err = v1core.NewForConfig(&eventClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet event client: %w", err)
		}

		// 禁用节流，附加超时的心跳创建一个单独的客户机
		heartbeatClientConfig := *clientConfig
		heartbeatClientConfig.Timeout = s.KubeletConfiguration.NodeStatusUpdateFrequency.Duration
		// The timeout is the minimum of the lease duration and status update frequency
		leaseTimeout := time.Duration(s.KubeletConfiguration.NodeLeaseDurationSeconds) * time.Second
		if heartbeatClientConfig.Timeout > leaseTimeout {
			heartbeatClientConfig.Timeout = leaseTimeout
		}

		heartbeatClientConfig.QPS = float32(-1)
		kubeDeps.HeartbeatClient, err = clientset.NewForConfig(&heartbeatClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet heartbeat client: %w", err)
		}
	}

	if kubeDeps.Auth == nil {
		auth, runAuthenticatorCAReload, err := BuildAuth(nodeName, kubeDeps.KubeClient, s.KubeletConfiguration)
		if err != nil {
			return err
		}
		kubeDeps.Auth = auth
		runAuthenticatorCAReload(ctx.Done())
	}

	var cgroupRoots []string
	nodeAllocatableRoot := cm.NodeAllocatableRoot(s.CgroupRoot, s.CgroupsPerQOS, s.CgroupDriver)
	cgroupRoots = append(cgroupRoots, nodeAllocatableRoot)
	kubeletCgroup, err := cm.GetKubeletContainer(s.KubeletCgroups)
	if err != nil {
		klog.InfoS("Failed to get the kubelet's cgroup. Kubelet system container metrics may be missing.", "err", err)
	} else if kubeletCgroup != "" {
		cgroupRoots = append(cgroupRoots, kubeletCgroup)
	}

	if s.RuntimeCgroups != "" {
		// RuntimeCgroups is optional, so ignore if it isn't specified
		cgroupRoots = append(cgroupRoots, s.RuntimeCgroups)
	}

	if s.SystemCgroups != "" {
		// SystemCgroups is optional, so ignore if it isn't specified
		cgroupRoots = append(cgroupRoots, s.SystemCgroups)
	}

	if kubeDeps.CAdvisorInterface == nil {
		imageFsInfoProvider := cadvisor.NewImageFsInfoProvider(s.RemoteRuntimeEndpoint)
		kubeDeps.CAdvisorInterface, err = cadvisor.New(imageFsInfoProvider, s.RootDirectory, cgroupRoots, cadvisor.UsingLegacyCadvisorStats(s.RemoteRuntimeEndpoint), s.LocalStorageCapacityIsolation)
		if err != nil {
			return err
		}
	}

	// 设置 event 记录器
	makeEventRecorder(kubeDeps, nodeName)

	if kubeDeps.ContainerManager == nil {
		if s.CgroupsPerQOS && s.CgroupRoot == "" {
			klog.InfoS("--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /")
			s.CgroupRoot = "/"
		}

		machineInfo, err := kubeDeps.CAdvisorInterface.MachineInfo()
		if err != nil {
			return err
		}
		reservedSystemCPUs, err := getReservedCPUs(machineInfo, s.ReservedSystemCPUs)
		if err != nil {
			return err
		}
		if reservedSystemCPUs.Size() > 0 {
			// at cmd option validation phase it is tested either --system-reserved-cgroup or --kube-reserved-cgroup is specified, so overwrite should be ok
			klog.InfoS("Option --reserved-cpus is specified, it will overwrite the cpu setting in KubeReserved and SystemReserved", "kubeReservedCPUs", s.KubeReserved, "systemReservedCPUs", s.SystemReserved)
			if s.KubeReserved != nil {
				delete(s.KubeReserved, "cpu")
			}
			if s.SystemReserved == nil {
				s.SystemReserved = make(map[string]string)
			}
			s.SystemReserved["cpu"] = strconv.Itoa(reservedSystemCPUs.Size())
			klog.InfoS("After cpu setting is overwritten", "kubeReservedCPUs", s.KubeReserved, "systemReservedCPUs", s.SystemReserved)
		}

		kubeReserved, err := parseResourceList(s.KubeReserved)
		if err != nil {
			return err
		}
		systemReserved, err := parseResourceList(s.SystemReserved)
		if err != nil {
			return err
		}
		var hardEvictionThresholds []evictionapi.Threshold
		// If the user requested to ignore eviction thresholds, then do not set valid values for hardEvictionThresholds here.
		if !s.ExperimentalNodeAllocatableIgnoreEvictionThreshold {
			hardEvictionThresholds, err = eviction.ParseThresholdConfig([]string{}, s.EvictionHard, nil, nil, nil)
			if err != nil {
				return err
			}
		}
		experimentalQOSReserved, err := cm.ParseQOSReserved(s.QOSReserved)
		if err != nil {
			return err
		}

		var cpuManagerPolicyOptions map[string]string
		if utilfeature.DefaultFeatureGate.Enabled(features.CPUManagerPolicyOptions) {
			cpuManagerPolicyOptions = s.CPUManagerPolicyOptions
		} else if s.CPUManagerPolicyOptions != nil {
			return fmt.Errorf("CPU Manager policy options %v require feature gates %q, %q enabled",
				s.CPUManagerPolicyOptions, features.CPUManager, features.CPUManagerPolicyOptions)
		}

		var topologyManagerPolicyOptions map[string]string
		if utilfeature.DefaultFeatureGate.Enabled(features.TopologyManager) {
			if utilfeature.DefaultFeatureGate.Enabled(features.TopologyManagerPolicyOptions) {
				topologyManagerPolicyOptions = s.TopologyManagerPolicyOptions
			} else if s.TopologyManagerPolicyOptions != nil {
				return fmt.Errorf("topology manager policy options %v require feature gates %q, %q enabled",
					s.TopologyManagerPolicyOptions, features.TopologyManager, features.TopologyManagerPolicyOptions)
			}
		}

		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				RuntimeCgroupsName:    s.RuntimeCgroups,
				SystemCgroupsName:     s.SystemCgroups,
				KubeletCgroupsName:    s.KubeletCgroups,
				KubeletOOMScoreAdj:    s.OOMScoreAdj,
				CgroupsPerQOS:         s.CgroupsPerQOS,
				CgroupRoot:            s.CgroupRoot,
				CgroupDriver:          s.CgroupDriver,
				KubeletRootDir:        s.RootDirectory,
				ProtectKernelDefaults: s.ProtectKernelDefaults,
				NodeAllocatableConfig: cm.NodeAllocatableConfig{
					KubeReservedCgroupName:   s.KubeReservedCgroup,
					SystemReservedCgroupName: s.SystemReservedCgroup,
					EnforceNodeAllocatable:   sets.NewString(s.EnforceNodeAllocatable...),
					KubeReserved:             kubeReserved,
					SystemReserved:           systemReserved,
					ReservedSystemCPUs:       reservedSystemCPUs,
					HardEvictionThresholds:   hardEvictionThresholds,
				},
				QOSReserved:                              *experimentalQOSReserved,
				CPUManagerPolicy:                         s.CPUManagerPolicy,
				CPUManagerPolicyOptions:                  cpuManagerPolicyOptions,
				CPUManagerReconcilePeriod:                s.CPUManagerReconcilePeriod.Duration,
				ExperimentalMemoryManagerPolicy:          s.MemoryManagerPolicy,
				ExperimentalMemoryManagerReservedMemory:  s.ReservedMemory,
				ExperimentalPodPidsLimit:                 s.PodPidsLimit,
				EnforceCPULimits:                         s.CPUCFSQuota,
				CPUCFSQuotaPeriod:                        s.CPUCFSQuotaPeriod.Duration,
				ExperimentalTopologyManagerPolicy:        s.TopologyManagerPolicy,
				ExperimentalTopologyManagerScope:         s.TopologyManagerScope,
				ExperimentalTopologyManagerPolicyOptions: topologyManagerPolicyOptions,
			},
			s.FailSwapOn,
			kubeDeps.Recorder)

		if err != nil {
			return err
		}
	}

	if kubeDeps.PodStartupLatencyTracker == nil {
		kubeDeps.PodStartupLatencyTracker = kubeletutil.NewPodStartupLatencyTracker()
	}

	// TODO(vmarmol): Do this through container config.
	oomAdjuster := kubeDeps.OOMAdjuster
	if err := oomAdjuster.ApplyOOMScoreAdj(0, int(s.OOMScoreAdj)); err != nil {
		klog.InfoS("Failed to ApplyOOMScoreAdj", "err", err)
	}

	err = kubelet.PreInitRuntimeService(&s.KubeletConfiguration, kubeDeps, s.RemoteRuntimeEndpoint, s.RemoteImageEndpoint)
	if err != nil {
		return err
	}

	if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
		return err
	}

	if s.HealthzPort > 0 {
		mux := http.NewServeMux()
		healthz.InstallHandler(mux)
		go wait.Until(func() {
			err := http.ListenAndServe(net.JoinHostPort(s.HealthzBindAddress, strconv.Itoa(int(s.HealthzPort))), mux)
			if err != nil {
				klog.ErrorS(err, "Failed to start healthz server")
			}
		}, 5*time.Second, wait.NeverStop)
	}

	if s.RunOnce {
		return nil
	}

	// 通知systemd,kubelet已经启动
	go daemon.SdNotify(false, "READY=1")

	select {
	case <-done:
		break
	case <-ctx.Done():
		break
	}

	return nil
}
```





  
