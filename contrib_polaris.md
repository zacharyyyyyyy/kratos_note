# 腾讯polaris mesh
### 集成服务管理、流量管理、故障容错、配置管理和可观测性[文档地址][markdown]--[github地址][markdown1]
[markdown]: http://polarismesh.cn/docs/%E5%8C%97%E6%9E%81%E6%98%9F%E6%98%AF%E4%BB%80%E4%B9%88/%E7%AE%80%E4%BB%8B/
[markdown1]: https://github.com/polarismesh/polaris/blob/main/README-zh.md

### limiter + ratelimit (限流)
```go
func buildRequest(opts limiterOptions) polaris.QuotaRequest {
quotaRequest := polaris.NewQuotaRequest()
.....
}
func (l *Limiter) Allow(method string, argument ...model.Argument) (ratelimit.DoneFunc, error) {
    request := buildRequest(l.opts)
    request.SetMethod(method)
    for _, arg := range argument {
        request.AddArgument(arg)
    }
    resp, err := l.limitAPI.GetQuota(request)
    ....
}

// Handler defines the handler invoked by Middleware.
type Handler func(ctx context.Context, req interface{}) (interface{}, error)

// Middleware is HTTP/gRPC transport middleware.
type Middleware func(Handler) Handler

func Ratelimit(l Limiter) middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (reply interface{}, err error) {
            ....   //args header 及 querystring 的键值
			done, e := l.Allow(tr.Operation(), args...)
			....
		}
	}
}

```
* 设置命名空间，服务名称，根据配置获取method的配额GetQuota(即限流)
* 实际使用在ratelimit的Ratelimit(l Limiter)调用

### registry (服务注册+服务发现+服务监听)
```go
var (
_ registry.Registrar = (*Registry)(nil)
_ registry.Discovery = (*Registry)(nil)
)
```
* Registrar 服务注册 Discovery 服务发现
```go
type Registry struct {
opt      registryOptions
provider polaris.ProviderAPI
consumer polaris.ConsumerAPI
}

type ServiceInstance struct {   //注册服务结构
// ID is the unique instance ID as registered.
ID string `json:"id"`
// Name is the service name as registered.
Name string `json:"name"`
// Version is the version of the compiled.
Version string `json:"version"`
// Metadata is the kv pair metadata associated with the service instance.
Metadata map[string]string `json:"metadata"`
// Endpoints are endpoint addresses of the service instance.
// schema:
//   http://127.0.0.1:8000?isSecure=false
//   grpc://127.0.0.1:9000?isSecure=false
Endpoints []string `json:"endpoints"`
}

func (r *Registry) Register(_ context.Context, instance *registry.ServiceInstance) error {
for _, endpoint := range instance.Endpoints {
	....
	_, err = r.provider.RegisterInstance(
        &polaris.InstanceRegisterRequest{
        }
    )
}
}

func (r *Registry) Deregister(_ context.Context, serviceInstance *registry.ServiceInstance) error {
for _, endpoint := range serviceInstance.Endpoints {
    err = r.provider.Deregister(
        &polaris.InstanceDeRegisterRequest{
        }
    )
}
}

func (r *Registry) GetService(_ context.Context, serviceName string) ([]*registry.ServiceInstance, error) {
// get all instances
    instancesResponse, err := r.consumer.GetInstances(&polaris.GetInstancesRequest{
        GetInstancesRequest: model.GetInstancesRequest{
		}
    )
}
```
* provider RegisterInstance() 注册服务 （InstanceRegisterRequest结构）
* provider Deregister() 取消注册服务 
* consumer GetInstances 根据serviceName获取注册的endpoints
```go
type Watcher struct {
	ServiceName      string
	Namespace        string
	Ctx              context.Context
	Cancel           context.CancelFunc
	Channel          <-chan model.SubScribeEvent
	service          *model.InstancesResponse
	ServiceInstances map[string][]model.Instance
	first            bool
}

func newWatcher(ctx context.Context, namespace string, serviceName string, consumer polaris.ConsumerAPI) (*Watcher, error) {
    watchServiceResponse, err := consumer.WatchService(&polaris.WatchServiceRequest{
        WatchServiceRequest: model.WatchServiceRequest{
        }
    )
}

func (w *Watcher) Next() ([]*registry.ServiceInstance, error) {
    select {
        case <-w.Ctx.Done():
            return nil, w.Ctx.Err()
        case event := <-w.Channel:
		if event.GetSubScribeEventType() == model.EventInstance {
			.....
		}
    }
	.......
	return instancesToServiceInstances(w.ServiceInstances), nil
}

func instancesToServiceInstances(instances map[string][]model.Instance) []*registry.ServiceInstance {
}
```
* newWatcher 调用consumer.WatchService实例化监听
* next 监听watcher的EventChannel，返回对应服务的变化，同时根据 event.(*model.InstanceEvent) event类型(增删改)，利用ServiceInstances map 缓存当前已注册服务
* 最后经过instancesToServiceInstances 构造 ServiceInstance 指针数组返回

### router  (动态路由)
```go
func (p *Polaris) NodeFilter(opts ...RouterOption) selector.NodeFilter {
    return func(ctx context.Context, nodes []selector.Node) []selector.Node {
        req := &polaris.ProcessRoutersRequest{
        ProcessRoutersRequest: model.ProcessRoutersRequest{
        SourceService: model.ServiceInfo{Namespace: p.namespace, Service: o.service},
        DstInstances:  buildPolarisInstance(p.namespace, nodes),
         },
        }
        req.AddArguments(model.Build... ...)
		....
        m, err := p.router.ProcessRouters(req)
		...
        for _, ins := range m.GetInstances() {
	        ....
        }
    }
}
```
* NodeFilter 路由填充 (polaris.ProcessRoutersRequest结构)  初始化ProcessRoutersRequest结构后
  AddArguments 添加标签，最终调用ProcessRouters执行路由，并通过GetInstances获取最新的服务路由返回
```go
func buildPolarisInstance(namespace string, nodes []selector.Node) *pb.ServiceInstancesInProto {
    ins := make([]*v1.Instance, 0, len(nodes))
	for _, node := range nodes {
        ins = append(ins, &v1.Instance{
			......
        }
    }
    d := &v1.DiscoverResponse{
    Code:      wrapperspb.UInt32(1),
    Info:      wrapperspb.String("ok"),
    Type:      v1.DiscoverResponse_INSTANCE,
    Service:   &v1.Service{Name: wrapperspb.String(nodes[0].ServiceName()), Namespace: wrapperspb.String("default")},
    Instances: ins,
    }
    return pb.NewServiceInstancesInProto(d, func(s string) local.InstanceLocalValue {
        return local.NewInstanceLocalValue()
    }, &pb.SvcPluginValues{Routers: nil, Loadbalancer: nil}, nil)
}

// ServiceInstancesInProto 通用的应答.
type ServiceInstancesInProto struct {
service         *namingpb.Service
instances       []model.Instance
instancesMap    map[string]model.Instance
initialized     bool
svcIDSet        model.HashSet
totalWeight     int
revision        string
clusterCache    atomic.Value
svcPluginValues *SvcPluginValues
svcLocalValue   local.ServiceLocalValue
CacheLoaded     int32
}
```
* 构建服务路由的实例列表，根据nodes切片构建Ins实例切片，构造成DiscoverResponse 的结构(protobuf对应golang的结构)，
  最终调用NewServiceInstancesInProto转换为ServiceInstancesInProto的应答结构

### config (读配置+配置监听更新配置)
### polaris (初始化 router+config+limiter+registry)
