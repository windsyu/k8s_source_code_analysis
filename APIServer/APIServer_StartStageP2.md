[TOC]

# Abstract

apiserver是k8s控制面的一个组件，在众多组件中唯一一个对接etcd,对外暴露http服务的形式为k8s中各种资源提供增删改查等服务。它是RESTful风格，每个资源的URI都会形如
`/apis/{apiGroup}/{version}/namsspaces/{ns-name}/{resource-kind}/{resource-name}`
或
`/apis/{apiGroup}/{version}/{resource-kind}/{resource-name}`
apiserver中包含3个server组件，apiserver依靠这3个组件来对不同类型的请求提供处理

- APIExtensionServer: 主要负责处理CustomResourceDefination（CRD）方面的请求
- KubeAPIServer: 主要负责处理k8s内置资源的请求，此外还会包括通用处理，认证、鉴权等
- AggregratorServer: 主要负责aggregrate方面的处理，它充当一个代理服务器，将请求转发到聚合进来的k8s service中。

# 启动流程

## Cobro参数解析

APIServer命令行参数解析通过`cmd/kube-apiserver/app/server.go`中`func NewAPIServerCommand()`实现，通过RunE中的return位置跳转到Run函数。

```go
RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags()

			// Activate logging as soon as possible, after that
			// show flags with the final logging configuration.
			if err := logsapi.ValidateAndApply(s.Logs, utilfeature.DefaultFeatureGate); err != nil {
				return err
			}
			cliflag.PrintFlags(fs)

			// set default options
			completedOptions, err := s.Complete()
			if err != nil {
				return err
			}

			// validate options
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}
			// add feature enablement metrics
			utilfeature.DefaultMutableFeatureGate.AddMetrics()
			return Run(cmd.Context(), completedOptions)
		},
```

## Run函数

run函数执行3件事情：

1. 启动APIServer的3个server组件的路由
2. 注册健康检查，就绪探针，存活探针的地址
3. 启动http服务

> cmd/kube-apiserver/app/server.go：134

```go
func Run(ctx context.Context, opts options.CompletedOptions) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	config, err := NewConfig(opts)
	if err != nil {
		return err
	}
	completed, err := config.Complete()
	if err != nil {
		return err
	}
    //注册三个server的路由
	server, err := CreateServerChain(completed)
	if err != nil {
		return err
	}
	//注册健康检查，就绪，存活探针的地址
	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}
	//运行http server
	return prepared.Run(ctx)
}
```

## 三个server的创建流程

CreateServerChain函数的调用流程如下：

> cmd/kube-apiserver/app/server.go：162

```go
func CreateServerChain(config CompletedConfig) (*aggregatorapiserver.APIAggregator, error) {
    //notFoundHandler，用于处理找不到服务器的资源的请求。
	notFoundHandler := notfoundhandler.New(config.KubeAPIs.ControlPlane.Generic.Serializer, genericapifilters.NoMuxAndDiscoveryIncompleteKey)
    //创建APIExtensionServer
	apiExtensionsServer, err := config.ApiExtensions.New(genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
	if err != nil {
		return nil, err
	}
	crdAPIEnabled := config.ApiExtensions.GenericConfig.MergedResourceConfig.ResourceEnabled(apiextensionsv1.SchemeGroupVersion.WithResource("customresourcedefinitions"))
	//创建KubeAPIServer
	kubeAPIServer, err := config.KubeAPIs.New(apiExtensionsServer.GenericAPIServer)
	if err != nil {
		return nil, err
	}

	// aggregator comes last in the chain
	aggregatorServer, err := controlplaneapiserver.CreateAggregatorServer(config.Aggregator, kubeAPIServer.ControlPlane.GenericAPIServer, apiExtensionsServer.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdAPIEnabled, apiVersionPriorities)
	if err != nil {
		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
		return nil, err
	}

	return aggregatorServer, nil
}
```

