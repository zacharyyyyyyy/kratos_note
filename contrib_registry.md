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
type options struct {
    namespace string
    user      string
    password  string
}
type Registry struct {
	opts *options
	conn *zk.Conn

	group singleflight.Group
}

func (r *Registry) ensureName(path string, data []byte, flags int32) error {
    exists, stat, err := r.conn.Exists(path)
	...
    if flags&zk.FlagEphemeral == zk.FlagEphemeral {
        err = r.conn.Delete(path, stat.Version)
        ...
        exists = false
    }
    if !exists {
        if len(r.opts.user) > 0 && len(r.opts.password) > 0 {
            _, err = r.conn.Create(path, data, flags, zk.DigestACL(zk.PermAll, r.opts.user, r.opts.password))
        } else {
            _, err = r.conn.Create(path, data, flags, zk.WorldACL(zk.PermAll))
		}
		....
    }
}
func (r *Registry) reRegister(path string, data []byte) {
    sessionID := r.conn.SessionID()
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    for range ticker.C {
        cur := r.conn.SessionID()
        // sessionID changed
        if cur > 0 && sessionID != cur {
        // re-ensureName
            if err := r.ensureName(path, data, zk.FlagEphemeral); err != nil {
                return
            }
            sessionID = cur
        }
    }
}
func (r *Registry) Register(_ context.Context, service *registry.ServiceInstance) error {
    ...
	if err = r.ensureName(r.opts.namespace, []byte(""), 0); err != nil {
	    ....
    }
	...
	go r.reRegister(servicePath, data)
	...
}
func (r *Registry) Deregister(ctx context.Context, service *registry.ServiceInstance) error {
	....
	err := r.conn.Delete(servicePath, -1)
	...
}
func (r *Registry) GetService(_ context.Context, serviceName string) ([]*registry.ServiceInstance, error) {
    instances, err, _ := r.group.Do(serviceName, func() (interface{}, error) {
        serviceNamePath := path.Join(r.opts.namespace, serviceName)
        servicesID, _, err := r.conn.Children(serviceNamePath)
		...
        for _, service := range servicesID {
            servicePath := path.Join(serviceNamePath, service)
            serviceInstanceByte, _, err := r.conn.Get(servicePath)
			...
		}
		...
    }
}
```
* ensureName 负责构建节点 存在且为临时节点，先删除对应的节点，保证节点未最新状态;根据传入flags构建对应节点(0 永久 1临时 2有序 3临时有序节点)
* reRegister 重新构建节点，开启定时器，定时判断sessionid有无变化，有变化表面节点存在改动，调用ensureName构建最新节点
* Register 服务注册，根据配置调用ensureName构建节点，并开协程异步调用reRegister，保证节点为最新状态(为临时节点时，相当于心跳包)
* Deregister 删除对应注册的服务信息
* GetService 通过servicename 获取以注册的节点信息。利用singleflight合并多个获取相同serviceName的操作，调用children()获取对应path下所有注册的节点信息

### watcher
```go
type watcher struct {
	ctx    context.Context
	event  chan zk.Event
	conn   *zk.Conn
	cancel context.CancelFunc

	first uint32
	// 前缀
	prefix string
	// watch 的服务名
	serviceName string
}
func (w *watcher) watch(ctx context.Context) {
    for {
        _, _, ch, err := w.conn.ChildrenW(w.prefix)
		...
        select {
            case <-ctx.Done():
                return
            case ev := <-ch:
                w.event <- ev
        }
    }
}
func (w *watcher) Next() ([]*registry.ServiceInstance, error) {
	...
    select {
        ...
        case e := <-w.event:
            ...
            return w.getServices()
    }
}
func (w *watcher) getServices() ([]*registry.ServiceInstance, error) {
        servicesID, _, err := w.conn.Children(w.prefix)
		....
        for _, id := range servicesID {
            servicePath := path.Join(w.prefix, id)
            b, _, err := w.conn.Get(servicePath)
			...
        }
}
```
* zookeeper watch机制比较特别，只能监听一次变化，需要不断循环监听，且循环监听间隔会存在节点变化遗漏监听问题；通过通道w.event传递监听到的节点变化信息
* Next 根据event传递的节点变化信息，重新调用getServices()获取对应目录下的节点信息