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
## helper_writer
```go
type writerWrapper struct {
	helper *Helper
	level  Level
}
type WriterOptionFn func(w *writerWrapper)
```
* helper的writer实现
## level 定义日志等级

## value 
```go
func Caller(depth int) Valuer {
	return func(context.Context) interface{} {
		_, file, line, _ := runtime.Caller(depth)
		idx := strings.LastIndexByte(file, '/')
		if idx == -1 {
			return file[idx+1:] + ":" + strconv.Itoa(line)
		}
		idx = strings.LastIndexByte(file[:idx], '/')
		return file[idx+1:] + ":" + strconv.Itoa(line)
	}
}

// Timestamp returns a timestamp Valuer with a custom time format.
func Timestamp(layout string) Valuer {
	return func(context.Context) interface{} {
		return time.Now().Format(layout)
	}
}
```
* caller定义输出调用文件的路径层数，常用于记录产生错误的文件所在路径
* timestamp 统一时间格式化时间

## std
```go
type stdLogger struct {
	log  *log.Logger
	pool *sync.Pool
}

// NewStdLogger new a logger with writer.
func NewStdLogger(w io.Writer) Logger {
	return &stdLogger{
		log: log.New(w, "", 0),
		pool: &sync.Pool{
			New: func() interface{} {
				return new(bytes.Buffer)
			},
		},
	}
}
func (l *stdLogger) Log(level Level, keyvals ...interface{}) error {
	...
	buf := l.pool.Get().(*bytes.Buffer)
	buf.WriteString(level.String())
	...
	_ = l.log.Output(4, buf.String()) //nolint:gomnd
	buf.Reset()
	l.pool.Put(buf)
}
```
* 主要作用利用sync.pool,各协程记录日志时复用buffer，减少多次实例化

## log 日志入口
```go
var DefaultLogger = NewStdLogger(log.Writer())
// Logger is a logger interface.
type Logger interface {
	Log(level Level, keyvals ...interface{}) error
}
type logger struct {
	logger    Logger
	prefix    []interface{}   
	hasValuer bool
	ctx       context.Context
}
```
* prefix 为日志中间件，如value 的caller 与timestamp
```go
func (c *logger) Log(level Level, keyvals ...interface{}) error {
	kvs := make([]interface{}, 0, len(c.prefix)+len(keyvals))
	kvs = append(kvs, c.prefix...)
	if c.hasValuer {
		bindValues(c.ctx, kvs)
	}
	kvs = append(kvs, keyvals...)
	return c.logger.Log(level, kvs...)
}
```
* bindValues 实际上是给中间件传入ctx 并执行，再存入kvs(字符串)，例如执行caller输出日志调用路径，调用timestamp
  输出时间戳，再执行Log()时，实际将kvs的值拼接输出
```go
func With(l Logger, kv ...interface{}) Logger {
	....
    kvs = append(kvs, c.prefix...)
    kvs = append(kvs, kv...)
    return &logger{
        logger:    c.logger,
        prefix:    kvs,
        hasValuer: containsValuer(kvs),
        ctx:       c.ctx,
    }
}
```
* 对logger添加中间件
