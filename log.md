# log 日志相关操作
## filter
```go
type Filter struct {
	ctx    context.Context
	logger Logger
	level  Level
	key    map[interface{}]struct{}
	value  map[interface{}]struct{}
	filter func(level Level, keyvals ...interface{}) bool
}
func NewFilter(logger Logger, opts ...FilterOption) *Filter {
    options := Filter{
        logger: logger,
        key:    make(map[interface{}]struct{}),
        value:  make(map[interface{}]struct{}),
    }
    for _, o := range opts {
        o(&options)
    }
    return &options
}
func (f *Filter) Log(level Level, keyvals ...interface{}) error {
    if level < f.level {
        return nil
    }
	......
	return f.logger.Log(level, keyvals...)
}
```
* 常规的option模式填充filter参数
* 实际是对log的扩展，新增过滤操作以及日志键值模糊操作

## global
```go
// globalLogger is designed as a global logger in current process.
var global = &loggerAppliance{}

// loggerAppliance is the proxy of `Logger` to
// make logger change will affect all sub-logger.
type loggerAppliance struct {
	lock sync.Mutex
	Logger
}
```
* global主要是实现一个全局通用的log组件
## helper
```go
// Helper is a logger helper.
type Helper struct {
	logger  Logger
	msgKey  string
	sprint  func(...interface{}) string
	sprintf func(format string, a ...interface{}) string
}
```
* helper 主要目的是针对不同的msgkey定义不同的日志输出格式
## 
