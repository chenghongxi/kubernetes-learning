# Controller-Manager

## 介绍:

Github: https://github.com/kubernetes/kubernetes

在`Kubernetes`中 `Controller-Manager` 作为集群内部的管理控制中心, 负责集群内的`Node`,`pod`,`Endpoint`,`service`
,`namespace`等资源的管理,这些资源各有一个`controller`来负责。

每个`Controller`通过 `Api-server`
提供的接口监控整个集群的每个资源对象的当前状态，当发生变化时，会尝试将状态恢复到`期望状态`。

Controller-Manager 由一系列的 Controller 组成:

- `Certificate Controller`
- `ClusterRoleAggregation Controller`
- `Node Controller`
- `CronJob Controller`
- `Daemon Controller`
- `Deployment Controller`
- `StatefulSet Controller`
- `Endpoint Controller`
- `Endpointslice Controller`
- `Garbage Collector`
- `Namespace Controller`
- `Job Controller`
- `Pod AutoScaler`
- `PodGC Controller`
- `ReplicaSet Controller`
- `Service Controller`
- `ServiceAccount Controller`
- `Volume Controller`
- `Resource quota Controller`
- `Disruption Controller`
- ...

## 源码分析 ( 启动流程 ):
Controller-Manager 组件与其他组件(如:kubelet,apiserver等)使用了corbra命令行框架处理启动命令,它从命令行的RunE回调函数-->Run函数，开始执行启动流程。
RunE回调函数: https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-controller-manager/app/controllermanager.go#L126
```go
func NewControllerManagerCommand() *cobra.Command {
	s, err := options.NewKubeControllerManagerOptions()
	if err != nil {
		klog.Fatalf("unable to initialize command options: %v", err)
	}

	cmd := &cobra.Command{
		Use: "kube-controller-manager",
		Long: `The Kubernetes controller manager is a daemon that embeds
the core control loops shipped with Kubernetes. In applications of robotics and
automation, a control loop is a non-terminating loop that regulates the state of
the system. In Kubernetes, a controller is a control loop that watches the shared
state of the cluster through the apiserver and makes changes attempting to move the
current state towards the desired state. Examples of controllers that ship with
Kubernetes today are the replication controller, endpoints controller, namespace
controller, and serviceaccounts controller.`,
		PersistentPreRunE: func(*cobra.Command, []string) error {
			// silence client-go warnings.
			// kube-controller-manager generically watches APIs (including deprecated ones),
			// and CI ensures it works properly against matching kube-apiserver versions.
			restclient.SetDefaultWarningHandler(restclient.NoWarnings{})
			return nil
		},
		RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()

			// Activate logging as soon as possible, after that
			// show flags with the final logging configuration.
			if err := logsapi.ValidateAndApply(s.Logs, utilfeature.DefaultFeatureGate); err != nil {
				return err
			}
			cliflag.PrintFlags(cmd.Flags())

			c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
			if err != nil {
				return err
			}
			// 添加一个功能指标
			utilfeature.DefaultMutableFeatureGate.AddMetrics()
			return Run(c.Complete(), wait.NeverStop)
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}

	fs := cmd.Flags()
	namedFlagSets := s.Flags(KnownControllers(), ControllersDisabledByDefault.List())
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name(), logs.SkipLoggingConfigurationFlags())
	registerLegacyGlobalFlags(namedFlagSets)
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}

	cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
	cliflag.SetUsageAndHelpFunc(cmd, namedFlagSets, cols)

	return cmd
}
```
Run函数: https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-controller-manager/app/controllermanager.go#L180
```text
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	// 开启事件处理管道
	c.EventBroadcaster.StartStructuredLogging(0)
	c.EventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: c.Client.CoreV1().Events("")})
	defer c.EventBroadcaster.Shutdown()

	if cfgz, err := configz.New(ConfigzName); err == nil {
		cfgz.Set(c.ComponentConfig)
	} else {
		klog.Errorf("unable to register configz: %v", err)
	}

	// 设置 healthz 检查
	var checks []healthz.HealthChecker
	var electionChecker *leaderelection.HealthzAdaptor
	if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		electionChecker = leaderelection.NewLeaderHealthzAdaptor(time.Second * 20)
		checks = append(checks, electionChecker)
	}
	healthzHandler := controllerhealthz.NewMutableHealthzHandler(checks...)

	// Start the controller manager HTTP server
	// unsecuredMux is the handler for these controller *after* authn/authz filters have been applied
	var unsecuredMux *mux.PathRecorderMux
	if c.SecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, healthzHandler)
		if utilfeature.DefaultFeatureGate.Enabled(features.ComponentSLIs) {
			slis.SLIMetricsWithReset{}.Install(unsecuredMux)
		}
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		// TODO: handle stoppedCh and listenerStoppedCh returned by c.SecureServing.Serve
		if _, _, err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}

	clientBuilder, rootClientBuilder := createClientBuilders(c)

	saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

	run := func(ctx context.Context, startSATokenController InitFunc, initializersFunc ControllerInitializersFunc) {
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			klog.Fatalf("error building controller context: %v", err)
		}
		controllerInitializers := initializersFunc(controllerContext.LoopMode)
		if err := StartControllers(ctx, controllerContext, startSATokenController, controllerInitializers, unsecuredMux, healthzHandler); err != nil {
			klog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(stopCh)
		controllerContext.ObjectOrMetadataInformerFactory.Start(stopCh)
		close(controllerContext.InformersStarted)

		<-ctx.Done()
	}

	// 没有领导者选举，直接参选
	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		ctx, _ := wait.ContextForChannel(stopCh)
		run(ctx, saTokenControllerInitFunc, NewControllerInitializers)
		return nil
	}

	id, err := os.Hostname()
	if err != nil {
		return err
	}

	// add a uniquifier so that two processes on the same host don't accidentally both become active
	id = id + "_" + string(uuid.NewUUID())

	// leaderMigrator will be non-nil if and only if Leader Migration is enabled.
	var leaderMigrator *leadermigration.LeaderMigrator = nil

	// startSATokenController will be original saTokenControllerInitFunc if leader migration is not enabled.
	startSATokenController := saTokenControllerInitFunc

	// If leader migration is enabled, create the LeaderMigrator and prepare for migration
	if leadermigration.Enabled(&c.ComponentConfig.Generic) {
		klog.Infof("starting leader migration")

		leaderMigrator = leadermigration.NewLeaderMigrator(&c.ComponentConfig.Generic.LeaderMigration,
			"kube-controller-manager")

		// Wrap saTokenControllerInitFunc to signal readiness for migration after starting
		//  the controller.
		startSATokenController = func(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
			defer close(leaderMigrator.MigrationReady)
			return saTokenControllerInitFunc(ctx, controllerContext)
		}
	}

	// 开启一个主协程锁
	go leaderElectAndRun(c, id, electionChecker,
		c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		c.ComponentConfig.Generic.LeaderElection.ResourceName,
		leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				initializersFunc := NewControllerInitializers
				if leaderMigrator != nil {
					// If leader migration is enabled, we should start only non-migrated controllers
					//  for the main lock.
					initializersFunc = createInitializersFunc(leaderMigrator.FilterFunc, leadermigration.ControllerNonMigrated)
					klog.Info("leader migration: starting main controllers.")
				}
				run(ctx, startSATokenController, initializersFunc)
			},
			OnStoppedLeading: func() {
				klog.ErrorS(nil, "leaderelection lost")
				klog.FlushAndExit(klog.ExitFlushTimeout, 1)
			},
		})

	// If Leader Migration is enabled, proceed to attempt the migration lock.
	if leaderMigrator != nil {
		// Wait for Service Account Token Controller to start before acquiring the migration lock.
		// At this point, the main lock must have already been acquired, or the KCM process already exited.
		// We wait for the main lock before acquiring the migration lock to prevent the situation
		//  where KCM instance A holds the main lock while KCM instance B holds the migration lock.
		<-leaderMigrator.MigrationReady

		// Start the migration lock.
		go leaderElectAndRun(c, id, electionChecker,
			c.ComponentConfig.Generic.LeaderMigration.ResourceLock,
			c.ComponentConfig.Generic.LeaderMigration.LeaderName,
			leaderelection.LeaderCallbacks{
				OnStartedLeading: func(ctx context.Context) {
					klog.Info("leader migration: starting migrated controllers.")
					// DO NOT start saTokenController under migration lock
					run(ctx, nil, createInitializersFunc(leaderMigrator.FilterFunc, leadermigration.ControllerMigrated))
				},
				OnStoppedLeading: func() {
					klog.ErrorS(nil, "migration leaderelection lost")
					klog.FlushAndExit(klog.ExitFlushTimeout, 1)
				},
			})
	}

	<-stopCh
	return nil
}
```


StartControllers函数: https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-controller-manager/app/controllermanager.go#L231
- `完成 controller-manager注册并启动`
```text
func StartControllers(ctx context.Context, controllerCtx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc,
	unsecuredMux *mux.PathRecorderMux, healthzHandler *controllerhealthz.MutableHealthzHandler) error {
	// Always start the SA token controller first using a full-power client, since it needs to mint tokens for the rest
	// If this fails, just return here and fail since other controllers won't be able to get credentials.
	if startSATokenController != nil {
		if _, _, err := startSATokenController(ctx, controllerCtx); err != nil {
			return err
		}
	}

	// Initialize the cloud provider with a reference to the clientBuilder only after token controller
	// has started in case the cloud provider uses the client builder.
	if controllerCtx.Cloud != nil {
		controllerCtx.Cloud.Initialize(controllerCtx.ClientBuilder, ctx.Done())
	}

	var controllerChecks []healthz.HealthChecker

	for controllerName, initFn := range controllers {
		if !controllerCtx.IsControllerEnabled(controllerName) {
			klog.Warningf("%q is disabled", controllerName)
			continue
		}

		time.Sleep(wait.Jitter(controllerCtx.ComponentConfig.Generic.ControllerStartInterval.Duration, ControllerStartJitter))

		klog.V(1).Infof("Starting %q", controllerName)
		ctrl, started, err := initFn(ctx, controllerCtx)
		if err != nil {
			klog.Errorf("Error starting %q", controllerName)
			return err
		}
		if !started {
			klog.Warningf("Skipping %q", controllerName)
			continue
		}
		check := controllerhealthz.NamedPingChecker(controllerName)
		if ctrl != nil {
			// check if the controller supports and requests a debugHandler
			// and it needs the unsecuredMux to mount the handler onto.
			if debuggable, ok := ctrl.(controller.Debuggable); ok && unsecuredMux != nil {
				if debugHandler := debuggable.DebuggingHandler(); debugHandler != nil {
					basePath := "/debug/controllers/" + controllerName
					unsecuredMux.UnlistedHandle(basePath, http.StripPrefix(basePath, debugHandler))
					unsecuredMux.UnlistedHandlePrefix(basePath+"/", http.StripPrefix(basePath, debugHandler))
				}
			}
			if healthCheckable, ok := ctrl.(controller.HealthCheckable); ok {
				if realCheck := healthCheckable.HealthChecker(); realCheck != nil {
					check = controllerhealthz.NamedHealthChecker(controllerName, realCheck)
				}
			}
		}
		controllerChecks = append(controllerChecks, check)

		klog.Infof("Started %q", controllerName)
	}

	healthzHandler.AddHealthChecker(controllerChecks...)

	return nil
}
```







