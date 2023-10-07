## value
```go
var (
	_ Value = (*atomicValue)(nil)
	_ Value = (*errValue)(nil)
)

type Value interface {
}

type atomicValue struct {
    atomic.Value
}
type errValue struct {
    err error
}
```
* 忽略变量的Value类型变量；空指针断言，将nil转化为*atomicValue及*errValue
* 作用为约束 atomicValue及errValue实现 Value接口功能
* atomicValue 负责原子存区值
* err 错误信息存取值


## reader
```go
type reader struct {
	opts   options
	values map[string]interface{}
	lock   sync.Mutex
}
```
```go
func (r *reader) cloneMap() (map[string]interface{}, error) {
r.lock.Lock()
defer r.lock.Unlock()
return cloneMap(r.values)
}
```
* cloneMap 实现深拷贝values 即拷贝map,主要利用了gob库序列化再反序列化(实际利用反射处理)
* 经压测，简单结构不如直接map取值(相差一个数量级)，但gob直接序列化再反序列化对复杂结构的map相对方便
```go

func (r *reader) Merge(kvs ...*KeyValue) error {}
```
* 通过cloneMap深拷贝map 后，读取传入kvs ,decoder及convertMap处理kvs
* 利用mergo.Map复制处理后的kvs 至拷贝出来的map中，最终更新到reader的values

## config
```go
type config struct {
	opts      options
	reader    Reader
	cached    sync.Map
	observers sync.Map
	watchers  []Watcher
}
```
* 常规的option模式载入配置参数
* 采用sync.Map构造localcache，减少直接读reader的map(sync.Map并发读优势)
```go
func (c *config) watch(w Watcher) {
    for {
       kvs, err := w.Next()
	   ......
	   if err := c.reader.Merge(kvs...); err != nil {
        log.Errorf("failed to merge next config: %v", err)
        continue
       }
	   ......
    }
}
```
* w.Next() 对应config/env 及 config/file 的watcher func，会堵塞
* 其中config/file常规地利用fsnotify监听文件夹变化，并重新Load配置参数,并调用reader Merge func 更新配置参数
```go
func (c *config) Load() error {
    for _, src := range c.opts.sources {
	    ......
	    w, err := src.Watch()
        if err != nil {
            log.Errorf("failed to watch config source: %v", err)
        return err
        }
        c.watchers = append(c.watchers, w)
        go c.watch(w)
    }
}
```
* 每次load都会new watcher,开协程调用watch()监听文件变化
