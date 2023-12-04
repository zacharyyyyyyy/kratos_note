# internal 框架内部的一些通用方法
## context  合并两个上下文
```go
type mergeCtx struct {
	parent1, parent2 context.Context

	done     chan struct{}
	doneMark uint32
	doneOnce sync.Once
	doneErr  error

	cancelCh   chan struct{}
	cancelOnce sync.Once
}

func (mc *mergeCtx) finish(err error) error {
    mc.doneOnce.Do(func() {
        mc.doneErr = err
        atomic.StoreUint32(&mc.doneMark, 1)
        close(mc.done)
    })
    return mc.doneErr
}
func (mc *mergeCtx) wait() {
    var err error
    select {
        case <-mc.parent1.Done():
            err = mc.parent1.Err()
        case <-mc.parent2.Done():
            err = mc.parent2.Err()
        case <-mc.cancelCh:
            err = context.Canceled
	}
    _ = mc.finish(err)
}
func (mc *mergeCtx) Err() error {
	
}

```
* mergeCtx实现 context接口的Deadline Done Err Value 方法，mergeCtx即为context
* finish 标记上下文已完成，利用sync.once保证只执行一次，原子操作对mc.doneMark存值(Err方法使用)，同时关闭mergeCtx.done通道
* wait 任意一个context done有输出或mergeCtx.cancelCh有输出则认为当前合并上下文运行结束；Err value deadline同理

## endpoint
* 主要作用为对scheme作简单校验

## group 
```go
type Group struct {
	new  func() interface{}
	vals map[string]interface{}
	sync.RWMutex
}
func NewGroup(new func() interface{}) *Group {
    if new == nil {
        panic("container.group: can't assign a nil to the new function")
	}
    return &Group{
        new:  new,
        vals: make(map[string]interface{}),
    }
}
```
* 统一复用一个简易容器结构(即map), 提供get reset clear等操作

## host
* 主要作用为对url截取对应的host与port

## marcher 
* 定义中间件方法
```go
type Matcher interface {
	Use(ms ...middleware.Middleware)
	Add(selector string, ms ...middleware.Middleware)
	Match(operation string) []middleware.Middleware
}

type Handler func(ctx context.Context, req interface{}) (interface{}, error)

// Middleware is HTTP/gRPC transport middleware.
type Middleware func(Handler) Handler
```
```go
func test(str string) Middleware {
    fmt.Println(str)
	//看test 此处大多执行option模式加载初始化所需参数
	return func(handler Handler) Handler {
		fmt.Println(1111)
		return func(ctx context.Context, req interface{}) (interface{}, error) {
			fmt.Println(222)
			fmt.Println(handler(ctx, nil))
			return nil, nil
		}
	}
}

func main() {
	t := func(ctx context.Context, req interface{}) (interface{}, error) {
		fmt.Println(000)
		return nil, nil
	}
	test("test")(t)(context.Background(), nil)
}

```
* middleware 采用了控制反转的思想，类似使用如上，设计思路为分别对中间件传入初始化所需参数，对应的的handler func以及handler所需的参数
* 采取与gin不太一致的设计，大概率是为了对各个中间件func新增一个初始化流程
```go
type matcher struct {
	prefix   []string
	defaults []middleware.Middleware
	matchs   map[string][]middleware.Middleware
}
func (m *matcher) Use(ms ...middleware.Middleware) {
    m.defaults = ms
}

func (m *matcher) Add(selector string, ms ...middleware.Middleware) {
    if strings.HasSuffix(selector, "*") {
        selector = strings.TrimSuffix(selector, "*")
        m.prefix = append(m.prefix, selector)
		sort.Slice(m.prefix, func(i, j int) bool {
            return m.prefix[i] > m.prefix[j]
        })
    }
    m.matchs[selector] = ms
}
```
* matcher 负责存储对应的中间件fuc
* 实现Matcher接口的use func ，在defaults切片存储通用的中间件方法
* 实现Matcher接口的add func , 在matchs map 传入需要匹配到对应前缀才执行的中间件方法
* 实现Matcher接口的match func，返回defaults的所有通用中间件方法+matchs前缀匹配成功的中间件方法合集