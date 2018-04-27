## 条件变量

我们在第6章讲多线程编程的时候详细说明过条件变量的概念、原理和适用场景。因此，我们在本小节仅对sync代码包中与条件变量相关的API进行简单的介绍，并使用它们来改造我们之前实现的*myDataFile类型的相关方法。

在Go语言中，sync.Cond类型代表了条件变量。与互斥锁和读写锁不同，简单的声明无法创建出一个可用的条件变量。为了得到这样一个条件变量，我们需要用到sync.NewCond函数。该函数的声明如下：

```go
func NewCond(l Locker) *Cond
```

我们在第6章中说过，条件变量总是要与互斥量组合使用。因此，sync.NewCond函数的唯一参数是sync.Locker类型的，而具体的参数值既可以是一个互斥锁也可以是一个读写锁。sync.NewCond函数在被调用之后会返回一个*sync.Cond类型的结果值。我们可以调用该值拥有的几个方法来操纵对应的条件变量。

 

类型*sync.Cond的方法集合中有三个方法，即：<b>Wait方法、Signal方法和Broadcast方法。</b>它们分别代表了等待通知、单发通知和广播通知的操作。

方法Wait会自动的对与该条件变量关联的那个锁进行解锁，并且使调用方所在的Goroutine被阻塞。一旦该方法收到通知，就会试图再次锁定该锁。如果锁定成功，它就会唤醒那个被它阻塞的Goroutine。否则，该方法会等待下一个通知，那个Goroutine也会继续被阻塞。而方法Signal和Broadcast的作用都是发送通知以唤醒正在为此而被阻塞的Goroutine。不同的是，前者的目标只有一个，而后者的目标则是所有。


我们在第6章的“线程的同步”小节中详细的描述过这些操作的行为和意义。读者可以在需要时回顾其中的内容。


在上一小节，我们在\*myDataFile类型的Read方法和Write方法的实现中使用到了读写锁fmutex。在Read方法中，我们对一种边界情况进行了特殊处理，即：如果*os.File类型的f字段的ReadAt方法在被调用后返回了一个非nil且等于io.EOF的错误值，那么Read方法就忽略这个错误并再次尝试读取相同位置的数据块，直到读取成功为止。从这个特殊处理的具体流程上来看，似乎使用条件变量来作为辅助手段会带来一些好处。下面我们就来动手试验一下。


我们先在结构体类型myDataFile增加一个类型为*sync.Cond的字段rcond。为了快速实现想法，我们暂时不考虑怎样初始化这个字段，而直接去改造Read方法和Write方法。


在Read方法中，我们使用一个for循环来达到重新尝试获取数据块的目的。为此，我们添加了若干条重复的语句、降低了程序的性能，还造成了一个潜在的问题——在某个情况下读写锁fmutex不会被读解锁。为了解决这一系列新生的问题，我们使用代表条件变量的字段rcond。Read方法的第三个版本如下：

```
func (df *myDataFile) Read() (rsn int64, d Data, err error) {
 
// 读取并更新读偏移量
 
// 省略若干条语句
 
 
//读取一个数据块
 
rsn = offset / int64(df.dataLen)
 
bytes := make([]byte, df.dataLen)
 
df.fmutex.RLock()
 
defer df.fmutex.RUnlock()
 
for {
	 
	_, err = df.f.ReadAt(bytes, offset)
	 
	if err != nil {
	 
		if err == io.EOF {
		 
			df.rcond.Wait()
			 
			continue
		 
		}
		 
		return
	 
	}
	 
	d = bytes
	 
	return
	 
	}
 
}
```

在这里，我们假设条件变量rcond与读写锁fmutex中的“读锁”相关联。可以看到，我们让defer df.fmutex.RUnlock()语句回归了，并删除了所有return语句和continue语句前面的针对fmutex的读解锁操作。这都得益于新增在continue语句前面的df.rcond.Wait()。添加这条语句的意义在于：当发现由文件内容读取造成的EOF错误时，要让当前Goroutine暂时放弃fmutex的“读锁”并等待通知的到来。放弃fmutex的“读锁”也就意味着Write方法中的数据块写操作不会受到它的阻碍了。在写操作完成之后，我们应该及时向条件变量rcond发送通知以唤醒为此而等待的Goroutine。请注意，在某个Goroutine被唤醒之后，应该再次检查需要被满足的条件。在这里，这个需要被满足的条件是在进行文件内容读取时不会造成EOF错误。如果该条件被满足，那么就可以进行后续的操作了。否则，应该再次放弃“读锁”并等待通知。这也是我们依然保留for循环的原因。

这里有两点需要特别注意。

一定要在调用rcond的Wait方法之前锁定与之关联的那个“读锁”，否则就会造成对Wait方法的调用永远无法返回。这种情况会导致流程执行的停滞，甚至整个程序的死锁！导致这种结果的原因与条件变量和读写锁的内部实现方式有关（结果也许并不应该是这样，作者已经向Go语言官方提交了一个issue；Go语言官方已经接受了这个issue，并承诺将会在Go 1.4版本中改进它）。假设，与条件变量rcond关联的是某个读写锁的“写锁”或普通的互斥锁，那么对rcond.Wait方法的调用将会引发一个运行时恐慌。原因是，该方法会先对与之关联的锁进行解锁，而试图解锁未被锁定的锁就会引发一个运行时恐慌。


一定不要忘记在读操作完成之前解锁与条件变量rcond关联的那个“读锁”，否则对读写锁的写锁定操作将会阻塞相关的Goroutine。其根本原因是，条件变量rcond的Wait方法在返回之前会重新锁定与之关联的那个“读锁”。因此，在结束这个从文件中读取一个数据块的流程之前，我们应该调用fmutex字段的RLock方法。那条defer语句就起到了这个作用。


我们对Read方法的这次改进使得它的实现变得更加简洁和清晰了。不过，要想使其中的条件变量rcond真正发挥作用，还需要Write方法的配合。换句话说，为了让rcond.Wait方法可以适时的返回，我们要在向文件写入一个数据块之后及时的向rcond发送通知。添加了这一操作的Write方法如下：

```
func (df *myDataFile) Write(d Data) (wsn int64, err error) {

	// 省略若干条语句
	
	var bytes []byte
	
	// 省略若干条语句
	
	df.fmutex.Lock()
	
	defer df.fmutex.Unlock()
	
	_, err = df.f.Write(bytes)
	
	df.rcond.Signal()
	
	return

}
```

由于一个数据块只能由某一个读操作读取，所以我们只是使用条件变量的Signal方法去通知某一个为此等待的Wait方法，并以此唤醒某一个相关的Goroutine。这可以免去其它相关的Goroutine中的一些无谓操作。

与Wait方法不同，我们在调用条件变量的Signal方法和Broadcast方法之前无需锁定与之关联的锁。随之，相应的解锁操作也是不需要的。在这个Write方法中的锁定操作和解锁的操作针对的并不是df.rcond.Signal()语句。

我们一直在说，条件变量rcond是与读写锁fmutex的“读锁”关联的。这是怎样做到的呢？读者还记得我们在上一节提到读写锁的RLocker方法吗？它会返回当前读写锁中的“读锁”。这个结果值同时也是sync.Locker接口的实现。因此，我们可以把它作为参数值传给sync.NewCond函数。所以，我们在NewDataFile函数中的声明df变量的语句的后面加入了这样一条语句：

```go
df.rcond = sync.NewCond(df.fmutex.RLocker())
```

在这之后，我们就可以像前面那样使用这个条件变量了。

随着对*myDataFile类型和NewDataFile函数的改造的完成，我们也将结束本节。Go语言提供的互斥锁、读写锁和条件变量都基本遵循了POSIX标准中描述的对应的同步工具的行为规范。它们简单且高效。我们可以使用它们为复杂的类型提供并发安全的保证。在一些情况下，它们比通道更加灵活。在只需对一个或多个临界区进行保护的时候，使用锁往往会对程序的性能损耗更小。