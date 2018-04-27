


## defer

### 一、defer 关键字工作原则

```go
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```
defer 函数中的变量 i 在 defer 函数被定义的时候就已经明确，值为0。随后 defer 之外的 i++ 并不会影响 defer 函数打印的，所以打印结果为：0。


```go
func b() {
    for i := 0; i < 4; i++ {
        defer fmt.Print(i)
    }
}
```
多个 defer 函数被调用的顺序遵循 后进先出 的原则，所以打印结果为：3210。


```go
func c() (i int) {
    defer func() { i++ }()
    return 1
}
```
defer 函数可以读取并赋值给外部函数的命名返回值。所以函数 c 的返回值为：2。

### 二、defer return 执行流程

```go
func f0() (r int) {
    t := 5
    defer func() {
        fmt.Printf("f0 defer t is :%d", t)//5
        fmt.Printf("f0 defer r is :%d", r)//5
        t = t + 5
        fmt.Printf("f0 defer r is :%d", r)//5
    }()
    return t
}
```
这里需要注意的是,return xxx语句并不是一条原子指令，defer 被插入到了 赋值 与 return 之间。因此 return t应该被拆解程两部分，r = t 和 return。

这样 f0 函数可以被改造成下面的函数

```go
func f0() (r int) {
    t := 5
    r = t
    defer func() {
        fmt.Printf("f0 defer t is :%d", t)//5
        fmt.Printf("f0 defer r is :%d", r)//5
        t = t + 5
        fmt.Printf("f0 defer r is :%d", r)//5
    }()
    return
}
```
最终函数 f0 的返回值为：5。

## context

```go
package TestProject

import (
    "context"
    "fmt"
    "time"
    "testing"
)

/*
    在这里我认为context.WithCancel返回的上下文对象、cancel函数
    使用场景ABC……
    统一对ABC停止- -！我只能想到这些
 */
func Test_ContextWithCancel(t *testing.T)  {
    ctx,cancel:=context.WithCancel(context.Background())
    go printx(ctx)
    time.Sleep(5*time.Second)
    cancel()
    time.Sleep(1*time.Second)
}

/*
    ABC……超时停止- -！
 */
func Test_ContextWithTimeout(t *testing.T)  {
    ctx,cancel:=context.WithTimeout(context.Background(),4*time.Second)
    go printx(ctx)
    time.Sleep(5*time.Second)
    cancel()
    time.Sleep(time.Second)

}

/*
    ABC……直到某个时刻去停止
 */
func Test_ContextWithDeadLine(t *testing.T)  {
    ctx,cancel:=context.WithDeadline(context.Background(),time.Now().Add(4*time.Second))
    go printx(ctx)
    time.Sleep(5*time.Second)
    cancel()
    time.Sleep(time.Second)
}

/*
    ABC……同时引用上下文中的值？下面是一处使用场景
    // NewContext returns a new Context carrying userIP.
    func NewContext(ctx context.Context, userIP net.IP) context.Context {
        return context.WithValue(ctx, userIPKey, userIP)
    }

    // FromContext extracts the user IP address from ctx, if present.
    func FromContext(ctx context.Context) (net.IP, bool) {
      // ctx.Value returns nil if ctx has no value for the key;
      // the net.IP type assertion returns ok=false for nil.
        userIP, ok := ctx.Value(userIPKey).(net.IP)
        return userIP, ok
    }
*/
func Test_ContextWithValue(t *testing.T)  {
    ctx:=context.WithValue(context.Background(),"key","value")
    go printx(ctx)
    time.Sleep(5*time.Second)
}

func printx(ctx context.Context)  {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("over")
            return
        default:
            fmt.Println("xxx\n")
            if ctx.Value("key")!=nil{
                fmt.Println(ctx.Value("key").(string)+"\n")
            }
            time.Sleep(time.Second)

        }
    }
}
```

## chan

## sync.WaitGroup

```go
var wg sync.WaitGroup
	ackChan := make(chan []byte, len(terDataSplit))
	defer close(ackChan)
	wg.Add(len(terDataSplit))
	for _, terData := range terDataSplit {
		go func(terByte []byte) {
			defer wg.Done()
			ack, errcode, err := dewav.TerDataDealTCP(conn, addr, terByte, len(terByte))
			if err != nil {
				formatterErrorInfo(terByte, len(terByte))
				log.GLog.Error("[DataHandlerTCP] errcode :%d error :%s", errcode, err)
			}
			ackChan <- ack
		}(terData)
	}
	wg.Wait()
	for i := 0; i < len(terDataSplit); i++ {
		ackByte = gomathbits.BytesCombine(ackByte, <-ackChan)
	}
	conn.Write(ackByte)
```


## struct

### 字节对齐

```go
// |x---|
So(unsafe.Sizeof(struct {
    i8 int8
}{}), ShouldEqual, 1)
```
简单封装一个 int8 的结构体，和 int8 一样，仅占1个字节，没有额外空间

```go
// |x---|xxxx|xx--|
So(unsafe.Sizeof(struct {
    i8  int8
    i32 int32
    i16 int16
}{}), ShouldEqual, 12)

// |x-xx|xxxx|
So(unsafe.Sizeof(struct {
    i8  int8
    i16 int16
    i32 int32
}{}), ShouldEqual, 8)
```
这两个结构体里面的内容完全一样，调整了一下字段顺序，节省了 33% 的空间

```go
// |x---|xxxx|xx--|----|xxxx|xxxx|
So(unsafe.Sizeof(struct {
    i8  int8
    i32 int32
    i16 int16
    i64 int64
}{}), ShouldEqual, 24)

// |x-xx|xxxx|xxxx|xxxx|
So(unsafe.Sizeof(struct {
    i8  int8
    i16 int16
    i32 int32
    i64 int64
}{}), ShouldEqual, 16)
```

这里需要注意的是 int64 只能出现在8的倍数的地址处，因此第一个结构体中，有连续的4个字节是空的

```go
type I8 int8
type I16 int16
type I32 int32

So(unsafe.Sizeof(struct {
    i8  I8
    i16 I16
    i32 I32
}{}), ShouldEqual, 8)
```