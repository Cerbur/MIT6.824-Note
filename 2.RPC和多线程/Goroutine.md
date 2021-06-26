# Goroutine 基本用法

## mutex 操作

```go
mu := sync.Mutex{}
mu.Lock()
// code
mu.Unlock()
```

类似 Java 的 Lock

另一种写法

```go
mu := sync.Mutex{}
mu.Lock()
defer mu.Unlock()
	// code
```

可以使得函数结束时候释放锁资源

## WaitGroup

```go

var done sync.WaitGroup
done.Add(1)
done.Done()
done.Wait()
```

