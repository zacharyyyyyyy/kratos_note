# 
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

## 