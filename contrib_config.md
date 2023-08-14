## 负责服务发现
## etcd
### config
```go
type Option func(o *options)

type options struct {
	ctx    context.Context
	path   string
	prefix bool
}
func (s *source) Load() ([]*config.KeyValue, error) {}
```
* 常规的option 模式
* Load ，调用etcd watch后，监听到事件则调用load，load函数调用etcd get方法，获取对应路径的数据返回
### watcher 
```go
type watcher struct {
	source *source
	ch     clientv3.WatchChan

	ctx    context.Context
	cancel context.CancelFunc
}
```
* 主要负责调用etcd watch,监听事件，事件分为put,delete事件，但由于Load方法每次都重新get一次数据，所以并没有区分事件类型处理

## 剩下组件暂未接触，待补充