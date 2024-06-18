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

![img](assets/apiserver_start.png)

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

# Run函数是主函数

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
	//写配置
	config, err := NewConfig(opts)
	if err != nil {
		return err
	}
	completed, err := config.Complete()
	if err != nil {
		return err
	}
    //注册server的路由，做好配置，但是没有启动
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

> cmd/kube-apiserver/app/config.go---func NewConfig(opts options.CompletedOptions) (*Config, error)
>
> cmd/kube-apiserver/app/config.go---func (c *Config) Complete() (CompletedConfig, error)
>
> cmd/kube-apiserver/app/server.go---func CreateServerChain(config CompletedConfig) (*aggregatorapiserver.APIAggregator, error)
>
> vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go---func (s *APIAggregator) PrepareRun() (preparedAPIAggregator, error)
>
> vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go---func (s preparedAPIAggregator) Run(ctx context.Context)





## APIServer的配置参数初始化

APIServer的配置参数比较多，大致做以下分类：

- genericConfig，通用配置，三个Server都会使用
- OpenAPI配置
- Storage（ETCD）配置
- Authentication认证配置
- Authorization授权配置

### genericConfig

```go
// cmd/kube-apiserver/app/config.go:74
// NewConfig creates all the resources for running kube-apiserver, but runs none of them.
func NewConfig(opts options.CompletedOptions)(*Config, error){
    c :=&Config{
        Options: opts,
    }
    //genericConfig, versionedInformers, storageFactory在CreateKubeAPIServerConfig函数中被使用
	genericConfig, versionedInformers, storageFactory, err := controlplaneapiserver.BuildGenericConfig(
		opts.CompletedOptions,
		[]*runtime.Scheme{legacyscheme.Scheme, apiextensionsapiserver.Scheme, aggregatorscheme.Scheme},
		controlplane.DefaultAPIResourceConfigSource(),
		generatedopenapi.GetOpenAPIDefinitions,
	)
    	if err != nil {
		return nil, err
	}
	//KubeAPIs中包含apiExtensions和aggregator中需要的很多依赖
	kubeAPIs, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(opts, genericConfig, versionedInformers, storageFactory)
	if err != nil {
		return nil, err
	}
	c.KubeAPIs = kubeAPIs

	apiExtensions, err := controlplaneapiserver.CreateAPIExtensionsConfig(*kubeAPIs.ControlPlane.Generic, kubeAPIs.ControlPlane.VersionedInformers, pluginInitializer, opts.CompletedOptions, opts.MasterCount,
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(kubeAPIs.ControlPlane.ProxyTransport, kubeAPIs.ControlPlane.Generic.EgressSelector, kubeAPIs.ControlPlane.Generic.LoopbackClientConfig, kubeAPIs.ControlPlane.Generic.TracerProvider))
	if err != nil {
		return nil, err
	}
	c.ApiExtensions = apiExtensions

	aggregator, err := controlplaneapiserver.CreateAggregatorConfig(*kubeAPIs.ControlPlane.Generic, opts.CompletedOptions, kubeAPIs.ControlPlane.VersionedInformers, serviceResolver, kubeAPIs.ControlPlane.ProxyTransport, kubeAPIs.ControlPlane.Extra.PeerProxy, pluginInitializer)
	if err != nil {
		return nil, err
	}
	c.Aggregator = aggregator

	return c, nil
}


```

### OpenAPI配置

Kubernetes 支持将其 API 的描述以 OpenAPI v3 形式发布。

APIServer会动态地生成OpenAPI的定义文件，描述集群中的所有API资源和操作；由OpenAPI服务器提供访问，OpenAPIServer从APIServer获取最新的OpenAPI 定义，并通过HTTP服务的方式对外提供定义文件。

当客户端需要获取 Kubernetes 集群的 OpenAPI 定义时,会向 OpenAPI 服务器发送请求。
OpenAPI 服务器收到请求后,会查找缓存或直接从 API 服务器拉取最新的 OpenAPI 定义,并返回给客户端。

OpenAPI服务器一般会同时部署v2和v3版本，v3可以提供更完整的API信息。

基本信息

```yaml
openapi: 3.0.0
info:
  title: Kubernetes API
  version: v1.23.0
```

服务器信息

```yaml
servers:
- url: https://kubernetes.default.svc
  description: The default API server
```

API路径和HTTP方法，请求参数和响应结构和内容

```yaml
paths:
  /api/v1/namespaces:
    get:
      summary: list or watch objects of kind Namespace
      parameters:
      - name: fieldSelector
        in: query
        schema:
          type: string
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/V1NamespaceList'
  /api/v1/namespaces/{name}:
    get:
      summary: read the specified Namespace
      parameters:
      - name: name
        in: path
        required: true
        schema:
          type: string
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/V1Namespace'
```

API的数据模型：

```yaml
components:
  schemas:
    V1Namespace:
      type: object
      properties:
        apiVersion:
          type: string
        kind:
          type: string
        metadata:
          $ref: '#/components/schemas/V1ObjectMeta'
        spec:
          $ref: '#/components/schemas/V1NamespaceSpec'
        status:
          $ref: '#/components/schemas/V1NamespaceStatus'
```



### Storage（etcd）配置

> pkg/controlplane/apiserver/config.go：187

```go
//配置	
storageFactoryConfig := kubeapiserver.NewStorageFactoryConfig()

storageFactoryConfig.APIResourceConfig = genericConfig.MergedResourceConfig
//实例化	
storageFactory, lastErr = storageFactoryConfig.Complete(s.Etcd).New()
```

定义了etcd的地址、认证、存储路径的prefix等信息后实例化了etcd storage对象。



### 认证配置

APIServer支持如下的认证策略：

- X509 Client Certs
- Static Token File
- Bootstrap Tokens
- Service Account Tokens
- OpenID Connect Tokens（OIDC）
- Webhook Token Authentication
- Authenticating Proxy





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

### APIExtensionServer

主要负责处理CRD方面的请求

1. 动态注册和管理CRD
2. CRD资源的验证和转化
3. 与KubeAPIServer的集成，动态扩展CRD资源并使用K8sAPI进行操作等等（例如kubectl操作）

### KubeAPIServer

负责处理K8s内置资源的请求，还包括通用处理，认证，鉴权等

1. K8s内置资源相关的API请求唯一入口点。
2. 和etcd集群进行交互 write&read
3. 对Kubernetes API请求进行授权和认证，授权规则存储在etcd中，由集群管理员统一管理和配置
4. 资源管理和操作
5. Watch&Informer

### AggregatorServer

1. 聚合和代理来自不同APIServer的请求，为K8s提供一个统一的API入口点，而不是直接访问哥哥独立的APIServer
2. 动态的API注册和发现：动态地扩展新的API资源，无需修改客户端代码
3. API请求的访问控制，授权规则独立配置和管理，之后转发到不同的APIServer上再进行授权和认证
4. 多个APIServer的负载均衡
5. 监控和日志记录

## 
