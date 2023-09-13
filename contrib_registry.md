# 服务注册+服务发现+服务监听 各工具包实现
## etcd
### registry
```go
type Registry struct {
	opts   *options
	client *clientv3.Client
	kv     clientv3.KV
	lease  clientv3.Lease
}

func (r *Registry) Register(ctx context.Context, service *registry.ServiceInstance) error {
	....
	leaseID, err := r.registerWithKV(ctx, key, value)
	....
    go r.heartBeat(r.opts.ctx, leaseID, key, value)
}

func (r *Registry) registerWithKV(ctx context.Context, key string, value string) (clientv3.LeaseID, error) {
    grant, err := r.lease.Grant(ctx, int64(r.opts.ttl.Seconds()))
    ....
    _, err = r.client.Put(ctx, key, value, clientv3.WithLease(grant.ID))
    ....
    return grant.ID, nil
}
func (r *Registry) heartBeat(ctx context.Context, leaseID clientv3.LeaseID, key string, value string) {
    kac, err := r.client.KeepAlive(ctx, leaseID)
	....
}
func (r *Registry) Deregister(ctx context.Context, service *registry.ServiceInstance) error {
    defer func() {
        if r.lease != nil {
        r.lease.Close()
		}
    }()
    key := fmt.Sprintf("%s/%s/%s", r.opts.namespace, service.Name, service.ID)
    _, err := r.client.Delete(ctx, key)
    return err
}
```
* register函数中调用registerWithKV，通过lease的grant生成租约id,put服务信息时携带id(续期使用)
* register再调用hearBeat,即通过etcd的keepalive函数，实际即持续对租约id发心跳续约
* Deregister 删除etcd服务存值，同时通过调用lease.close关闭续约
* GetService 获取etcd存储的服务
### watcher
```go
type watcher struct {
	key         string
	ctx         context.Context
	cancel      context.CancelFunc
	client      *clientv3.Client
	watchChan   clientv3.WatchChan
	watcher     clientv3.Watcher
	kv          clientv3.KV
	first       bool
	serviceName string
}
func newWatcher(ctx context.Context, key, name string, client *clientv3.Client) (*watcher, error) {
    ....
	w.watchChan = w.watcher.Watch(...)
	....
}
func (w *watcher) Next() ([]*registry.ServiceInstance, error) {
    if w.first {
        item, err := w.getInstance()
        w.first = false
        return item, err
    }
    select {
        case <-w.ctx.Done():
            return nil, w.ctx.Err()
        case watchResp, ok := <-w.watchChan:
            if !ok || watchResp.Err() != nil {
                time.Sleep(time.Second)
                err := w.reWatch()
				if err != nil {
                    return nil, err
			    }
	        }
        return w.getInstance()
    }
}

func (w *watcher) getInstance() ([]*registry.ServiceInstance, error) {
    resp, err := w.kv.Get(w.ctx, w.key, clientv3.WithPrefix())
}


```
* newWatcher 调用etcd 的watch，监听对应服务变化，通过返回的chan获得变化信息
* Next 获取服务变化信息，初次时调用get获得先用服务，后续实际监听etcd watch返回的通道，但并没有根据返回类型做处理，而是再次调用get获取当前服务，猜测是为了发生并发修改注册服务时，直接获取最新注册服务

## zookeeper
### register
```go

```